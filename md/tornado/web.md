[TOC]
# tornado 之 web.py
## 摘要

import base64
import binascii
import datetime
import email.utils
import functools
import gzip
import hashlib
import hmac
import http.cookies
from inspect import isclass
from io import BytesIO
import mimetypes
import numbers
import os.path
import re
import sys
import threading
import time
import tornado
import traceback
import types
import urllib.parse
from urllib.parse import urlencode

from tornado.concurrent import Future, future_set_result_unless_cancelled
from tornado import escape
from tornado import gen
from tornado.httpserver import HTTPServer
from tornado import httputil
from tornado import iostream
import tornado.locale
from tornado import locale
from tornado.log import access_log, app_log, gen_log
from tornado import template
from tornado.escape import utf8, _unicode
from tornado.routing import (
    AnyMatches,
    DefaultHostMatches,
    HostMatches,
    ReversibleRouter,
    Rule,
    ReversibleRuleRouter,
    URLSpec,
    _RuleList,
)
from tornado.util import ObjectDict, unicode_type, _websocket_mask

url = URLSpec

from typing import (
    Dict,
    Any,
    Union,
    Optional,
    Awaitable,
    Tuple,
    List,
    Callable,
    Iterable,
    Generator,
    Type,
    cast,
    overload,
)
from types import TracebackType
import typing

if typing.TYPE_CHECKING:
    from typing import Set  # noqa: F401


# The following types are accepted by RequestHandler.set_header
# and related methods.
_HeaderTypes = Union[bytes, unicode_type, int, numbers.Integral, datetime.datetime]

_CookieSecretTypes = Union[str, bytes, Dict[int, str], Dict[int, bytes]]


MIN_SUPPORTED_SIGNED_VALUE_VERSION = 1
"""The oldest signed value version supported by this version of Tornado.

Signed values older than this version cannot be decoded.

.. versionadded:: 3.2.1
"""

MAX_SUPPORTED_SIGNED_VALUE_VERSION = 2
"""The newest signed value version supported by this version of Tornado.

Signed values newer than this version cannot be decoded.

.. versionadded:: 3.2.1
"""

DEFAULT_SIGNED_VALUE_VERSION = 2
"""The signed value version produced by `.RequestHandler.create_signed_value`.

May be overridden by passing a ``version`` keyword argument.

.. versionadded:: 3.2.1
"""

DEFAULT_SIGNED_VALUE_MIN_VERSION = 1
"""The oldest signed value accepted by `.RequestHandler.get_secure_cookie`.

May be overridden by passing a ``min_version`` keyword argument.

.. versionadded:: 3.2.1
"""


class _ArgDefaultMarker:
    pass


_ARG_DEFAULT = _ArgDefaultMarker()

