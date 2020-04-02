# tornado 之 options.py
## 摘要
命令行解析模块，允许模块定义自己的选项

"""
.. note::

   When using multiple ``parse_*`` functions, pass ``final=False`` to all
   but the last one, or side effects may occur twice (in particular,
   this can result in log messages being doubled).

.. note::

   By default, several options are defined that will configure the
   standard `logging` module when `parse_command_line` or `parse_config_file`
   are called.  If you want Tornado to leave the logging configuration
   alone so you can manage it yourself, either pass ``--logging=none``
   on the command line or do the following to disable it in code::

       from tornado.options import options, parse_command_line
       options.logging = None
       parse_command_line()
"""

import datetime
import numbers
import re
import sys
import os
import textwrap

from tornado.escape import _unicode, native_str
from tornado.log import define_logging_options
from tornado.util import basestring_type, exec_in

import typing
from typing import Any, Iterator, Iterable, Tuple, Set, Dict, Callable, List, TextIO

if typing.TYPE_CHECKING:
    from typing import Optional  # noqa: F401


class Error(Exception):
    """Exception raised by errors in the options module."""

    pass


class OptionParser(object):
    def __init__(self) -> None:
        # we have to use self.__dict__ because we override setattr.
        self.__dict__["_options"] = {}
        self.__dict__["_parse_callbacks"] = []
        self.define(
            "help",
            type=bool,
            help="show this help information",
            callback=self._help_callback,
        )
    # 把连接符转换成下划线
    def _normalize_name(self, name: str) -> str:
        return name.replace("_", "-")

    # 获取选项的值
    def __getattr__(self, name: str) -> Any:
        name = self._normalize_name(name)
        if isinstance(self._options.get(name), _Option):
            return self._options[name].value()
        raise AttributeError("Unrecognized option %r" % name)

    # 设置选项的值
    def __setattr__(self, name: str, value: Any) -> None:
        name = self._normalize_name(name)
        if isinstance(self._options.get(name), _Option):
            return self._options[name].set(value)
        raise AttributeError("Unrecognized option %r" % name)

    # 选项名称的迭代器
    def __iter__(self) -> Iterator:
        return (opt.name for opt in self._options.values())

    # 判断选项是否已经存在
    def __contains__(self, name: str) -> bool:
        name = self._normalize_name(name)
        return name in self._options

    def __getitem__(self, name: str) -> Any:
        return self.__getattr__(name)

    def __setitem__(self, name: str, value: Any) -> None:
        return self.__setattr__(name, value)

    # 选项和默认值的迭代器
    def items(self) -> Iterable[Tuple[str, Any]]:
        return [(opt.name, opt.value()) for name, opt in self._options.items()]

    # 选项组的集合
    def groups(self) -> Set[str]:
        return set(opt.group_name for opt in self._options.values())

    # 获取选项组的字典
    def group_dict(self, group: str) -> Dict[str, Any]:
        return dict(
            (opt.name, opt.value())
            for name, opt in self._options.items()
            if not group or group == opt.group_name
        )

    # 返回所有选项名和值的字典
    def as_dict(self) -> Dict[str, Any]:
        return dict((opt.name, opt.value()) for name, opt in self._options.items())

    # 新增一个选项
    def define(
        self,
        name: str,
        default: Any = None,
        type: type = None,
        help: str = None,
        metavar: str = None,
        multiple: bool = False,
        group: str = None,
        callback: Callable[[Any], None] = None,
    ) -> None:
        normalized = self._normalize_name(name)
        if normalized in self._options:
            raise Error(
                "Option %r already defined in %s"
                % (normalized, self._options[normalized].file_name)
            )
        frame = sys._getframe(0)
        options_file = frame.f_code.co_filename

        # 寻找调用的真正文件
        if (
            frame.f_back.f_code.co_filename == options_file
            and frame.f_back.f_code.co_name == "define"
        ):
            frame = frame.f_back

        file_name = frame.f_back.f_code.co_filename
        if file_name == options_file:
            file_name = ""
        if type is None:
            if not multiple and default is not None:
                type = default.__class__
            else:
                type = str
        if group:
            group_name = group  # type: Optional[str]
        else:
            group_name = file_name
        option = _Option(
            name,
            file_name=file_name,
            default=default,
            type=type,
            help=help,
            metavar=metavar,
            multiple=multiple,
            group_name=group_name,
            callback=callback,
        )
        self._options[normalized] = option

    # 解析命令行中给出的选项，返回未解析的列表
    def parse_command_line(
        self, args: List[str] = None, final: bool = True
    ) -> List[str]:
        if args is None:
            args = sys.argv
        remaining = []  # type: List[str]
        for i in range(1, len(args)):
            # All things after the last option are command line arguments
            if not args[i].startswith("-"):
                remaining = args[i:]
                break
            if args[i] == "--":
                remaining = args[i + 1 :]
                break
            arg = args[i].lstrip("-")
            name, equals, value = arg.partition("=")
            name = self._normalize_name(name)
            if name not in self._options:
                self.print_help()
                raise Error("Unrecognized command line option: %r" % name)
            option = self._options[name]
            if not equals:
                if option.type == bool:
                    value = "true"
                else:
                    raise Error("Option %r requires a value" % name)
            option.parse(value)

        if final:
            self.run_parse_callbacks()

        return remaining

    # 导入配置文件
    def parse_config_file(self, path: str, final: bool = True) -> None:
        config = {"__file__": os.path.abspath(path)}
        with open(path, "rb") as f:
            exec_in(native_str(f.read()), config, config)
        for name in config:
            normalized = self._normalize_name(name)
            if normalized in self._options:
                option = self._options[normalized]
                if option.multiple:
                    if not isinstance(config[name], (list, str)):
                        raise Error(
                            "Option %r is required to be a list of %s "
                            "or a comma-separated string"
                            % (option.name, option.type.__name__)
                        )

                if type(config[name]) == str and option.type != str:
                    option.parse(config[name])
                else:
                    option.set(config[name])

        if final:
            self.run_parse_callbacks()

    def print_help(self, file: TextIO = None) -> None:
        """Prints all the command line options to stderr (or another file)."""
        if file is None:
            file = sys.stderr
        print("Usage: %s [OPTIONS]" % sys.argv[0], file=file)
        print("\nOptions:\n", file=file)
        by_group = {}  # type: Dict[str, List[_Option]]
        for option in self._options.values():
            by_group.setdefault(option.group_name, []).append(option)

        for filename, o in sorted(by_group.items()):
            if filename:
                print("\n%s options:\n" % os.path.normpath(filename), file=file)
            o.sort(key=lambda option: option.name)
            for option in o:
                # Always print names with dashes in a CLI context.
                prefix = self._normalize_name(option.name)
                if option.metavar:
                    prefix += "=" + option.metavar
                description = option.help or ""
                if option.default is not None and option.default != "":
                    description += " (default %s)" % option.default
                lines = textwrap.wrap(description, 79 - 35)
                if len(prefix) > 30 or len(lines) == 0:
                    lines.insert(0, "")
                print("  --%-30s %s" % (prefix, lines[0]), file=file)
                for line in lines[1:]:
                    print("%-34s %s" % (" ", line), file=file)
        print(file=file)

    def _help_callback(self, value: bool) -> None:
        if value:
            self.print_help()
            sys.exit(0)

    def add_parse_callback(self, callback: Callable[[], None]) -> None:
        """Adds a parse callback, to be invoked when option parsing is done."""
        self._parse_callbacks.append(callback)

    def run_parse_callbacks(self) -> None:
        for callback in self._parse_callbacks:
            callback()

    def mockable(self) -> "_Mockable":
        """Returns a wrapper around self that is compatible with
        `mock.patch <unittest.mock.patch>`.

        The `mock.patch <unittest.mock.patch>` function (included in
        the standard library `unittest.mock` package since Python 3.3,
        or in the third-party ``mock`` package for older versions of
        Python) is incompatible with objects like ``options`` that
        override ``__getattr__`` and ``__setattr__``.  This function
        returns an object that can be used with `mock.patch.object
        <unittest.mock.patch.object>` to modify option values::

            with mock.patch.object(options.mockable(), 'name', value):
                assert options.name == value
        """
        return _Mockable(self)


