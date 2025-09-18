# tornado 之 routing.py
## 摘要

import re
from functools import partial

from tornado import httputil
from tornado.httpserver import _CallableAdapter
from tornado.escape import url_escape, url_unescape, utf8
from tornado.log import app_log
from tornado.util import basestring_type, import_object, re_unescape, unicode_type

from typing import Any, Union, Optional, Awaitable, List, Dict, Pattern, Tuple, overload


## 抽象的路由器接口
class Router(httputil.HTTPServerConnectionDelegate):
    def find_handler(
        self, request: httputil.HTTPServerRequest, **kwargs: Any
    ) -> Optional[httputil.HTTPMessageDelegate]:
        raise NotImplementedError()

    def start_request(
        self, server_conn: object, request_conn: httputil.HTTPConnection
    ) -> httputil.HTTPMessageDelegate:
        return _RoutingDelegate(self, server_conn, request_conn)

## 支持命名路由，支持将它们反转到原始的urls
class ReversibleRouter(Router):
    def reverse_url(self, name: str, *args: Any) -> Optional[str]:
        raise NotImplementedError()

## 路由代理
class _RoutingDelegate(httputil.HTTPMessageDelegate):
    def __init__(
        self, router: Router, server_conn: object, request_conn: httputil.HTTPConnection
    ) -> None:
        self.server_conn = server_conn
        self.request_conn = request_conn
        self.delegate = None  # type: Optional[httputil.HTTPMessageDelegate]
        self.router = router  # type: Router

    def headers_received(
        self,
        start_line: Union[httputil.RequestStartLine, httputil.ResponseStartLine],
        headers: httputil.HTTPHeaders,
    ) -> Optional[Awaitable[None]]:
        assert isinstance(start_line, httputil.RequestStartLine)
        request = httputil.HTTPServerRequest(
            connection=self.request_conn,
            server_connection=self.server_conn,
            start_line=start_line,
            headers=headers,
        )

        self.delegate = self.router.find_handler(request)
        if self.delegate is None:
            app_log.debug(
                "Delegate for %s %s request not found",
                start_line.method,
                start_line.path,
            )
            self.delegate = _DefaultMessageDelegate(self.request_conn)

        return self.delegate.headers_received(start_line, headers)

    def data_received(self, chunk: bytes) -> Optional[Awaitable[None]]:
        assert self.delegate is not None
        return self.delegate.data_received(chunk)

    def finish(self) -> None:
        assert self.delegate is not None
        self.delegate.finish()

    def on_connection_close(self) -> None:
        assert self.delegate is not None
        self.delegate.on_connection_close()

## 默认的消息代理
class _DefaultMessageDelegate(httputil.HTTPMessageDelegate):
    def __init__(self, connection: httputil.HTTPConnection) -> None:
        self.connection = connection

    def finish(self) -> None:
        self.connection.write_headers(
            httputil.ResponseStartLine("HTTP/1.1", 404, "Not Found"),
            httputil.HTTPHeaders(),
        )
        self.connection.finish()


# 规则列表
_RuleList = List[
    Union[
        "Rule",
        List[Any],  # Can't do detailed typechecking of lists.
        Tuple[Union[str, "Matcher"], Any],
        Tuple[Union[str, "Matcher"], Any, Dict[str, Any]],
        Tuple[Union[str, "Matcher"], Any, Dict[str, Any], str],
    ]
]

## 规则路由器
class RuleRouter(Router):
    def __init__(self, rules: _RuleList = None) -> None:
        # 所有的规则对象的列表
        self.rules = []  # type: List[Rule]
        if rules:
            self.add_rules(rules)

    def add_rules(self, rules: _RuleList) -> None:
        # 向路由器添加规则
        for rule in rules:
            if isinstance(rule, (tuple, list)):
                assert len(rule) in (2, 3, 4)
                if isinstance(rule[0], basestring_type):
                    rule = Rule(PathMatches(rule[0]), *rule[1:])
                else:
                    rule = Rule(*rule)

            self.rules.append(self.process_rule(rule))

    # 重写此方法可以对规则做预处理
    def process_rule(self, rule: "Rule") -> "Rule":
        return rule

    # 寻找规则代理(http msg delegate)
    def find_handler(
        self, request: httputil.HTTPServerRequest, **kwargs: Any
    ) -> Optional[httputil.HTTPMessageDelegate]:
        for rule in self.rules:
            target_params = rule.matcher.match(request)
            if target_params is not None:
                if rule.target_kwargs:
                    target_params["target_kwargs"] = rule.target_kwargs

                delegate = self.get_target_delegate(
                    rule.target, request, **target_params
                )

                if delegate is not None:
                    return delegate

        return None

    # 返回路由的 消息代理
    def get_target_delegate(
        self, target: Any, request: httputil.HTTPServerRequest, **target_params: Any
    ) -> Optional[httputil.HTTPMessageDelegate]:
        # 如果是路由器对象，寻找代理
        if isinstance(target, Router):
            return target.find_handler(request, **target_params)
        # 如果是服务器连接代理对象
        elif isinstance(target, httputil.HTTPServerConnectionDelegate):
            assert request.connection is not None
            return target.start_request(request.server_connection, request.connection)
        # 如果是可调用方法
        elif callable(target):
            assert request.connection is not None
            return _CallableAdapter(
                partial(target, **target_params), request.connection
            )

        return None