## class RequestHandler
class RequestHandler(object):
    # 基础的请求处理程序

    SUPPORTED_METHODS = ("GET", "HEAD", "POST", "DELETE", "PATCH", "PUT", "OPTIONS")

    _template_loaders = {}  # type: Dict[str, template.BaseLoader]
    _template_loader_lock = threading.Lock()
    _remove_control_chars_regex = re.compile(r"[\x00-\x08\x0e-\x1f]")

    _stream_request_body = False

    # Will be set in _execute.
    _transforms = None  # type: List[OutputTransform]
    path_args = None  # type: List[str]
    path_kwargs = None  # type: Dict[str, str]

    def __init__(
        self,
        application: "Application",
        request: httputil.HTTPServerRequest,
        **kwargs: Any
    ) -> None:
        super(RequestHandler, self).__init__()
        # 应用程序
        self.application = application
        # 请求对象
        self.request = request
        # 
        self._headers_written = False
        # 
        self._finished = False
        self._auto_finish = True
        # 
        self._prepared_future = None
        self.ui = ObjectDict(
            (n, self._ui_method(m)) for n, m in application.ui_methods.items()
        )
        self.ui["_tt_modules"] = _UIModuleNamespace(self, application.ui_modules)
        self.ui["modules"] = self.ui["_tt_modules"]
        self.clear()
        assert self.request.connection is not None
        # 连接关闭回调函数
        self.request.connection.set_close_callback(  # type: ignore
            self.on_connection_close
        )
        self.initialize(**kwargs)  # type: ignore

    def _initialize(self) -> None:
        pass

    initialize = _initialize  # type: Callable[..., None]

    @property
    def settings(self) -> Dict[str, Any]:
        return self.application.settings

    # 未实现的方法
    def _unimplemented_method(self, *args: str, **kwargs: str) -> None:
        raise HTTPError(405)

    head = _unimplemented_method  # type: Callable[..., Optional[Awaitable[None]]]
    get = _unimplemented_method  # type: Callable[..., Optional[Awaitable[None]]]
    post = _unimplemented_method  # type: Callable[..., Optional[Awaitable[None]]]
    delete = _unimplemented_method  # type: Callable[..., Optional[Awaitable[None]]]
    patch = _unimplemented_method  # type: Callable[..., Optional[Awaitable[None]]]
    put = _unimplemented_method  # type: Callable[..., Optional[Awaitable[None]]]
    options = _unimplemented_method  # type: Callable[..., Optional[Awaitable[None]]]

    # 重写此方法以在请求处理之前进行初始化
    def prepare(self) -> Optional[Awaitable[None]]:
        pass

    # 重写此方法清理数据，记录日志之类的，这个方法在响应之后执行
    def on_finish(self) -> None:
        pass

    # 在响应过程中客户端断开连接时调用
    def on_connection_close(self) -> None:
        if _has_stream_request_body(self.__class__):
            if not self.request._body_future.done():
                self.request._body_future.set_exception(iostream.StreamClosedError())
                self.request._body_future.exception()

    # 重置响应的标题和内容，在实例化时被调用
    def clear(self) -> None:
        self._headers = httputil.HTTPHeaders(
            {
                "Server": "TornadoServer/%s" % tornado.version,
                "Content-Type": "text/html; charset=UTF-8",
                "Date": httputil.format_timestamp(time.time()),
            }
        )
        self.set_default_headers()
        self._write_buffer = []  # type: List[bytes]
        self._status_code = 200
        self._reason = httputil.responses[200]

    # 重写此方法设置默认的头部，处理异常时可能会覆盖
    def set_default_headers(self) -> None:
        pass

    # 设置状态和原因
    def set_status(self, status_code: int, reason: str = None) -> None:
        self._status_code = status_code
        if reason is not None:
            self._reason = escape.native_str(reason)
        else:
            self._reason = httputil.responses.get(status_code, "Unknown")

    # 获取当前的状态码
    def get_status(self) -> int:
        return self._status_code

    # 设置响应头的参数和值
    def set_header(self, name: str, value: _HeaderTypes) -> None:
        self._headers[name] = self._convert_header_value(value)

    # 添加响应头的参数和值，可以多次调用设置多个值
    def add_header(self, name: str, value: _HeaderTypes) -> None:
        self._headers.add(name, self._convert_header_value(value))

    # 删除头部的参数，此方法不适用于多值参数
    def clear_header(self, name: str) -> None:
        if name in self._headers:
            del self._headers[name]

    _INVALID_HEADER_CHAR_RE = re.compile(r"[\x00-\x1f]")

    # 检查响应头的值，转换成标准的类型
    def _convert_header_value(self, value: _HeaderTypes) -> str:
        if isinstance(value, str):
            retval = value
        elif isinstance(value, bytes):  # py3
            retval = value.decode("latin1")
        elif isinstance(value, unicode_type):  # py2
            retval = escape.utf8(value)
        elif isinstance(value, numbers.Integral):
            return str(value)
        elif isinstance(value, datetime.datetime):
            return httputil.format_timestamp(value)
        else:
            raise TypeError("Unsupported header value %r" % value)
        # 不允许 \n 进入标头
        if RequestHandler._INVALID_HEADER_CHAR_RE.search(retval):
            raise ValueError("Unsafe header value %r", retval)
        return retval

    @overload
    def get_argument(self, name: str, default: str, strip: bool = True) -> str:
        pass

    @overload  # noqa: F811
    def get_argument(
        self, name: str, default: _ArgDefaultMarker = _ARG_DEFAULT, strip: bool = True
    ) -> str:
        pass

    @overload  # noqa: F811
    def get_argument(
        self, name: str, default: None, strip: bool = True
    ) -> Optional[str]:
        pass

    # 在请求头和主体中查找参数，如果不存在返回默认值，没有给默认值则抛出异常
    def get_argument(  # noqa: F811
        self,
        name: str,
        default: Union[None, str, _ArgDefaultMarker] = _ARG_DEFAULT,
        strip: bool = True,
    ) -> Optional[str]:
        return self._get_argument(name, default, self.request.arguments, strip)

    # 在请求头和主体中查找参数，返回所有值的列表
    def get_arguments(self, name: str, strip: bool = True) -> List[str]:
        assert isinstance(strip, bool)
        return self._get_arguments(name, self.request.arguments, strip)

    # 在请求体中查找参数，不存在则返回默认值，默认值不存在则抛出异常
    def get_body_argument(
        self,
        name: str,
        default: Union[None, str, _ArgDefaultMarker] = _ARG_DEFAULT,
        strip: bool = True,
    ) -> Optional[str]:
        return self._get_argument(name, default, self.request.body_arguments, strip)

    # 返回一个列表
    def get_body_arguments(self, name: str, strip: bool = True) -> List[str]:
        return self._get_arguments(name, self.request.body_arguments, strip)

    # 在查询字符串中查找参数，不存在则返回默认值，默认值不存在抛出异常
    def get_query_argument(
        self,
        name: str,
        default: Union[None, str, _ArgDefaultMarker] = _ARG_DEFAULT,
        strip: bool = True,
    ) -> Optional[str]:
        return self._get_argument(name, default, self.request.query_arguments, strip)

    # 返回一个列表
    def get_query_arguments(self, name: str, strip: bool = True) -> List[str]:
        return self._get_arguments(name, self.request.query_arguments, strip)

    # 从目标中获取参数，只返回第一个
    def _get_argument(
        self,
        name: str,
        default: Union[None, str, _ArgDefaultMarker],
        source: Dict[str, List[bytes]],
        strip: bool = True,
    ) -> Optional[str]:
        args = self._get_arguments(name, source, strip=strip)
        if not args:
            if isinstance(default, _ArgDefaultMarker):
                raise MissingArgumentError(name)
            return default
        return args[-1]

    # 从目标中获取参数列表
    def _get_arguments(
        self, name: str, source: Dict[str, List[bytes]], strip: bool = True
    ) -> List[str]:
        values = []
        for v in source.get(name, []):
            s = self.decode_argument(v, name=name)
            # 去掉奇怪的控制字符
            if isinstance(s, unicode_type):
                s = RequestHandler._remove_control_chars_regex.sub(" ", s)
            if strip:
                s = s.strip()
            values.append(s)
        return values

    # 解码参数的值
    def decode_argument(self, value: bytes, name: str = None) -> str:
        try:
            return _unicode(value)
        except UnicodeDecodeError:
            raise HTTPError(
                400, "Invalid unicode in %s: %r" % (name or "url", value[:40])
            )

    # cookies
    @property
    def cookies(self) -> Dict[str, http.cookies.Morsel]:
        return self.request.cookies

    # 返回具有给定名称的cookie的值，仅仅返回请求中的cookie
    def get_cookie(self, name: str, default: str = None) -> Optional[str]:
        if self.request.cookies is not None and name in self.request.cookies:
            return self.request.cookies[name].value
        return default

    # 设置响应的cookie
    def set_cookie(
        self,
        name: str,
        value: Union[str, bytes],
        domain: str = None,
        expires: Union[float, Tuple, datetime.datetime] = None,
        path: str = "/",
        expires_days: int = None,
        **kwargs: Any
    ) -> None:
        name = escape.native_str(name)
        value = escape.native_str(value)
        # cookie中不应该出现的字符
        if re.search(r"[\x00-\x20]", name + value):
            raise ValueError("Invalid cookie %r: %r" % (name, value))
        if not hasattr(self, "_new_cookie"):
            self._new_cookie = http.cookies.SimpleCookie()
        if name in self._new_cookie:
            del self._new_cookie[name]
        self._new_cookie[name] = value
        morsel = self._new_cookie[name]
        # 设置cookie适用的域和路径
        if domain:
            morsel["domain"] = domain
        if expires_days is not None and not expires:
            expires = datetime.datetime.utcnow() + datetime.timedelta(days=expires_days)
        if expires:
            morsel["expires"] = httputil.format_timestamp(expires)
        if path:
            morsel["path"] = path
        for k, v in kwargs.items():
            if k == "max_age":
                k = "max-age"

            if k in ["httponly", "secure"] and not v:
                continue

            morsel[k] = v

    # 清除cookie的参数，要等到下次请求才会生效
    def clear_cookie(self, name: str, path: str = "/", domain: str = None) -> None:
        expires = datetime.datetime.utcnow() - datetime.timedelta(days=365)
        self.set_cookie(name, value="", path=path, expires=expires, domain=domain)

    # 删除所有请求中的cookie，下次请求才能生效
    def clear_all_cookies(self, path: str = "/", domain: str = None) -> None:
        for name in self.request.cookies:
            self.clear_cookie(name, path=path, domain=domain)

    # 设置安全的cookie，进行签名
    def set_secure_cookie(
        self,
        name: str,
        value: Union[str, bytes],
        expires_days: int = 30,
        version: int = None,
        **kwargs: Any
    ) -> None:
        self.set_cookie(
            name,
            self.create_signed_value(name, value, version=version),
            expires_days=expires_days,
            **kwargs
        )

    # 对cookie进行了加密等操作
    def create_signed_value(
        self, name: str, value: Union[str, bytes], version: int = None
    ) -> bytes:
        self.require_setting("cookie_secret", "secure cookies")
        secret = self.application.settings["cookie_secret"]
        key_version = None
        if isinstance(secret, dict):
            if self.application.settings.get("key_version") is None:
                raise Exception("key_version setting must be used for secret_key dicts")
            key_version = self.application.settings["key_version"]

        return create_signed_value(
            secret, name, value, version=version, key_version=key_version
        )

    # 获取安全的cookie
    def get_secure_cookie(
        self,
        name: str,
        value: str = None,
        max_age_days: int = 31,
        min_version: int = None,
    ) -> Optional[bytes]:
        self.require_setting("cookie_secret", "secure cookies")
        if value is None:
            value = self.get_cookie(name)
        return decode_signed_value(
            self.application.settings["cookie_secret"],
            name,
            value,
            max_age_days=max_age_days,
            min_version=min_version,
        )

    # 获取签名的秘钥版本
    def get_secure_cookie_key_version(
        self, name: str, value: str = None
    ) -> Optional[int]:
        self.require_setting("cookie_secret", "secure cookies")
        if value is None:
            value = self.get_cookie(name)
        if value is None:
            return None
        return get_signature_key_version(value)

    # 发送一个重定向
    def redirect(self, url: str, permanent: bool = False, status: int = None) -> None:
        if self._headers_written:
            raise Exception("Cannot redirect after headers have been written")
        if status is None:
            status = 301 if permanent else 302
        else:
            assert isinstance(status, int) and 300 <= status <= 399
        self.set_status(status)
        self.set_header("Location", utf8(url))
        self.finish()

    # 把数据写入到缓冲区
    def write(self, chunk: Union[str, bytes, dict]) -> None:
        if self._finished:
            raise RuntimeError("Cannot write() after finish()")
        if not isinstance(chunk, (bytes, unicode_type, dict)):
            message = "write() only accepts bytes, unicode, and dict objects"
            if isinstance(chunk, list):
                message += (
                    ". Lists not accepted for security reasons; see "
                    + "http://www.tornadoweb.org/en/stable/web.html#tornado.web.RequestHandler.write"  # noqa: E501
                )
            raise TypeError(message)
        if isinstance(chunk, dict):
            chunk = escape.json_encode(chunk)
            self.set_header("Content-Type", "application/json; charset=UTF-8")
        chunk = utf8(chunk)
        self._write_buffer.append(chunk)

    # 把参数导入到模板并返回
    def render(self, template_name: str, **kwargs: Any) -> "Future[None]":
        if self._finished:
            raise RuntimeError("Cannot render() after finish()")
        html = self.render_string(template_name, **kwargs)

        # Insert the additional JS and CSS added by the modules on the page
        js_embed = []
        js_files = []
        css_embed = []
        css_files = []
        html_heads = []
        html_bodies = []
        for module in getattr(self, "_active_modules", {}).values():
            embed_part = module.embedded_javascript()
            if embed_part:
                js_embed.append(utf8(embed_part))
            file_part = module.javascript_files()
            if file_part:
                if isinstance(file_part, (unicode_type, bytes)):
                    js_files.append(_unicode(file_part))
                else:
                    js_files.extend(file_part)
            embed_part = module.embedded_css()
            if embed_part:
                css_embed.append(utf8(embed_part))
            file_part = module.css_files()
            if file_part:
                if isinstance(file_part, (unicode_type, bytes)):
                    css_files.append(_unicode(file_part))
                else:
                    css_files.extend(file_part)
            head_part = module.html_head()
            if head_part:
                html_heads.append(utf8(head_part))
            body_part = module.html_body()
            if body_part:
                html_bodies.append(utf8(body_part))

        if js_files:
            # Maintain order of JavaScript files given by modules
            js = self.render_linked_js(js_files)
            sloc = html.rindex(b"</body>")
            html = html[:sloc] + utf8(js) + b"\n" + html[sloc:]
        if js_embed:
            js_bytes = self.render_embed_js(js_embed)
            sloc = html.rindex(b"</body>")
            html = html[:sloc] + js_bytes + b"\n" + html[sloc:]
        if css_files:
            css = self.render_linked_css(css_files)
            hloc = html.index(b"</head>")
            html = html[:hloc] + utf8(css) + b"\n" + html[hloc:]
        if css_embed:
            css_bytes = self.render_embed_css(css_embed)
            hloc = html.index(b"</head>")
            html = html[:hloc] + css_bytes + b"\n" + html[hloc:]
        if html_heads:
            hloc = html.index(b"</head>")
            html = html[:hloc] + b"".join(html_heads) + b"\n" + html[hloc:]
        if html_bodies:
            hloc = html.index(b"</body>")
            html = html[:hloc] + b"".join(html_bodies) + b"\n" + html[hloc:]
        return self.finish(html)

    def render_linked_js(self, js_files: Iterable[str]) -> str:
        """Default method used to render the final js links for the
        rendered webpage.

        Override this method in a sub-classed controller to change the output.
        """
        paths = []
        unique_paths = set()  # type: Set[str]

        for path in js_files:
            if not is_absolute(path):
                path = self.static_url(path)
            if path not in unique_paths:
                paths.append(path)
                unique_paths.add(path)

        return "".join(
            '<script src="'
            + escape.xhtml_escape(p)
            + '" type="text/javascript"></script>'
            for p in paths
        )

    def render_embed_js(self, js_embed: Iterable[bytes]) -> bytes:
        """Default method used to render the final embedded js for the
        rendered webpage.

        Override this method in a sub-classed controller to change the output.
        """
        return (
            b'<script type="text/javascript">\n//<![CDATA[\n'
            + b"\n".join(js_embed)
            + b"\n//]]>\n</script>"
        )

    def render_linked_css(self, css_files: Iterable[str]) -> str:
        """Default method used to render the final css links for the
        rendered webpage.

        Override this method in a sub-classed controller to change the output.
        """
        paths = []
        unique_paths = set()  # type: Set[str]

        for path in css_files:
            if not is_absolute(path):
                path = self.static_url(path)
            if path not in unique_paths:
                paths.append(path)
                unique_paths.add(path)

        return "".join(
            '<link href="' + escape.xhtml_escape(p) + '" '
            'type="text/css" rel="stylesheet"/>'
            for p in paths
        )

    def render_embed_css(self, css_embed: Iterable[bytes]) -> bytes:
        """Default method used to render the final embedded css for the
        rendered webpage.

        Override this method in a sub-classed controller to change the output.
        """
        return b'<style type="text/css">\n' + b"\n".join(css_embed) + b"\n</style>"

    def render_string(self, template_name: str, **kwargs: Any) -> bytes:
        """Generate the given template with the given arguments.

        We return the generated byte string (in utf8). To generate and
        write a template as a response, use render() above.
        """
        # If no template_path is specified, use the path of the calling file
        template_path = self.get_template_path()
        if not template_path:
            frame = sys._getframe(0)
            web_file = frame.f_code.co_filename
            while frame.f_code.co_filename == web_file:
                frame = frame.f_back
            assert frame.f_code.co_filename is not None
            template_path = os.path.dirname(frame.f_code.co_filename)
        with RequestHandler._template_loader_lock:
            if template_path not in RequestHandler._template_loaders:
                loader = self.create_template_loader(template_path)
                RequestHandler._template_loaders[template_path] = loader
            else:
                loader = RequestHandler._template_loaders[template_path]
        t = loader.load(template_name)
        namespace = self.get_template_namespace()
        namespace.update(kwargs)
        return t.generate(**namespace)

    def get_template_namespace(self) -> Dict[str, Any]:
        """Returns a dictionary to be used as the default template namespace.

        May be overridden by subclasses to add or modify values.

        The results of this method will be combined with additional
        defaults in the `tornado.template` module and keyword arguments
        to `render` or `render_string`.
        """
        namespace = dict(
            handler=self,
            request=self.request,
            current_user=self.current_user,
            locale=self.locale,
            _=self.locale.translate,
            pgettext=self.locale.pgettext,
            static_url=self.static_url,
            xsrf_form_html=self.xsrf_form_html,
            reverse_url=self.reverse_url,
        )
        namespace.update(self.ui)
        return namespace

    def create_template_loader(self, template_path: str) -> template.BaseLoader:
        """Returns a new template loader for the given path.

        May be overridden by subclasses.  By default returns a
        directory-based loader on the given path, using the
        ``autoescape`` and ``template_whitespace`` application
        settings.  If a ``template_loader`` application setting is
        supplied, uses that instead.
        """
        settings = self.application.settings
        if "template_loader" in settings:
            return settings["template_loader"]
        kwargs = {}
        if "autoescape" in settings:
            # autoescape=None means "no escaping", so we have to be sure
            # to only pass this kwarg if the user asked for it.
            kwargs["autoescape"] = settings["autoescape"]
        if "template_whitespace" in settings:
            kwargs["whitespace"] = settings["template_whitespace"]
        return template.Loader(template_path, **kwargs)

    # 刷新缓冲区到套接字
    def flush(self, include_footers: bool = False) -> "Future[None]":
        assert self.request.connection is not None
        chunk = b"".join(self._write_buffer)
        self._write_buffer = []
        if not self._headers_written:
            self._headers_written = True
            for transform in self._transforms:
                assert chunk is not None
                self._status_code, self._headers, chunk = transform.transform_first_chunk(
                    self._status_code, self._headers, chunk, include_footers
                )
            # Ignore the chunk and only write the headers for HEAD requests
            if self.request.method == "HEAD":
                chunk = b""

            # Finalize the cookie headers (which have been stored in a side
            # object so an outgoing cookie could be overwritten before it
            # is sent).
            if hasattr(self, "_new_cookie"):
                for cookie in self._new_cookie.values():
                    self.add_header("Set-Cookie", cookie.OutputString(None))

            start_line = httputil.ResponseStartLine("", self._status_code, self._reason)
            return self.request.connection.write_headers(
                start_line, self._headers, chunk
            )
        else:
            for transform in self._transforms:
                chunk = transform.transform_chunk(chunk, include_footers)
            # Ignore the chunk and only write the headers for HEAD requests
            if self.request.method != "HEAD":
                return self.request.connection.write(chunk)
            else:
                future = Future()  # type: Future[None]
                future.set_result(None)
                return future

    # 完成响应
    def finish(self, chunk: Union[str, bytes, dict] = None) -> "Future[None]":
        if self._finished:
            raise RuntimeError("finish() called twice")

        if chunk is not None:
            self.write(chunk)

        # Automatically support ETags and add the Content-Length header if
        # we have not flushed any content yet.
        if not self._headers_written:
            if (
                self._status_code == 200
                and self.request.method in ("GET", "HEAD")
                and "Etag" not in self._headers
            ):
                self.set_etag_header()
                if self.check_etag_header():
                    self._write_buffer = []
                    self.set_status(304)
            if self._status_code in (204, 304) or (
                self._status_code >= 100 and self._status_code < 200
            ):
                assert not self._write_buffer, (
                    "Cannot send body with %s" % self._status_code
                )
                self._clear_headers_for_304()
            elif "Content-Length" not in self._headers:
                content_length = sum(len(part) for part in self._write_buffer)
                self.set_header("Content-Length", content_length)

        assert self.request.connection is not None
        # Now that the request is finished, clear the callback we
        # set on the HTTPConnection (which would otherwise prevent the
        # garbage collection of the RequestHandler when there
        # are keepalive connections)
        self.request.connection.set_close_callback(None)  # type: ignore

        future = self.flush(include_footers=True)
        self.request.connection.finish()
        self._log()
        self._finished = True
        self.on_finish()
        self._break_cycles()
        return future

    def detach(self) -> iostream.IOStream:
        self._finished = True
        # TODO: add detach to HTTPConnection?
        return self.request.connection.detach()  # type: ignore

    def _break_cycles(self) -> None:
        self.ui = None  # type: ignore

    # 发送异常
    def send_error(self, status_code: int = 500, **kwargs: Any) -> None:
        if self._headers_written:
            gen_log.error("Cannot send error response after headers written")
            if not self._finished:
                # If we get an error between writing headers and finishing,
                # we are unlikely to be able to finish due to a
                # Content-Length mismatch. Try anyway to release the
                # socket.
                try:
                    self.finish()
                except Exception:
                    gen_log.error("Failed to flush partial response", exc_info=True)
            return
        self.clear()

        reason = kwargs.get("reason")
        if "exc_info" in kwargs:
            exception = kwargs["exc_info"][1]
            if isinstance(exception, HTTPError) and exception.reason:
                reason = exception.reason
        self.set_status(status_code, reason=reason)
        try:
            self.write_error(status_code, **kwargs)
        except Exception:
            app_log.error("Uncaught exception in write_error", exc_info=True)
        if not self._finished:
            self.finish()

    # 重写以实现异常页面
    def write_error(self, status_code: int, **kwargs: Any) -> None:
        if self.settings.get("serve_traceback") and "exc_info" in kwargs:
            # in debug mode, try to send a traceback
            self.set_header("Content-Type", "text/plain")
            for line in traceback.format_exception(*kwargs["exc_info"]):
                self.write(line)
            self.finish()
        else:
            self.finish(
                "<html><title>%(code)d: %(message)s</title>"
                "<body>%(code)d: %(message)s</body></html>"
                % {"code": status_code, "message": self._reason}
            )

    # 当前会话的区域设置
    @property
    def locale(self) -> tornado.locale.Locale:
        if not hasattr(self, "_locale"):
            loc = self.get_user_locale()
            if loc is not None:
                self._locale = loc
            else:
                self._locale = self.get_browser_locale()
                assert self._locale
        return self._locale

    @locale.setter
    def locale(self, value: tornado.locale.Locale) -> None:
        self._locale = value

    # 重写，返回经过身份验证的用户的区域设置
    def get_user_locale(self) -> Optional[tornado.locale.Locale]:
        return None

    # 从语言标头确定用户的区域设置
    def get_browser_locale(self, default: str = "en_US") -> tornado.locale.Locale:
        if "Accept-Language" in self.request.headers:
            languages = self.request.headers["Accept-Language"].split(",")
            locales = []
            for language in languages:
                parts = language.strip().split(";")
                if len(parts) > 1 and parts[1].startswith("q="):
                    try:
                        score = float(parts[1][2:])
                    except (ValueError, TypeError):
                        score = 0.0
                else:
                    score = 1.0
                locales.append((parts[0], score))
            if locales:
                locales.sort(key=lambda pair: pair[1], reverse=True)
                codes = [l[0] for l in locales]
                return locale.get(*codes)
        return locale.get(default)

    # 经过验证的用户
    @property
    def current_user(self) -> Any:
        if not hasattr(self, "_current_user"):
            self._current_user = self.get_current_user()
        return self._current_user

    # 设置用户名称
    @current_user.setter
    def current_user(self, value: Any) -> None:
        self._current_user = value

    # 覆盖以确定当前用户
    def get_current_user(self) -> Any:
        """Override to determine the current user from, e.g., a cookie.

        This method may not be a coroutine.
        """
        return None

    # 覆写以确定请求的登录 url
    def get_login_url(self) -> str:
        self.require_setting("login_url", "@tornado.web.authenticated")
        return self.application.settings["login_url"]

    # 获取模板的路径
    def get_template_path(self) -> Optional[str]:
        return self.application.settings.get("template_path")

    # 当前用户会话的令牌
    @property
    def xsrf_token(self) -> bytes:
        if not hasattr(self, "_xsrf_token"):
            version, token, timestamp = self._get_raw_xsrf_token()
            output_version = self.settings.get("xsrf_cookie_version", 2)
            cookie_kwargs = self.settings.get("xsrf_cookie_kwargs", {})
            if output_version == 1:
                self._xsrf_token = binascii.b2a_hex(token)
            elif output_version == 2:
                mask = os.urandom(4)
                self._xsrf_token = b"|".join(
                    [
                        b"2",
                        binascii.b2a_hex(mask),
                        binascii.b2a_hex(_websocket_mask(mask, token)),
                        utf8(str(int(timestamp))),
                    ]
                )
            else:
                raise ValueError("unknown xsrf cookie version %d", output_version)
            if version is None:
                if self.current_user and "expires_days" not in cookie_kwargs:
                    cookie_kwargs["expires_days"] = 30
                self.set_cookie("_xsrf", self._xsrf_token, **cookie_kwargs)
        return self._xsrf_token

    def _get_raw_xsrf_token(self) -> Tuple[Optional[int], bytes, float]:
        if not hasattr(self, "_raw_xsrf_token"):
            cookie = self.get_cookie("_xsrf")
            if cookie:
                version, token, timestamp = self._decode_xsrf_token(cookie)
            else:
                version, token, timestamp = None, None, None
            if token is None:
                version = None
                token = os.urandom(16)
                timestamp = time.time()
            assert token is not None
            assert timestamp is not None
            self._raw_xsrf_token = (version, token, timestamp)
        return self._raw_xsrf_token

    def _decode_xsrf_token(
        self, cookie: str
    ) -> Tuple[Optional[int], Optional[bytes], Optional[float]]:
        try:
            m = _signed_value_version_re.match(utf8(cookie))

            if m:
                version = int(m.group(1))
                if version == 2:
                    _, mask_str, masked_token, timestamp_str = cookie.split("|")

                    mask = binascii.a2b_hex(utf8(mask_str))
                    token = _websocket_mask(mask, binascii.a2b_hex(utf8(masked_token)))
                    timestamp = int(timestamp_str)
                    return version, token, timestamp
                else:
                    # Treat unknown versions as not present instead of failing.
                    raise Exception("Unknown xsrf cookie version")
            else:
                version = 1
                try:
                    token = binascii.a2b_hex(utf8(cookie))
                except (binascii.Error, TypeError):
                    token = utf8(cookie)
                # We don't have a usable timestamp in older versions.
                timestamp = int(time.time())
                return (version, token, timestamp)
        except Exception:
            # Catch exceptions and return nothing instead of failing.
            gen_log.debug("Uncaught exception in _decode_xsrf_token", exc_info=True)
            return None, None, None

    # 验证xsrf正确性
    def check_xsrf_cookie(self) -> None:
        token = (
            self.get_argument("_xsrf", None)
            or self.request.headers.get("X-Xsrftoken")
            or self.request.headers.get("X-Csrftoken")
        )
        if not token:
            raise HTTPError(403, "'_xsrf' argument missing from POST")
        _, token, _ = self._decode_xsrf_token(token)
        _, expected_token, _ = self._get_raw_xsrf_token()
        if not token:
            raise HTTPError(403, "'_xsrf' argument has invalid format")
        if not hmac.compare_digest(utf8(token), utf8(expected_token)):
            raise HTTPError(403, "XSRF cookie does not match POST argument")

    def xsrf_form_html(self) -> str:
        return (
            '<input type="hidden" name="_xsrf" value="'
            + escape.xhtml_escape(self.xsrf_token)
            + '"/>'
        )

    # 静态路径
    def static_url(self, path: str, include_host: bool = None, **kwargs: Any) -> str:
        self.require_setting("static_path", "static_url")
        get_url = self.settings.get(
            "static_handler_class", StaticFileHandler
        ).make_static_url

        if include_host is None:
            include_host = getattr(self, "include_host", False)

        if include_host:
            base = self.request.protocol + "://" + self.request.host
        else:
            base = ""

        return base + get_url(self.settings, path, **kwargs)

    # 检查是否配置了名称
    def require_setting(self, name: str, feature: str = "this feature") -> None:
        if not self.application.settings.get(name):
            raise Exception(
                "You must define the '%s' setting in your "
                "application to use %s" % (name, feature)
            )

    # 反向url
    def reverse_url(self, name: str, *args: Any) -> str:
        return self.application.reverse_url(name, *args)

    # 解析此请求的 etag 标头
    def compute_etag(self) -> Optional[str]:
        hasher = hashlib.sha1()
        for part in self._write_buffer:
            hasher.update(part)
        return '"%s"' % hasher.hexdigest()

    def set_etag_header(self) -> None:
        """Sets the response's Etag header using ``self.compute_etag()``.

        Note: no header will be set if ``compute_etag()`` returns ``None``.

        This method is called automatically when the request is finished.
        """
        etag = self.compute_etag()
        if etag is not None:
            self.set_header("Etag", etag)

    def check_etag_header(self) -> bool:
        """Checks the ``Etag`` header against requests's ``If-None-Match``.

        Returns ``True`` if the request's Etag matches and a 304 should be
        returned. For example::

            self.set_etag_header()
            if self.check_etag_header():
                self.set_status(304)
                return

        This method is called automatically when the request is finished,
        but may be called earlier for applications that override
        `compute_etag` and want to do an early check for ``If-None-Match``
        before completing the request.  The ``Etag`` header should be set
        (perhaps with `set_etag_header`) before calling this method.
        """
        computed_etag = utf8(self._headers.get("Etag", ""))
        # Find all weak and strong etag values from If-None-Match header
        # because RFC 7232 allows multiple etag values in a single header.
        etags = re.findall(
            br'\*|(?:W/)?"[^"]*"', utf8(self.request.headers.get("If-None-Match", ""))
        )
        if not computed_etag or not etags:
            return False

        match = False
        if etags[0] == b"*":
            match = True
        else:
            # Use a weak comparison when comparing entity-tags.
            def val(x: bytes) -> bytes:
                return x[2:] if x.startswith(b"W/") else x

            for etag in etags:
                if val(etag) == val(computed_etag):
                    match = True
                    break
        return match

    async def _execute(
        self, transforms: List["OutputTransform"], *args: bytes, **kwargs: bytes
    ) -> None:
        """Executes this request with the given output transforms."""
        self._transforms = transforms
        try:
            if self.request.method not in self.SUPPORTED_METHODS:
                raise HTTPError(405)
            self.path_args = [self.decode_argument(arg) for arg in args]
            self.path_kwargs = dict(
                (k, self.decode_argument(v, name=k)) for (k, v) in kwargs.items()
            )
            # If XSRF cookies are turned on, reject form submissions without
            # the proper cookie
            if self.request.method not in (
                "GET",
                "HEAD",
                "OPTIONS",
            ) and self.application.settings.get("xsrf_cookies"):
                self.check_xsrf_cookie()

            result = self.prepare()
            if result is not None:
                result = await result
            if self._prepared_future is not None:
                # Tell the Application we've finished with prepare()
                # and are ready for the body to arrive.
                future_set_result_unless_cancelled(self._prepared_future, None)
            if self._finished:
                return

            if _has_stream_request_body(self.__class__):
                # In streaming mode request.body is a Future that signals
                # the body has been completely received.  The Future has no
                # result; the data has been passed to self.data_received
                # instead.
                try:
                    await self.request._body_future
                except iostream.StreamClosedError:
                    return

            method = getattr(self, self.request.method.lower())
            result = method(*self.path_args, **self.path_kwargs)
            if result is not None:
                result = await result
            if self._auto_finish and not self._finished:
                self.finish()
        except Exception as e:
            try:
                self._handle_request_exception(e)
            except Exception:
                app_log.error("Exception in exception handler", exc_info=True)
            finally:
                # Unset result to avoid circular references
                result = None
            if self._prepared_future is not None and not self._prepared_future.done():
                # In case we failed before setting _prepared_future, do it
                # now (to unblock the HTTP server).  Note that this is not
                # in a finally block to avoid GC issues prior to Python 3.4.
                self._prepared_future.set_result(None)

    def data_received(self, chunk: bytes) -> Optional[Awaitable[None]]:
        """Implement this method to handle streamed request data.

        Requires the `.stream_request_body` decorator.

        May be a coroutine for flow control.
        """
        raise NotImplementedError()

    def _log(self) -> None:
        """Logs the current request.

        Sort of deprecated since this functionality was moved to the
        Application, but left in place for the benefit of existing apps
        that have overridden this method.
        """
        self.application.log_request(self)

    def _request_summary(self) -> str:
        return "%s %s (%s)" % (
            self.request.method,
            self.request.uri,
            self.request.remote_ip,
        )

    def _handle_request_exception(self, e: BaseException) -> None:
        if isinstance(e, Finish):
            # Not an error; just finish the request without logging.
            if not self._finished:
                self.finish(*e.args)
            return
        try:
            self.log_exception(*sys.exc_info())
        except Exception:
            # An error here should still get a best-effort send_error()
            # to avoid leaking the connection.
            app_log.error("Error in exception logger", exc_info=True)
        if self._finished:
            # Extra errors after the request has been finished should
            # be logged, but there is no reason to continue to try and
            # send a response.
            return
        if isinstance(e, HTTPError):
            self.send_error(e.status_code, exc_info=sys.exc_info())
        else:
            self.send_error(500, exc_info=sys.exc_info())

    def log_exception(
        self,
        typ: "Optional[Type[BaseException]]",
        value: Optional[BaseException],
        tb: Optional[TracebackType],
    ) -> None:
        """Override to customize logging of uncaught exceptions.

        By default logs instances of `HTTPError` as warnings without
        stack traces (on the ``tornado.general`` logger), and all
        other exceptions as errors with stack traces (on the
        ``tornado.application`` logger).

        .. versionadded:: 3.1
        """
        if isinstance(value, HTTPError):
            if value.log_message:
                format = "%d %s: " + value.log_message
                args = [value.status_code, self._request_summary()] + list(value.args)
                gen_log.warning(format, *args)
        else:
            app_log.error(  # type: ignore
                "Uncaught exception %s\n%r",
                self._request_summary(),
                self.request,
                exc_info=(typ, value, tb),
            )

    def _ui_module(self, name: str, module: Type["UIModule"]) -> Callable[..., str]:
        def render(*args, **kwargs) -> str:  # type: ignore
            if not hasattr(self, "_active_modules"):
                self._active_modules = {}  # type: Dict[str, UIModule]
            if name not in self._active_modules:
                self._active_modules[name] = module(self)
            rendered = self._active_modules[name].render(*args, **kwargs)
            return rendered

        return render

    def _ui_method(self, method: Callable[..., str]) -> Callable[..., str]:
        return lambda *args, **kwargs: method(self, *args, **kwargs)

    def _clear_headers_for_304(self) -> None:
        # 304 responses should not contain entity headers (defined in
        # http://www.w3.org/Protocols/rfc2616/rfc2616-sec7.html#sec7.1)
        # not explicitly allowed by
        # http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.3.5
        headers = [
            "Allow",
            "Content-Encoding",
            "Content-Language",
            "Content-Length",
            "Content-MD5",
            "Content-Range",
            "Content-Type",
            "Last-Modified",
        ]
        for h in headers:
            self.clear_header(h)