class _Mockable(object):
    """`mock.patch` compatible wrapper for `OptionParser`.

    As of ``mock`` version 1.0.1, when an object uses ``__getattr__``
    hooks instead of ``__dict__``, ``patch.__exit__`` tries to delete
    the attribute it set instead of setting a new one (assuming that
    the object does not catpure ``__setattr__``, so the patch
    created a new attribute in ``__dict__``).

    _Mockable's getattr and setattr pass through to the underlying
    OptionParser, and delattr undoes the effect of a previous setattr.
    """

    def __init__(self, options: OptionParser) -> None:
        # Modify __dict__ directly to bypass __setattr__
        self.__dict__["_options"] = options
        self.__dict__["_originals"] = {}

    def __getattr__(self, name: str) -> Any:
        return getattr(self._options, name)

    def __setattr__(self, name: str, value: Any) -> None:
        assert name not in self._originals, "don't reuse mockable objects"
        self._originals[name] = getattr(self._options, name)
        setattr(self._options, name, value)

    def __delattr__(self, name: str) -> None:
        setattr(self._options, name, self._originals.pop(name))


class _Option(object):
    # This class could almost be made generic, but the way the types
    # interact with the multiple argument makes this tricky. (default
    # and the callback use List[T], but type is still Type[T]).
    UNSET = object()

    def __init__(
        self,
        name: str,
        default: Any = None,
        type: type = None,
        help: str = None,
        metavar: str = None,
        multiple: bool = False,
        file_name: str = None,
        group_name: str = None,
        callback: Callable[[Any], None] = None,
    ) -> None:
        if default is None and multiple:
            default = []
        self.name = name
        if type is None:
            raise ValueError("type must not be None")
        self.type = type
        self.help = help
        self.metavar = metavar
        self.multiple = multiple
        self.file_name = file_name
        self.group_name = group_name
        self.callback = callback
        self.default = default
        self._value = _Option.UNSET  # type: Any

    def value(self) -> Any:
        return self.default if self._value is _Option.UNSET else self._value

    def parse(self, value: str) -> Any:
        _parse = {
            datetime.datetime: self._parse_datetime,
            datetime.timedelta: self._parse_timedelta,
            bool: self._parse_bool,
            basestring_type: self._parse_string,
        }.get(
            self.type, self.type
        )  # type: Callable[[str], Any]
        if self.multiple:
            self._value = []
            for part in value.split(","):
                if issubclass(self.type, numbers.Integral):
                    # allow ranges of the form X:Y (inclusive at both ends)
                    lo_str, _, hi_str = part.partition(":")
                    lo = _parse(lo_str)
                    hi = _parse(hi_str) if hi_str else lo
                    self._value.extend(range(lo, hi + 1))
                else:
                    self._value.append(_parse(part))
        else:
            self._value = _parse(value)
        if self.callback is not None:
            self.callback(self._value)
        return self.value()

    def set(self, value: Any) -> None:
        if self.multiple:
            if not isinstance(value, list):
                raise Error(
                    "Option %r is required to be a list of %s"
                    % (self.name, self.type.__name__)
                )
            for item in value:
                if item is not None and not isinstance(item, self.type):
                    raise Error(
                        "Option %r is required to be a list of %s"
                        % (self.name, self.type.__name__)
                    )
        else:
            if value is not None and not isinstance(value, self.type):
                raise Error(
                    "Option %r is required to be a %s (%s given)"
                    % (self.name, self.type.__name__, type(value))
                )
        self._value = value
        if self.callback is not None:
            self.callback(self._value)

    # Supported date/time formats in our options
    _DATETIME_FORMATS = [
        "%a %b %d %H:%M:%S %Y",
        "%Y-%m-%d %H:%M:%S",
        "%Y-%m-%d %H:%M",
        "%Y-%m-%dT%H:%M",
        "%Y%m%d %H:%M:%S",
        "%Y%m%d %H:%M",
        "%Y-%m-%d",
        "%Y%m%d",
        "%H:%M:%S",
        "%H:%M",
    ]

    def _parse_datetime(self, value: str) -> datetime.datetime:
        for format in self._DATETIME_FORMATS:
            try:
                return datetime.datetime.strptime(value, format)
            except ValueError:
                pass
        raise Error("Unrecognized date/time format: %r" % value)

    _TIMEDELTA_ABBREV_DICT = {
        "h": "hours",
        "m": "minutes",
        "min": "minutes",
        "s": "seconds",
        "sec": "seconds",
        "ms": "milliseconds",
        "us": "microseconds",
        "d": "days",
        "w": "weeks",
    }

    _FLOAT_PATTERN = r"[-+]?(?:\d+(?:\.\d*)?|\.\d+)(?:[eE][-+]?\d+)?"

    _TIMEDELTA_PATTERN = re.compile(
        r"\s*(%s)\s*(\w*)\s*" % _FLOAT_PATTERN, re.IGNORECASE
    )

    def _parse_timedelta(self, value: str) -> datetime.timedelta:
        try:
            sum = datetime.timedelta()
            start = 0
            while start < len(value):
                m = self._TIMEDELTA_PATTERN.match(value, start)
                if not m:
                    raise Exception()
                num = float(m.group(1))
                units = m.group(2) or "seconds"
                units = self._TIMEDELTA_ABBREV_DICT.get(units, units)
                sum += datetime.timedelta(**{units: num})
                start = m.end()
            return sum
        except Exception:
            raise

    def _parse_bool(self, value: str) -> bool:
        return value.lower() not in ("false", "0", "f")

    def _parse_string(self, value: str) -> str:
        return _unicode(value)