# 可重构原始uri的规则路由器
class ReversibleRuleRouter(ReversibleRouter, RuleRouter):
    def __init__(self, rules: _RuleList = None) -> None:
        # 名称和规则映射表
        self.named_rules = {}  # type: Dict[str, Any]
        super(ReversibleRuleRouter, self).__init__(rules)

    # 重构父类的解析路由方法
    def process_rule(self, rule: "Rule") -> "Rule":
        rule = super(ReversibleRuleRouter, self).process_rule(rule)
        # 为每个规则添加名称映射
        if rule.name:
            if rule.name in self.named_rules:
                app_log.warning(
                    "Multiple handlers named %s; replacing previous value", rule.name
                )
            self.named_rules[rule.name] = rule

        return rule

    # 重构uri
    def reverse_url(self, name: str, *args: Any) -> Optional[str]:
        if name in self.named_rules:
            return self.named_rules[name].matcher.reverse(*args)

        for rule in self.rules:
            if isinstance(rule.target, ReversibleRouter):
                reversed_url = rule.target.reverse_url(name, *args)
                if reversed_url is not None:
                    return reversed_url

        return None

# 规则
class Rule(object):
    """A routing rule."""

    def __init__(
        self,
        matcher: "Matcher",
        target: Any,
        target_kwargs: Dict[str, Any] = None,
        name: str = None,
    ) -> None:
        """Constructs a Rule instance.

        :arg Matcher matcher: a `Matcher` instance used for determining
            whether the rule should be considered a match for a specific
            request.
        :arg target: a Rule's target (typically a ``RequestHandler`` or
            `~.httputil.HTTPServerConnectionDelegate` subclass or even a nested `Router`,
            depending on routing implementation).
        :arg dict target_kwargs: a dict of parameters that can be useful
            at the moment of target instantiation (for example, ``status_code``
            for a ``RequestHandler`` subclass). They end up in
            ``target_params['target_kwargs']`` of `RuleRouter.get_target_delegate`
            method.
        :arg str name: the name of the rule that can be used to find it
            in `ReversibleRouter.reverse_url` implementation.
        """
        if isinstance(target, str):
            # import the Module and instantiate the class
            # Must be a fully qualified name (module.ClassName)
            target = import_object(target)

        # 规则匹配器
        self.matcher = matcher  # type: Matcher
        # 规则对象本身
        self.target = target
        # 参数字典
        self.target_kwargs = target_kwargs if target_kwargs else {}
        # 反向url的实现名称
        self.name = name

    # 反转url
    def reverse(self, *args: Any) -> Optional[str]:
        return self.matcher.reverse(*args)

    def __repr__(self) -> str:
        return "%s(%r, %s, kwargs=%r, name=%r)" % (
            self.__class__.__name__,
            self.matcher,
            self.target,
            self.target_kwargs,
            self.name,
        )

# 规则匹配器
class Matcher(object):
    # 将规则与请求匹配
    def match(self, request: httputil.HTTPServerRequest) -> Optional[Dict[str, Any]]:
        raise NotImplementedError()
    # 反转url
    def reverse(self, *args: Any) -> Optional[str]:
        """Reconstructs full url from matcher instance and additional arguments."""
        return None

# 匹配所有的请求
class AnyMatches(Matcher):
    def match(self, request: httputil.HTTPServerRequest) -> Optional[Dict[str, Any]]:
        return {}

# 匹配指定主机的请求
class HostMatches(Matcher):
    def __init__(self, host_pattern: Union[str, Pattern]) -> None:
        # 主机匹配器
        if isinstance(host_pattern, basestring_type):
            if not host_pattern.endswith("$"):
                host_pattern += "$"
            self.host_pattern = re.compile(host_pattern)
        else:
            self.host_pattern = host_pattern
    # 匹配请求的主机名
    def match(self, request: httputil.HTTPServerRequest) -> Optional[Dict[str, Any]]:
        if self.host_pattern.match(request.host_name):
            return {}

        return None