def stream_request_body(cls: Type[RequestHandler]) -> Type[RequestHandler]:
    """Apply to `RequestHandler` subclasses to enable streaming body support.

    This decorator implies the following changes:

    * `.HTTPServerRequest.body` is undefined, and body arguments will not
      be included in `RequestHandler.get_argument`.
    * `RequestHandler.prepare` is called when the request headers have been
      read instead of after the entire body has been read.
    * The subclass must define a method ``data_received(self, data):``, which
      will be called zero or more times as data is available.  Note that
      if the request has an empty body, ``data_received`` may not be called.
    * ``prepare`` and ``data_received`` may return Futures (such as via
      ``@gen.coroutine``, in which case the next method will not be called
      until those futures have completed.
    * The regular HTTP method (``post``, ``put``, etc) will be called after
      the entire body has been read.

    See the `file receiver demo <https://github.com/tornadoweb/tornado/tree/master/demos/file_upload/>`_
    for example usage.
    """  # noqa: E501
    if not issubclass(cls, RequestHandler):
        raise TypeError("expected subclass of RequestHandler, got %r", cls)
    cls._stream_request_body = True
    return cls


def _has_stream_request_body(cls: Type[RequestHandler]) -> bool:
    if not issubclass(cls, RequestHandler):
        raise TypeError("expected subclass of RequestHandler, got %r", cls)
    return cls._stream_request_body