options = OptionParser()
"""Global options object.

All defined options are available as attributes on this object.
"""


def define(
    name: str,
    default: Any = None,
    type: type = None,
    help: str = None,
    metavar: str = None,
    multiple: bool = False,
    group: str = None,
    callback: Callable[[Any], None] = None,
) -> None:
    """Defines an option in the global namespace.

    See `OptionParser.define`.
    """
    return options.define(
        name,
        default=default,
        type=type,
        help=help,
        metavar=metavar,
        multiple=multiple,
        group=group,
        callback=callback,
    )


def parse_command_line(args: List[str] = None, final: bool = True) -> List[str]:
    """Parses global options from the command line.

    See `OptionParser.parse_command_line`.
    """
    return options.parse_command_line(args, final=final)


def parse_config_file(path: str, final: bool = True) -> None:
    """Parses global options from a config file.

    See `OptionParser.parse_config_file`.
    """
    return options.parse_config_file(path, final=final)


def print_help(file: TextIO = None) -> None:
    """Prints all the command line options to stderr (or another file).

    See `OptionParser.print_help`.
    """
    return options.print_help(file)


def add_parse_callback(callback: Callable[[], None]) -> None:
    """Adds a parse callback, to be invoked when option parsing is done.

    See `OptionParser.add_parse_callback`
    """
    options.add_parse_callback(callback)


# Default options
define_logging_options(options)