# 匹配来自主机的请求，如果是经过转发之后的请求则始终返回不匹配
class DefaultHostMatches(Matcher):
    def __init__(self, application: Any, host_pattern: Pattern) -> None:
        self.application = application
        self.host_pattern = host_pattern

    def match(self, request: httputil.HTTPServerRequest) -> Optional[Dict[str, Any]]:
        # Look for default host if not behind load balancer (for debugging)
        if "X-Real-Ip" not in request.headers:
            if self.host_pattern.match(self.application.default_host):
                return {}
        return None

# 路径匹配器
class PathMatches(Matcher):
    def __init__(self, path_pattern: Union[str, Pattern]) -> None:
        if isinstance(path_pattern, basestring_type):
            if not path_pattern.endswith("$"):
                path_pattern += "$"
            self.regex = re.compile(path_pattern)
        else:
            self.regex = path_pattern

        assert len(self.regex.groupindex) in (0, self.regex.groups), (
            "groups in url regexes must either be all named or all "
            "positional: %r" % self.regex.pattern
        )

        self._path, self._group_count = self._find_groups()

    def match(self, request: httputil.HTTPServerRequest) -> Optional[Dict[str, Any]]:
        match = self.regex.match(request.path)
        if match is None:
            return None
        if not self.regex.groups:
            return {}

        path_args = []  # type: List[bytes]
        path_kwargs = {}  # type: Dict[str, bytes]

        # Pass matched groups to the handler.  Since
        # match.groups() includes both named and
        # unnamed groups, we want to use either groups
        # or groupdict but not both.
        if self.regex.groupindex:
            path_kwargs = dict(
                (str(k), _unquote_or_none(v)) for (k, v) in match.groupdict().items()
            )
        else:
            path_args = [_unquote_or_none(s) for s in match.groups()]

        return dict(path_args=path_args, path_kwargs=path_kwargs)

    def reverse(self, *args: Any) -> Optional[str]:
        if self._path is None:
            raise ValueError("Cannot reverse url regex " + self.regex.pattern)
        assert len(args) == self._group_count, (
            "required number of arguments " "not found"
        )
        if not len(args):
            return self._path
        converted_args = []
        for a in args:
            if not isinstance(a, (unicode_type, bytes)):
                a = str(a)
            converted_args.append(url_escape(utf8(a), plus=False))
        return self._path % tuple(converted_args)

    def _find_groups(self) -> Tuple[Optional[str], Optional[int]]:
        """Returns a tuple (reverse string, group count) for a url.

        For example: Given the url pattern /([0-9]{4})/([a-z-]+)/, this method
        would return ('/%s/%s/', 2).
        """
        pattern = self.regex.pattern
        if pattern.startswith("^"):
            pattern = pattern[1:]
        if pattern.endswith("$"):
            pattern = pattern[:-1]

        if self.regex.groups != pattern.count("("):
            # The pattern is too complicated for our simplistic matching,
            # so we can't support reversing it.
            return None, None

        pieces = []
        for fragment in pattern.split("("):
            if ")" in fragment:
                paren_loc = fragment.index(")")
                if paren_loc >= 0:
                    pieces.append("%s" + fragment[paren_loc + 1 :])
            else:
                try:
                    unescaped_fragment = re_unescape(fragment)
                except ValueError:
                    # If we can't unescape part of it, we can't
                    # reverse this url.
                    return (None, None)
                pieces.append(unescaped_fragment)

        return "".join(pieces), self.regex.groups

# 向后兼容
class URLSpec(Rule):

    def __init__(
        self,
        pattern: Union[str, Pattern],
        handler: Any,
        kwargs: Dict[str, Any] = None,
        name: str = None,
    ) -> None:
        matcher = PathMatches(pattern)
        super(URLSpec, self).__init__(matcher, handler, kwargs, name)

        self.regex = matcher.regex
        self.handler_class = self.target
        self.kwargs = kwargs

    def __repr__(self) -> str:
        return "%s(%r, %s, kwargs=%r, name=%r)" % (
            self.__class__.__name__,
            self.regex.pattern,
            self.handler_class,
            self.kwargs,
            self.name,
        )


@overload
def _unquote_or_none(s: str) -> bytes:
    pass


@overload  # noqa: F811
def _unquote_or_none(s: None) -> None:
    pass


def _unquote_or_none(s: Optional[str]) -> Optional[bytes]:  # noqa: F811
    if s is None:
        return s
    return url_unescape(s, encoding=None, plus=False)