def removeslash(
    method: Callable[..., Optional[Awaitable[None]]]
) -> Callable[..., Optional[Awaitable[None]]]:
    """Use this decorator to remove trailing slashes from the request path.

    For example, a request to ``/foo/`` would redirect to ``/foo`` with this
    decorator. Your request handler mapping should use a regular expression
    like ``r'/foo/*'`` in conjunction with using the decorator.
    """

    @functools.wraps(method)
    def wrapper(  # type: ignore
        self: RequestHandler, *args, **kwargs
    ) -> Optional[Awaitable[None]]:
        if self.request.path.endswith("/"):
            if self.request.method in ("GET", "HEAD"):
                uri = self.request.path.rstrip("/")
                if uri:  # don't try to redirect '/' to ''
                    if self.request.query:
                        uri += "?" + self.request.query
                    self.redirect(uri, permanent=True)
                    return None
            else:
                raise HTTPError(404)
        return method(self, *args, **kwargs)

    return wrapper


def addslash(
    method: Callable[..., Optional[Awaitable[None]]]
) -> Callable[..., Optional[Awaitable[None]]]:
    """Use this decorator to add a missing trailing slash to the request path.

    For example, a request to ``/foo`` would redirect to ``/foo/`` with this
    decorator. Your request handler mapping should use a regular expression
    like ``r'/foo/?'`` in conjunction with using the decorator.
    """

    @functools.wraps(method)
    def wrapper(  # type: ignore
        self: RequestHandler, *args, **kwargs
    ) -> Optional[Awaitable[None]]:
        if not self.request.path.endswith("/"):
            if self.request.method in ("GET", "HEAD"):
                uri = self.request.path + "/"
                if self.request.query:
                    uri += "?" + self.request.query
                self.redirect(uri, permanent=True)
                return None
            raise HTTPError(404)
        return method(self, *args, **kwargs)

    return wrapper

## class _ApplicationRouter
## 应用程序内部使用的规则路由实现
class _ApplicationRouter(ReversibleRuleRouter):
    def __init__(self, application: "Application", rules: _RuleList = None) -> None:
        assert isinstance(application, Application)
        self.application = application
        # 初始化，将规则添加进路由中
        super(_ApplicationRouter, self).__init__(rules)

    # 解析路由
    def process_rule(self, rule: Rule) -> Rule:
        rule = super(_ApplicationRouter, self).process_rule(rule)
        # 如果是嵌套路由，则继续解析
        if isinstance(rule.target, (list, tuple)):
            rule.target = _ApplicationRouter(  # type: ignore
                self.application, rule.target
            )

        return rule

    def get_target_delegate(
        self, target: Any, request: httputil.HTTPServerRequest, **target_params: Any
    ) -> Optional[httputil.HTTPMessageDelegate]:
        # 如果对象是一个 RequestHandler 的子类，交给 application 解析
        if isclass(target) and issubclass(target, RequestHandler):
            return self.application.get_handler_delegate(
                request, target, **target_params
            )
        # 对象无法识别，交给父类处理
        return super(_ApplicationRouter, self).get_target_delegate(
            target, request, **target_params
        )

## class Application
## web 请求处理程序的集合
class Application(ReversibleRouter):
    def __init__(
        self,
        handlers: _RuleList = None,
        default_host: str = None,
        transforms: List[Type["OutputTransform"]] = None,
        **settings: Any
    ) -> None:
        # 根据配置设置传输压缩方法
        if transforms is None:
            self.transforms = []  # type: List[Type[OutputTransform]]
            if settings.get("compress_response") or settings.get("gzip"):
                self.transforms.append(GZipContentEncoding)
        else:
            self.transforms = transforms
        # 默认的主机
        self.default_host = default_host
        # 传入的配置
        self.settings = settings
        # 网页模块
        self.ui_modules = {
            "linkify": _linkify,
            "xsrf_form_html": _xsrf_form_html,
            "Template": TemplateModule,
        }
        # 导入配置文件中的 UI 模块和方法
        self.ui_methods = {}  # type: Dict[str, Callable[..., str]]
        self._load_ui_modules(settings.get("ui_modules", {}))
        self._load_ui_methods(settings.get("ui_methods", {}))
        # 配置静态文件路径
        if self.settings.get("static_path"):
            path = self.settings["static_path"]
            handlers = list(handlers or [])
            static_url_prefix = settings.get("static_url_prefix", "/static/")
            static_handler_class = settings.get(
                "static_handler_class", StaticFileHandler
            )
            static_handler_args = settings.get("static_handler_args", {})
            static_handler_args["path"] = path
            for pattern in [
                re.escape(static_url_prefix) + r"(.*)",
                r"/(favicon\.ico)",
                r"/(robots\.txt)",
            ]:
                handlers.insert(0, (pattern, static_handler_class, static_handler_args))
        # 调试模式设置的参数
        if self.settings.get("debug"):
            # 自动重载
            self.settings.setdefault("autoreload", True)
            # 取消模板缓存
            self.settings.setdefault("compiled_template_cache", False)
            # 取消静态哈希
            self.settings.setdefault("static_hash_cache", False)
            # 打开回溯
            self.settings.setdefault("serve_traceback", True)
        # 设置 url 设置路由
        self.wildcard_router = _ApplicationRouter(self, handlers)
        # 默认路由
        self.default_router = _ApplicationRouter(
            self, [Rule(AnyMatches(), self.wildcard_router)]
        )

        # 自动重载
        if self.settings.get("autoreload"):
            from tornado import autoreload

            autoreload.start()

    # 创建一个httpserver监听端口
    # 多进程等模式请自己创建server并启动它
    def listen(self, port: int, address: str = "", **kwargs: Any) -> HTTPServer:
        # 创建服务器实例
        server = HTTPServer(self, **kwargs)
        # 监听端口
        server.listen(port, address)
        return server

    # 添加一个处理程序
    def add_handlers(self, host_pattern: str, host_handlers: _RuleList) -> None:
        host_matcher = HostMatches(host_pattern)
        rule = Rule(host_matcher, _ApplicationRouter(self, host_handlers))

        self.default_router.rules.insert(-1, rule)

        if self.default_host is not None:
            self.wildcard_router.add_rules(
                [(DefaultHostMatches(self, host_matcher.host_pattern), host_handlers)]
            )

    # 添加一个请求返回处理
    def add_transform(self, transform_class: Type["OutputTransform"]) -> None:
        self.transforms.append(transform_class)

    def _load_ui_methods(self, methods: Any) -> None:
        if isinstance(methods, types.ModuleType):
            self._load_ui_methods(dict((n, getattr(methods, n)) for n in dir(methods)))
        elif isinstance(methods, list):
            for m in methods:
                self._load_ui_methods(m)
        else:
            for name, fn in methods.items():
                if (
                    not name.startswith("_")
                    and hasattr(fn, "__call__")
                    and name[0].lower() == name[0]
                ):
                    self.ui_methods[name] = fn

    def _load_ui_modules(self, modules: Any) -> None:
        if isinstance(modules, types.ModuleType):
            self._load_ui_modules(dict((n, getattr(modules, n)) for n in dir(modules)))
        elif isinstance(modules, list):
            for m in modules:
                self._load_ui_modules(m)
        else:
            assert isinstance(modules, dict)
            for name, cls in modules.items():
                try:
                    if issubclass(cls, UIModule):
                        self.ui_modules[name] = cls
                except TypeError:
                    pass

    def __call__(
        self, request: httputil.HTTPServerRequest
    ) -> Optional[Awaitable[None]]:
        # Legacy HTTPServer interface
        dispatcher = self.find_handler(request)
        return dispatcher.execute()

    def find_handler(
        self, request: httputil.HTTPServerRequest, **kwargs: Any
    ) -> "_HandlerDelegate":
        route = self.default_router.find_handler(request)
        if route is not None:
            return cast("_HandlerDelegate", route)

        if self.settings.get("default_handler_class"):
            return self.get_handler_delegate(
                request,
                self.settings["default_handler_class"],
                self.settings.get("default_handler_args", {}),
            )
        # 默认的处理程序，返回404
        return self.get_handler_delegate(request, ErrorHandler, {"status_code": 404})

    def get_handler_delegate(
        self,
        request: httputil.HTTPServerRequest,
        target_class: Type[RequestHandler],
        target_kwargs: Dict[str, Any] = None,
        path_args: List[bytes] = None,
        path_kwargs: Dict[str, bytes] = None,
    ) -> "_HandlerDelegate":
        return _HandlerDelegate(
            self, request, target_class, target_kwargs, path_args, path_kwargs
        )

    # 逆转url，盲猜重定向
    def reverse_url(self, name: str, *args: Any) -> str:
        reversed_url = self.default_router.reverse_url(name, *args)
        if reversed_url is not None:
            return reversed_url

        raise KeyError("%s not found in named urls" % name)

    # 请求日志记录函数
    def log_request(self, handler: RequestHandler) -> None:
        if "log_function" in self.settings:
            self.settings["log_function"](handler)
            return
        if handler.get_status() < 400:
            log_method = access_log.info
        elif handler.get_status() < 500:
            log_method = access_log.warning
        else:
            log_method = access_log.error
        request_time = 1000.0 * handler.request.request_time()
        log_method(
            "%d %s %.2fms",
            handler.get_status(),
            handler._request_summary(),
            request_time,
        )

## class _HandlerDelegate
class _HandlerDelegate(httputil.HTTPMessageDelegate):
    # 处理程序的代理
    def __init__(
        self,
        application: Application,
        request: httputil.HTTPServerRequest,
        handler_class: Type[RequestHandler],
        handler_kwargs: Optional[Dict[str, Any]],
        path_args: Optional[List[bytes]],
        path_kwargs: Optional[Dict[str, bytes]],
    ) -> None:
        # 应用程序
        self.application = application
        # 请求的连接
        self.connection = request.connection
        # 请求对象
        self.request = request
        # 匹配的处理程序类对象
        self.handler_class = handler_class
        # 传递的参数
        self.handler_kwargs = handler_kwargs or {}
        # 路径参数
        self.path_args = path_args or []
        # 路径字典参数
        self.path_kwargs = path_kwargs or {}
        # 缓存
        self.chunks = []  # type: List[bytes]
        # 客户端中途断开连接是否需要做处理
        self.stream_request_body = _has_stream_request_body(self.handler_class)

    # 收到请求头
    def headers_received(
        self,
        start_line: Union[httputil.RequestStartLine, httputil.ResponseStartLine],
        headers: httputil.HTTPHeaders,
    ) -> Optional[Awaitable[None]]:
        if self.stream_request_body:
            self.request._body_future = Future()
            return self.execute()
        return None

    # 收到请求体
    def data_received(self, data: bytes) -> Optional[Awaitable[None]]:
        if self.stream_request_body:
            return self.handler.data_received(data)
        else:
            self.chunks.append(data)
            return None

    # 请求处理完成
    def finish(self) -> None:
        if self.stream_request_body:
            future_set_result_unless_cancelled(self.request._body_future, None)
        else:
            self.request.body = b"".join(self.chunks)
            self.request._parse_body()
            self.execute()

    # 连接关闭，判断是否需要处理
    def on_connection_close(self) -> None:
        if self.stream_request_body:
            self.handler.on_connection_close()
        else:
            self.chunks = None  # type: ignore

    # 执行
    def execute(self) -> Optional[Awaitable[None]]:
        # 调试模式，重新加载模板
        if not self.application.settings.get("compiled_template_cache", True):
            with RequestHandler._template_loader_lock:
                for loader in RequestHandler._template_loaders.values():
                    loader.reset()
        if not self.application.settings.get("static_hash_cache", True):
            StaticFileHandler.reset()

        # 请求处理类对象 `RequestHandle`
        self.handler = self.handler_class(
            self.application, self.request, **self.handler_kwargs
        )
        # 对请求进行预处理
        transforms = [t(self.request) for t in self.application.transforms]

        if self.stream_request_body:
            self.handler._prepared_future = Future()
        # Note that if an exception escapes handler._execute it will be
        # trapped in the Future it returns (which we are ignoring here,
        # leaving it to be logged when the Future is GC'd).
        # However, that shouldn't happen because _execute has a blanket
        # except handler, and we cannot easily access the IOLoop here to
        # call add_future (because of the requirement to remain compatible
        # with WSGI)
        fut = gen.convert_yielded(
            self.handler._execute(transforms, *self.path_args, **self.path_kwargs)
        )
        fut.add_done_callback(lambda f: f.result())
        # If we are streaming the request body, then execute() is finished
        # when the handler has prepared to receive the body.  If not,
        # it doesn't matter when execute() finishes (so we return None)
        return self.handler._prepared_future


class HTTPError(Exception):
    """An exception that will turn into an HTTP error response.

    Raising an `HTTPError` is a convenient alternative to calling
    `RequestHandler.send_error` since it automatically ends the
    current function.

    To customize the response sent with an `HTTPError`, override
    `RequestHandler.write_error`.

    :arg int status_code: HTTP status code.  Must be listed in
        `httplib.responses <http.client.responses>` unless the ``reason``
        keyword argument is given.
    :arg str log_message: Message to be written to the log for this error
        (will not be shown to the user unless the `Application` is in debug
        mode).  May contain ``%s``-style placeholders, which will be filled
        in with remaining positional parameters.
    :arg str reason: Keyword-only argument.  The HTTP "reason" phrase
        to pass in the status line along with ``status_code``.  Normally
        determined automatically from ``status_code``, but can be used
        to use a non-standard numeric code.
    """

    def __init__(
        self, status_code: int = 500, log_message: str = None, *args: Any, **kwargs: Any
    ) -> None:
        self.status_code = status_code
        self.log_message = log_message
        self.args = args
        self.reason = kwargs.get("reason", None)
        if log_message and not args:
            self.log_message = log_message.replace("%", "%%")

    def __str__(self) -> str:
        message = "HTTP %d: %s" % (
            self.status_code,
            self.reason or httputil.responses.get(self.status_code, "Unknown"),
        )
        if self.log_message:
            return message + " (" + (self.log_message % self.args) + ")"
        else:
            return message


class Finish(Exception):
    """An exception that ends the request without producing an error response.

    When `Finish` is raised in a `RequestHandler`, the request will
    end (calling `RequestHandler.finish` if it hasn't already been
    called), but the error-handling methods (including
    `RequestHandler.write_error`) will not be called.

    If `Finish()` was created with no arguments, the pending response
    will be sent as-is. If `Finish()` was given an argument, that
    argument will be passed to `RequestHandler.finish()`.

    This can be a more convenient way to implement custom error pages
    than overriding ``write_error`` (especially in library code)::

        if self.current_user is None:
            self.set_status(401)
            self.set_header('WWW-Authenticate', 'Basic realm="something"')
            raise Finish()

    .. versionchanged:: 4.3
       Arguments passed to ``Finish()`` will be passed on to
       `RequestHandler.finish`.
    """

    pass


class MissingArgumentError(HTTPError):
    """Exception raised by `RequestHandler.get_argument`.

    This is a subclass of `HTTPError`, so if it is uncaught a 400 response
    code will be used instead of 500 (and a stack trace will not be logged).

    .. versionadded:: 3.1
    """

    def __init__(self, arg_name: str) -> None:
        super(MissingArgumentError, self).__init__(
            400, "Missing argument %s" % arg_name
        )
        self.arg_name = arg_name


class ErrorHandler(RequestHandler):
    """Generates an error response with ``status_code`` for all requests."""

    def initialize(self, status_code: int) -> None:
        self.set_status(status_code)

    def prepare(self) -> None:
        raise HTTPError(self._status_code)

    def check_xsrf_cookie(self) -> None:
        # POSTs to an ErrorHandler don't actually have side effects,
        # so we don't need to check the xsrf token.  This allows POSTs
        # to the wrong url to return a 404 instead of 403.
        pass


class RedirectHandler(RequestHandler):
    """Redirects the client to the given URL for all GET requests.

    You should provide the keyword argument ``url`` to the handler, e.g.::

        application = web.Application([
            (r"/oldpath", web.RedirectHandler, {"url": "/newpath"}),
        ])

    `RedirectHandler` supports regular expression substitutions. E.g., to
    swap the first and second parts of a path while preserving the remainder::

        application = web.Application([
            (r"/(.*?)/(.*?)/(.*)", web.RedirectHandler, {"url": "/{1}/{0}/{2}"}),
        ])

    The final URL is formatted with `str.format` and the substrings that match
    the capturing groups. In the above example, a request to "/a/b/c" would be
    formatted like::

        str.format("/{1}/{0}/{2}", "a", "b", "c")  # -> "/b/a/c"

    Use Python's :ref:`format string syntax <formatstrings>` to customize how
    values are substituted.

    .. versionchanged:: 4.5
       Added support for substitutions into the destination URL.

    .. versionchanged:: 5.0
       If any query arguments are present, they will be copied to the
       destination URL.
    """

    def initialize(self, url: str, permanent: bool = True) -> None:
        self._url = url
        self._permanent = permanent

    def get(self, *args: Any) -> None:
        to_url = self._url.format(*args)
        if self.request.query_arguments:
            # TODO: figure out typing for the next line.
            to_url = httputil.url_concat(
                to_url,
                list(httputil.qs_to_qsl(self.request.query_arguments)),  # type: ignore
            )
        self.redirect(to_url, permanent=self._permanent)


class StaticFileHandler(RequestHandler):
    """A simple handler that can serve static content from a directory.

    A `StaticFileHandler` is configured automatically if you pass the
    ``static_path`` keyword argument to `Application`.  This handler
    can be customized with the ``static_url_prefix``, ``static_handler_class``,
    and ``static_handler_args`` settings.

    To map an additional path to this handler for a static data directory
    you would add a line to your application like::

        application = web.Application([
            (r"/content/(.*)", web.StaticFileHandler, {"path": "/var/www"}),
        ])

    The handler constructor requires a ``path`` argument, which specifies the
    local root directory of the content to be served.

    Note that a capture group in the regex is required to parse the value for
    the ``path`` argument to the get() method (different than the constructor
    argument above); see `URLSpec` for details.

    To serve a file like ``index.html`` automatically when a directory is
    requested, set ``static_handler_args=dict(default_filename="index.html")``
    in your application settings, or add ``default_filename`` as an initializer
    argument for your ``StaticFileHandler``.

    To maximize the effectiveness of browser caching, this class supports
    versioned urls (by default using the argument ``?v=``).  If a version
    is given, we instruct the browser to cache this file indefinitely.
    `make_static_url` (also available as `RequestHandler.static_url`) can
    be used to construct a versioned url.

    This handler is intended primarily for use in development and light-duty
    file serving; for heavy traffic it will be more efficient to use
    a dedicated static file server (such as nginx or Apache).  We support
    the HTTP ``Accept-Ranges`` mechanism to return partial content (because
    some browsers require this functionality to be present to seek in
    HTML5 audio or video).

    **Subclassing notes**

    This class is designed to be extensible by subclassing, but because
    of the way static urls are generated with class methods rather than
    instance methods, the inheritance patterns are somewhat unusual.
    Be sure to use the ``@classmethod`` decorator when overriding a
    class method.  Instance methods may use the attributes ``self.path``
    ``self.absolute_path``, and ``self.modified``.

    Subclasses should only override methods discussed in this section;
    overriding other methods is error-prone.  Overriding
    ``StaticFileHandler.get`` is particularly problematic due to the
    tight coupling with ``compute_etag`` and other methods.

    To change the way static urls are generated (e.g. to match the behavior
    of another server or CDN), override `make_static_url`, `parse_url_path`,
    `get_cache_time`, and/or `get_version`.

    To replace all interaction with the filesystem (e.g. to serve
    static content from a database), override `get_content`,
    `get_content_size`, `get_modified_time`, `get_absolute_path`, and
    `validate_absolute_path`.

    .. versionchanged:: 3.1
       Many of the methods for subclasses were added in Tornado 3.1.
    """

    CACHE_MAX_AGE = 86400 * 365 * 10  # 10 years

    _static_hashes = {}  # type: Dict[str, Optional[str]]
    _lock = threading.Lock()  # protects _static_hashes

    def initialize(self, path: str, default_filename: str = None) -> None:
        self.root = path
        self.default_filename = default_filename

    @classmethod
    def reset(cls) -> None:
        with cls._lock:
            cls._static_hashes = {}

    def head(self, path: str) -> Awaitable[None]:
        return self.get(path, include_body=False)

    async def get(self, path: str, include_body: bool = True) -> None:
        # Set up our path instance variables.
        self.path = self.parse_url_path(path)
        del path  # make sure we don't refer to path instead of self.path again
        absolute_path = self.get_absolute_path(self.root, self.path)
        self.absolute_path = self.validate_absolute_path(self.root, absolute_path)
        if self.absolute_path is None:
            return

        self.modified = self.get_modified_time()
        self.set_headers()

        if self.should_return_304():
            self.set_status(304)
            return

        request_range = None
        range_header = self.request.headers.get("Range")
        if range_header:
            # As per RFC 2616 14.16, if an invalid Range header is specified,
            # the request will be treated as if the header didn't exist.
            request_range = httputil._parse_request_range(range_header)

        size = self.get_content_size()
        if request_range:
            start, end = request_range
            if start is not None and start < 0:
                start += size
                if start < 0:
                    start = 0
            if (
                start is not None
                and (start >= size or (end is not None and start >= end))
            ) or end == 0:
                # As per RFC 2616 14.35.1, a range is not satisfiable only: if
                # the first requested byte is equal to or greater than the
                # content, or when a suffix with length 0 is specified.
                # https://tools.ietf.org/html/rfc7233#section-2.1
                # A byte-range-spec is invalid if the last-byte-pos value is present
                # and less than the first-byte-pos.
                self.set_status(416)  # Range Not Satisfiable
                self.set_header("Content-Type", "text/plain")
                self.set_header("Content-Range", "bytes */%s" % (size,))
                return
            if end is not None and end > size:
                # Clients sometimes blindly use a large range to limit their
                # download size; cap the endpoint at the actual file size.
                end = size
            # Note: only return HTTP 206 if less than the entire range has been
            # requested. Not only is this semantically correct, but Chrome
            # refuses to play audio if it gets an HTTP 206 in response to
            # ``Range: bytes=0-``.
            if size != (end or size) - (start or 0):
                self.set_status(206)  # Partial Content
                self.set_header(
                    "Content-Range", httputil._get_content_range(start, end, size)
                )
        else:
            start = end = None

        if start is not None and end is not None:
            content_length = end - start
        elif end is not None:
            content_length = end
        elif start is not None:
            content_length = size - start
        else:
            content_length = size
        self.set_header("Content-Length", content_length)

        if include_body:
            content = self.get_content(self.absolute_path, start, end)
            if isinstance(content, bytes):
                content = [content]
            for chunk in content:
                try:
                    self.write(chunk)
                    await self.flush()
                except iostream.StreamClosedError:
                    return
        else:
            assert self.request.method == "HEAD"

    def compute_etag(self) -> Optional[str]:
        """Sets the ``Etag`` header based on static url version.

        This allows efficient ``If-None-Match`` checks against cached
        versions, and sends the correct ``Etag`` for a partial response
        (i.e. the same ``Etag`` as the full file).

        .. versionadded:: 3.1
        """
        assert self.absolute_path is not None
        version_hash = self._get_cached_version(self.absolute_path)
        if not version_hash:
            return None
        return '"%s"' % (version_hash,)

    def set_headers(self) -> None:
        """Sets the content and caching headers on the response.

        .. versionadded:: 3.1
        """
        self.set_header("Accept-Ranges", "bytes")
        self.set_etag_header()

        if self.modified is not None:
            self.set_header("Last-Modified", self.modified)

        content_type = self.get_content_type()
        if content_type:
            self.set_header("Content-Type", content_type)

        cache_time = self.get_cache_time(self.path, self.modified, content_type)
        if cache_time > 0:
            self.set_header(
                "Expires",
                datetime.datetime.utcnow() + datetime.timedelta(seconds=cache_time),
            )
            self.set_header("Cache-Control", "max-age=" + str(cache_time))

        self.set_extra_headers(self.path)

    def should_return_304(self) -> bool:
        """Returns True if the headers indicate that we should return 304.

        .. versionadded:: 3.1
        """
        # If client sent If-None-Match, use it, ignore If-Modified-Since
        if self.request.headers.get("If-None-Match"):
            return self.check_etag_header()

        # Check the If-Modified-Since, and don't send the result if the
        # content has not been modified
        ims_value = self.request.headers.get("If-Modified-Since")
        if ims_value is not None:
            date_tuple = email.utils.parsedate(ims_value)
            if date_tuple is not None:
                if_since = datetime.datetime(*date_tuple[:6])
                assert self.modified is not None
                if if_since >= self.modified:
                    return True

        return False

    @classmethod
    def get_absolute_path(cls, root: str, path: str) -> str:
        """Returns the absolute location of ``path`` relative to ``root``.

        ``root`` is the path configured for this `StaticFileHandler`
        (in most cases the ``static_path`` `Application` setting).

        This class method may be overridden in subclasses.  By default
        it returns a filesystem path, but other strings may be used
        as long as they are unique and understood by the subclass's
        overridden `get_content`.

        .. versionadded:: 3.1
        """
        abspath = os.path.abspath(os.path.join(root, path))
        return abspath

    def validate_absolute_path(self, root: str, absolute_path: str) -> Optional[str]:
        """Validate and return the absolute path.

        ``root`` is the configured path for the `StaticFileHandler`,
        and ``path`` is the result of `get_absolute_path`

        This is an instance method called during request processing,
        so it may raise `HTTPError` or use methods like
        `RequestHandler.redirect` (return None after redirecting to
        halt further processing).  This is where 404 errors for missing files
        are generated.

        This method may modify the path before returning it, but note that
        any such modifications will not be understood by `make_static_url`.

        In instance methods, this method's result is available as
        ``self.absolute_path``.

        .. versionadded:: 3.1
        """
        # os.path.abspath strips a trailing /.
        # We must add it back to `root` so that we only match files
        # in a directory named `root` instead of files starting with
        # that prefix.
        root = os.path.abspath(root)
        if not root.endswith(os.path.sep):
            # abspath always removes a trailing slash, except when
            # root is '/'. This is an unusual case, but several projects
            # have independently discovered this technique to disable
            # Tornado's path validation and (hopefully) do their own,
            # so we need to support it.
            root += os.path.sep
        # The trailing slash also needs to be temporarily added back
        # the requested path so a request to root/ will match.
        if not (absolute_path + os.path.sep).startswith(root):
            raise HTTPError(403, "%s is not in root static directory", self.path)
        if os.path.isdir(absolute_path) and self.default_filename is not None:
            # need to look at the request.path here for when path is empty
            # but there is some prefix to the path that was already
            # trimmed by the routing
            if not self.request.path.endswith("/"):
                self.redirect(self.request.path + "/", permanent=True)
                return None
            absolute_path = os.path.join(absolute_path, self.default_filename)
        if not os.path.exists(absolute_path):
            raise HTTPError(404)
        if not os.path.isfile(absolute_path):
            raise HTTPError(403, "%s is not a file", self.path)
        return absolute_path

    @classmethod
    def get_content(
        cls, abspath: str, start: int = None, end: int = None
    ) -> Generator[bytes, None, None]:
        """Retrieve the content of the requested resource which is located
        at the given absolute path.

        This class method may be overridden by subclasses.  Note that its
        signature is different from other overridable class methods
        (no ``settings`` argument); this is deliberate to ensure that
        ``abspath`` is able to stand on its own as a cache key.

        This method should either return a byte string or an iterator
        of byte strings.  The latter is preferred for large files
        as it helps reduce memory fragmentation.

        .. versionadded:: 3.1
        """
        with open(abspath, "rb") as file:
            if start is not None:
                file.seek(start)
            if end is not None:
                remaining = end - (start or 0)  # type: Optional[int]
            else:
                remaining = None
            while True:
                chunk_size = 64 * 1024
                if remaining is not None and remaining < chunk_size:
                    chunk_size = remaining
                chunk = file.read(chunk_size)
                if chunk:
                    if remaining is not None:
                        remaining -= len(chunk)
                    yield chunk
                else:
                    if remaining is not None:
                        assert remaining == 0
                    return

    @classmethod
    def get_content_version(cls, abspath: str) -> str:
        """Returns a version string for the resource at the given path.

        This class method may be overridden by subclasses.  The
        default implementation is a hash of the file's contents.

        .. versionadded:: 3.1
        """
        data = cls.get_content(abspath)
        hasher = hashlib.md5()
        if isinstance(data, bytes):
            hasher.update(data)
        else:
            for chunk in data:
                hasher.update(chunk)
        return hasher.hexdigest()

    def _stat(self) -> os.stat_result:
        assert self.absolute_path is not None
        if not hasattr(self, "_stat_result"):
            self._stat_result = os.stat(self.absolute_path)
        return self._stat_result

    def get_content_size(self) -> int:
        """Retrieve the total size of the resource at the given path.

        This method may be overridden by subclasses.

        .. versionadded:: 3.1

        .. versionchanged:: 4.0
           This method is now always called, instead of only when
           partial results are requested.
        """
        stat_result = self._stat()
        return stat_result.st_size

    def get_modified_time(self) -> Optional[datetime.datetime]:
        """Returns the time that ``self.absolute_path`` was last modified.

        May be overridden in subclasses.  Should return a `~datetime.datetime`
        object or None.

        .. versionadded:: 3.1
        """
        stat_result = self._stat()
        # NOTE: Historically, this used stat_result[stat.ST_MTIME],
        # which truncates the fractional portion of the timestamp. It
        # was changed from that form to stat_result.st_mtime to
        # satisfy mypy (which disallows the bracket operator), but the
        # latter form returns a float instead of an int. For
        # consistency with the past (and because we have a unit test
        # that relies on this), we truncate the float here, although
        # I'm not sure that's the right thing to do.
        modified = datetime.datetime.utcfromtimestamp(int(stat_result.st_mtime))
        return modified

    def get_content_type(self) -> str:
        """Returns the ``Content-Type`` header to be used for this request.

        .. versionadded:: 3.1
        """
        assert self.absolute_path is not None
        mime_type, encoding = mimetypes.guess_type(self.absolute_path)
        # per RFC 6713, use the appropriate type for a gzip compressed file
        if encoding == "gzip":
            return "application/gzip"
        # As of 2015-07-21 there is no bzip2 encoding defined at
        # http://www.iana.org/assignments/media-types/media-types.xhtml
        # So for that (and any other encoding), use octet-stream.
        elif encoding is not None:
            return "application/octet-stream"
        elif mime_type is not None:
            return mime_type
        # if mime_type not detected, use application/octet-stream
        else:
            return "application/octet-stream"

    def set_extra_headers(self, path: str) -> None:
        """For subclass to add extra headers to the response"""
        pass

    def get_cache_time(
        self, path: str, modified: Optional[datetime.datetime], mime_type: str
    ) -> int:
        """Override to customize cache control behavior.

        Return a positive number of seconds to make the result
        cacheable for that amount of time or 0 to mark resource as
        cacheable for an unspecified amount of time (subject to
        browser heuristics).

        By default returns cache expiry of 10 years for resources requested
        with ``v`` argument.
        """
        return self.CACHE_MAX_AGE if "v" in self.request.arguments else 0

    @classmethod
    def make_static_url(
        cls, settings: Dict[str, Any], path: str, include_version: bool = True
    ) -> str:
        """Constructs a versioned url for the given path.

        This method may be overridden in subclasses (but note that it
        is a class method rather than an instance method).  Subclasses
        are only required to implement the signature
        ``make_static_url(cls, settings, path)``; other keyword
        arguments may be passed through `~RequestHandler.static_url`
        but are not standard.

        ``settings`` is the `Application.settings` dictionary.  ``path``
        is the static path being requested.  The url returned should be
        relative to the current host.

        ``include_version`` determines whether the generated URL should
        include the query string containing the version hash of the
        file corresponding to the given ``path``.

        """
        url = settings.get("static_url_prefix", "/static/") + path
        if not include_version:
            return url

        version_hash = cls.get_version(settings, path)
        if not version_hash:
            return url

        return "%s?v=%s" % (url, version_hash)

    def parse_url_path(self, url_path: str) -> str:
        """Converts a static URL path into a filesystem path.

        ``url_path`` is the path component of the URL with
        ``static_url_prefix`` removed.  The return value should be
        filesystem path relative to ``static_path``.

        This is the inverse of `make_static_url`.
        """
        if os.path.sep != "/":
            url_path = url_path.replace("/", os.path.sep)
        return url_path

    @classmethod
    def get_version(cls, settings: Dict[str, Any], path: str) -> Optional[str]:
        """Generate the version string to be used in static URLs.

        ``settings`` is the `Application.settings` dictionary and ``path``
        is the relative location of the requested asset on the filesystem.
        The returned value should be a string, or ``None`` if no version
        could be determined.

        .. versionchanged:: 3.1
           This method was previously recommended for subclasses to override;
           `get_content_version` is now preferred as it allows the base
           class to handle caching of the result.
        """
        abs_path = cls.get_absolute_path(settings["static_path"], path)
        return cls._get_cached_version(abs_path)

    @classmethod
    def _get_cached_version(cls, abs_path: str) -> Optional[str]:
        with cls._lock:
            hashes = cls._static_hashes
            if abs_path not in hashes:
                try:
                    hashes[abs_path] = cls.get_content_version(abs_path)
                except Exception:
                    gen_log.error("Could not open static file %r", abs_path)
                    hashes[abs_path] = None
            hsh = hashes.get(abs_path)
            if hsh:
                return hsh
        return None


class FallbackHandler(RequestHandler):
    """A `RequestHandler` that wraps another HTTP server callback.

    The fallback is a callable object that accepts an
    `~.httputil.HTTPServerRequest`, such as an `Application` or
    `tornado.wsgi.WSGIContainer`.  This is most useful to use both
    Tornado ``RequestHandlers`` and WSGI in the same server.  Typical
    usage::

        wsgi_app = tornado.wsgi.WSGIContainer(
            django.core.handlers.wsgi.WSGIHandler())
        application = tornado.web.Application([
            (r"/foo", FooHandler),
            (r".*", FallbackHandler, dict(fallback=wsgi_app),
        ])
    """

    def initialize(
        self, fallback: Callable[[httputil.HTTPServerRequest], None]
    ) -> None:
        self.fallback = fallback

    def prepare(self) -> None:
        self.fallback(self.request)
        self._finished = True
        self.on_finish()


class OutputTransform(object):
    """A transform modifies the result of an HTTP request (e.g., GZip encoding)

    Applications are not expected to create their own OutputTransforms
    or interact with them directly; the framework chooses which transforms
    (if any) to apply.
    """

    def __init__(self, request: httputil.HTTPServerRequest) -> None:
        pass

    def transform_first_chunk(
        self,
        status_code: int,
        headers: httputil.HTTPHeaders,
        chunk: bytes,
        finishing: bool,
    ) -> Tuple[int, httputil.HTTPHeaders, bytes]:
        return status_code, headers, chunk

    def transform_chunk(self, chunk: bytes, finishing: bool) -> bytes:
        return chunk


class GZipContentEncoding(OutputTransform):
    """Applies the gzip content encoding to the response.

    See http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.11

    .. versionchanged:: 4.0
        Now compresses all mime types beginning with ``text/``, instead
        of just a whitelist. (the whitelist is still used for certain
        non-text mime types).
    """

    # Whitelist of compressible mime types (in addition to any types
    # beginning with "text/").
    CONTENT_TYPES = set(
        [
            "application/javascript",
            "application/x-javascript",
            "application/xml",
            "application/atom+xml",
            "application/json",
            "application/xhtml+xml",
            "image/svg+xml",
        ]
    )
    # Python's GzipFile defaults to level 9, while most other gzip
    # tools (including gzip itself) default to 6, which is probably a
    # better CPU/size tradeoff.
    GZIP_LEVEL = 6
    # Responses that are too short are unlikely to benefit from gzipping
    # after considering the "Content-Encoding: gzip" header and the header
    # inside the gzip encoding.
    # Note that responses written in multiple chunks will be compressed
    # regardless of size.
    MIN_LENGTH = 1024

    def __init__(self, request: httputil.HTTPServerRequest) -> None:
        self._gzipping = "gzip" in request.headers.get("Accept-Encoding", "")

    def _compressible_type(self, ctype: str) -> bool:
        return ctype.startswith("text/") or ctype in self.CONTENT_TYPES

    def transform_first_chunk(
        self,
        status_code: int,
        headers: httputil.HTTPHeaders,
        chunk: bytes,
        finishing: bool,
    ) -> Tuple[int, httputil.HTTPHeaders, bytes]:
        # TODO: can/should this type be inherited from the superclass?
        if "Vary" in headers:
            headers["Vary"] += ", Accept-Encoding"
        else:
            headers["Vary"] = "Accept-Encoding"
        if self._gzipping:
            ctype = _unicode(headers.get("Content-Type", "")).split(";")[0]
            self._gzipping = (
                self._compressible_type(ctype)
                and (not finishing or len(chunk) >= self.MIN_LENGTH)
                and ("Content-Encoding" not in headers)
            )
        if self._gzipping:
            headers["Content-Encoding"] = "gzip"
            self._gzip_value = BytesIO()
            self._gzip_file = gzip.GzipFile(
                mode="w", fileobj=self._gzip_value, compresslevel=self.GZIP_LEVEL
            )
            chunk = self.transform_chunk(chunk, finishing)
            if "Content-Length" in headers:
                # The original content length is no longer correct.
                # If this is the last (and only) chunk, we can set the new
                # content-length; otherwise we remove it and fall back to
                # chunked encoding.
                if finishing:
                    headers["Content-Length"] = str(len(chunk))
                else:
                    del headers["Content-Length"]
        return status_code, headers, chunk

    def transform_chunk(self, chunk: bytes, finishing: bool) -> bytes:
        if self._gzipping:
            self._gzip_file.write(chunk)
            if finishing:
                self._gzip_file.close()
            else:
                self._gzip_file.flush()
            chunk = self._gzip_value.getvalue()
            self._gzip_value.truncate(0)
            self._gzip_value.seek(0)
        return chunk


def authenticated(
    method: Callable[..., Optional[Awaitable[None]]]
) -> Callable[..., Optional[Awaitable[None]]]:
    """Decorate methods with this to require that the user be logged in.

    If the user is not logged in, they will be redirected to the configured
    `login url <RequestHandler.get_login_url>`.

    If you configure a login url with a query parameter, Tornado will
    assume you know what you're doing and use it as-is.  If not, it
    will add a `next` parameter so the login page knows where to send
    you once you're logged in.
    """

    @functools.wraps(method)
    def wrapper(  # type: ignore
        self: RequestHandler, *args, **kwargs
    ) -> Optional[Awaitable[None]]:
        if not self.current_user:
            if self.request.method in ("GET", "HEAD"):
                url = self.get_login_url()
                if "?" not in url:
                    if urllib.parse.urlsplit(url).scheme:
                        # if login url is absolute, make next absolute too
                        next_url = self.request.full_url()
                    else:
                        assert self.request.uri is not None
                        next_url = self.request.uri
                    url += "?" + urlencode(dict(next=next_url))
                self.redirect(url)
                return None
            raise HTTPError(403)
        return method(self, *args, **kwargs)

    return wrapper


class UIModule(object):
    """A re-usable, modular UI unit on a page.

    UI modules often execute additional queries, and they can include
    additional CSS and JavaScript that will be included in the output
    page, which is automatically inserted on page render.

    Subclasses of UIModule must override the `render` method.
    """

    def __init__(self, handler: RequestHandler) -> None:
        self.handler = handler
        self.request = handler.request
        self.ui = handler.ui
        self.locale = handler.locale

    @property
    def current_user(self) -> Any:
        return self.handler.current_user

    def render(self, *args: Any, **kwargs: Any) -> str:
        """Override in subclasses to return this module's output."""
        raise NotImplementedError()

    def embedded_javascript(self) -> Optional[str]:
        """Override to return a JavaScript string
        to be embedded in the page."""
        return None

    def javascript_files(self) -> Optional[Iterable[str]]:
        """Override to return a list of JavaScript files needed by this module.

        If the return values are relative paths, they will be passed to
        `RequestHandler.static_url`; otherwise they will be used as-is.
        """
        return None

    def embedded_css(self) -> Optional[str]:
        """Override to return a CSS string
        that will be embedded in the page."""
        return None

    def css_files(self) -> Optional[Iterable[str]]:
        """Override to returns a list of CSS files required by this module.

        If the return values are relative paths, they will be passed to
        `RequestHandler.static_url`; otherwise they will be used as-is.
        """
        return None

    def html_head(self) -> Optional[str]:
        """Override to return an HTML string that will be put in the <head/>
        element.
        """
        return None

    def html_body(self) -> Optional[str]:
        """Override to return an HTML string that will be put at the end of
        the <body/> element.
        """
        return None

    def render_string(self, path: str, **kwargs: Any) -> bytes:
        """Renders a template and returns it as a string."""
        return self.handler.render_string(path, **kwargs)


class _linkify(UIModule):
    def render(self, text: str, **kwargs: Any) -> str:  # type: ignore
        return escape.linkify(text, **kwargs)


class _xsrf_form_html(UIModule):
    def render(self) -> str:  # type: ignore
        return self.handler.xsrf_form_html()


class TemplateModule(UIModule):
    """UIModule that simply renders the given template.

    {% module Template("foo.html") %} is similar to {% include "foo.html" %},
    but the module version gets its own namespace (with kwargs passed to
    Template()) instead of inheriting the outer template's namespace.

    Templates rendered through this module also get access to UIModule's
    automatic javascript/css features.  Simply call set_resources
    inside the template and give it keyword arguments corresponding to
    the methods on UIModule: {{ set_resources(js_files=static_url("my.js")) }}
    Note that these resources are output once per template file, not once
    per instantiation of the template, so they must not depend on
    any arguments to the template.
    """

    def __init__(self, handler: RequestHandler) -> None:
        super(TemplateModule, self).__init__(handler)
        # keep resources in both a list and a dict to preserve order
        self._resource_list = []  # type: List[Dict[str, Any]]
        self._resource_dict = {}  # type: Dict[str, Dict[str, Any]]

    def render(self, path: str, **kwargs: Any) -> bytes:  # type: ignore
        def set_resources(**kwargs) -> str:  # type: ignore
            if path not in self._resource_dict:
                self._resource_list.append(kwargs)
                self._resource_dict[path] = kwargs
            else:
                if self._resource_dict[path] != kwargs:
                    raise ValueError(
                        "set_resources called with different "
                        "resources for the same template"
                    )
            return ""

        return self.render_string(path, set_resources=set_resources, **kwargs)

    def _get_resources(self, key: str) -> Iterable[str]:
        return (r[key] for r in self._resource_list if key in r)

    def embedded_javascript(self) -> str:
        return "\n".join(self._get_resources("embedded_javascript"))

    def javascript_files(self) -> Iterable[str]:
        result = []
        for f in self._get_resources("javascript_files"):
            if isinstance(f, (unicode_type, bytes)):
                result.append(f)
            else:
                result.extend(f)
        return result

    def embedded_css(self) -> str:
        return "\n".join(self._get_resources("embedded_css"))

    def css_files(self) -> Iterable[str]:
        result = []
        for f in self._get_resources("css_files"):
            if isinstance(f, (unicode_type, bytes)):
                result.append(f)
            else:
                result.extend(f)
        return result

    def html_head(self) -> str:
        return "".join(self._get_resources("html_head"))

    def html_body(self) -> str:
        return "".join(self._get_resources("html_body"))


class _UIModuleNamespace(object):
    """Lazy namespace which creates UIModule proxies bound to a handler."""

    def __init__(
        self, handler: RequestHandler, ui_modules: Dict[str, Type[UIModule]]
    ) -> None:
        self.handler = handler
        self.ui_modules = ui_modules

    def __getitem__(self, key: str) -> Callable[..., str]:
        return self.handler._ui_module(key, self.ui_modules[key])

    def __getattr__(self, key: str) -> Callable[..., str]:
        try:
            return self[key]
        except KeyError as e:
            raise AttributeError(str(e))


def create_signed_value(
    secret: _CookieSecretTypes,
    name: str,
    value: Union[str, bytes],
    version: int = None,
    clock: Callable[[], float] = None,
    key_version: int = None,
) -> bytes:
    if version is None:
        version = DEFAULT_SIGNED_VALUE_VERSION
    if clock is None:
        clock = time.time

    timestamp = utf8(str(int(clock())))
    value = base64.b64encode(utf8(value))
    if version == 1:
        assert not isinstance(secret, dict)
        signature = _create_signature_v1(secret, name, value, timestamp)
        value = b"|".join([value, timestamp, signature])
        return value
    elif version == 2:
        # The v2 format consists of a version number and a series of
        # length-prefixed fields "%d:%s", the last of which is a
        # signature, all separated by pipes.  All numbers are in
        # decimal format with no leading zeros.  The signature is an
        # HMAC-SHA256 of the whole string up to that point, including
        # the final pipe.
        #
        # The fields are:
        # - format version (i.e. 2; no length prefix)
        # - key version (integer, default is 0)
        # - timestamp (integer seconds since epoch)
        # - name (not encoded; assumed to be ~alphanumeric)
        # - value (base64-encoded)
        # - signature (hex-encoded; no length prefix)
        def format_field(s: Union[str, bytes]) -> bytes:
            return utf8("%d:" % len(s)) + utf8(s)

        to_sign = b"|".join(
            [
                b"2",
                format_field(str(key_version or 0)),
                format_field(timestamp),
                format_field(name),
                format_field(value),
                b"",
            ]
        )

        if isinstance(secret, dict):
            assert (
                key_version is not None
            ), "Key version must be set when sign key dict is used"
            assert version >= 2, "Version must be at least 2 for key version support"
            secret = secret[key_version]

        signature = _create_signature_v2(secret, to_sign)
        return to_sign + signature
    else:
        raise ValueError("Unsupported version %d" % version)


# A leading version number in decimal
# with no leading zeros, followed by a pipe.
_signed_value_version_re = re.compile(br"^([1-9][0-9]*)\|(.*)$")


def _get_version(value: bytes) -> int:
    # Figures out what version value is.  Version 1 did not include an
    # explicit version field and started with arbitrary base64 data,
    # which makes this tricky.
    m = _signed_value_version_re.match(value)
    if m is None:
        version = 1
    else:
        try:
            version = int(m.group(1))
            if version > 999:
                # Certain payloads from the version-less v1 format may
                # be parsed as valid integers.  Due to base64 padding
                # restrictions, this can only happen for numbers whose
                # length is a multiple of 4, so we can treat all
                # numbers up to 999 as versions, and for the rest we
                # fall back to v1 format.
                version = 1
        except ValueError:
            version = 1
    return version


def decode_signed_value(
    secret: _CookieSecretTypes,
    name: str,
    value: Union[None, str, bytes],
    max_age_days: int = 31,
    clock: Callable[[], float] = None,
    min_version: int = None,
) -> Optional[bytes]:
    if clock is None:
        clock = time.time
    if min_version is None:
        min_version = DEFAULT_SIGNED_VALUE_MIN_VERSION
    if min_version > 2:
        raise ValueError("Unsupported min_version %d" % min_version)
    if not value:
        return None

    value = utf8(value)
    version = _get_version(value)

    if version < min_version:
        return None
    if version == 1:
        assert not isinstance(secret, dict)
        return _decode_signed_value_v1(secret, name, value, max_age_days, clock)
    elif version == 2:
        return _decode_signed_value_v2(secret, name, value, max_age_days, clock)
    else:
        return None


def _decode_signed_value_v1(
    secret: Union[str, bytes],
    name: str,
    value: bytes,
    max_age_days: int,
    clock: Callable[[], float],
) -> Optional[bytes]:
    parts = utf8(value).split(b"|")
    if len(parts) != 3:
        return None
    signature = _create_signature_v1(secret, name, parts[0], parts[1])
    if not hmac.compare_digest(parts[2], signature):
        gen_log.warning("Invalid cookie signature %r", value)
        return None
    timestamp = int(parts[1])
    if timestamp < clock() - max_age_days * 86400:
        gen_log.warning("Expired cookie %r", value)
        return None
    if timestamp > clock() + 31 * 86400:
        # _cookie_signature does not hash a delimiter between the
        # parts of the cookie, so an attacker could transfer trailing
        # digits from the payload to the timestamp without altering the
        # signature.  For backwards compatibility, sanity-check timestamp
        # here instead of modifying _cookie_signature.
        gen_log.warning("Cookie timestamp in future; possible tampering %r", value)
        return None
    if parts[1].startswith(b"0"):
        gen_log.warning("Tampered cookie %r", value)
        return None
    try:
        return base64.b64decode(parts[0])
    except Exception:
        return None


def _decode_fields_v2(value: bytes) -> Tuple[int, bytes, bytes, bytes, bytes]:
    def _consume_field(s: bytes) -> Tuple[bytes, bytes]:
        length, _, rest = s.partition(b":")
        n = int(length)
        field_value = rest[:n]
        # In python 3, indexing bytes returns small integers; we must
        # use a slice to get a byte string as in python 2.
        if rest[n : n + 1] != b"|":
            raise ValueError("malformed v2 signed value field")
        rest = rest[n + 1 :]
        return field_value, rest

    rest = value[2:]  # remove version number
    key_version, rest = _consume_field(rest)
    timestamp, rest = _consume_field(rest)
    name_field, rest = _consume_field(rest)
    value_field, passed_sig = _consume_field(rest)
    return int(key_version), timestamp, name_field, value_field, passed_sig


def _decode_signed_value_v2(
    secret: _CookieSecretTypes,
    name: str,
    value: bytes,
    max_age_days: int,
    clock: Callable[[], float],
) -> Optional[bytes]:
    try:
        key_version, timestamp_bytes, name_field, value_field, passed_sig = _decode_fields_v2(
            value
        )
    except ValueError:
        return None
    signed_string = value[: -len(passed_sig)]

    if isinstance(secret, dict):
        try:
            secret = secret[key_version]
        except KeyError:
            return None

    expected_sig = _create_signature_v2(secret, signed_string)
    if not hmac.compare_digest(passed_sig, expected_sig):
        return None
    if name_field != utf8(name):
        return None
    timestamp = int(timestamp_bytes)
    if timestamp < clock() - max_age_days * 86400:
        # The signature has expired.
        return None
    try:
        return base64.b64decode(value_field)
    except Exception:
        return None


def get_signature_key_version(value: Union[str, bytes]) -> Optional[int]:
    value = utf8(value)
    version = _get_version(value)
    if version < 2:
        return None
    try:
        key_version, _, _, _, _ = _decode_fields_v2(value)
    except ValueError:
        return None

    return key_version


def _create_signature_v1(secret: Union[str, bytes], *parts: Union[str, bytes]) -> bytes:
    hash = hmac.new(utf8(secret), digestmod=hashlib.sha1)
    for part in parts:
        hash.update(utf8(part))
    return utf8(hash.hexdigest())


def _create_signature_v2(secret: Union[str, bytes], s: bytes) -> bytes:
    hash = hmac.new(utf8(secret), digestmod=hashlib.sha256)
    hash.update(utf8(s))
    return utf8(hash.hexdigest())


def is_absolute(path: str) -> bool:
    return any(path.startswith(x) for x in ["/", "http:", "https:"])
