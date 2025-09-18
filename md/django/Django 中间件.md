[TOC]
## 摘要
中间件是一个用来处理`Django`的请求和响应的框架级别的钩子。它是一个轻量、低级别的插件系统，用于在全局范围内改变`Django`的输入和输出。每个中间件组件负责做一些特定的功能。
由于其影响的是全局，所以需要谨慎使用，使用不当会影响性能。中间件可以帮助我们在视图函数执行之前和执行之后做一些额外的操作。
## 示例
```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```
## 中间件混合类
```python
# django.util.deprecation
class MiddlewareMixin:
    def __init__(self, get_response=None):
        self.get_response = get_response
        super().__init__()

    def __call__(self, request):
        response = None
        if hasattr(self, 'process_request'):
            response = self.process_request(request)
        response = response or self.get_response(request)
        if hasattr(self, 'process_response'):
            response = self.process_response(request, response)
        return response

# 自定义中间件
class Example(MiddlewareMixin):
    def __init__(self, get_response=None):
        self.get_response = get_response
        pass

    def process_request(self, request):
        pass
        return HttpResponse or None

    def process_view(self, request, view_func, view_args, view_kwargs):
        pass
        return HttpResponse or None

    def process_template_response(self, requst, response):
        pass
        return HttpResponse or None

    def process_exception(self, request, exception):
        pass
        return HttpResponse or None

    def process_response(self, resonse):
        pass
        return HttpResponse or None
```
## 安全中间件
请求重定向: `http` 转 `https`，`xss`攻击过滤等网站安全设置。
```python
class SecurityMiddleware(MiddlewareMixin):
    def __init__(self, get_response=None):
        # 过期时间
        self.sts_seconds = settings.SECURE_HSTS_SECONDS
        # 是否包含网站下的所有子域名
        self.sts_include_subdomains = settings.SECURE_HSTS_INCLUDE_SUBDOMAINS
        # 预加载，非标准
        self.sts_preload = settings.SECURE_HSTS_PRELOAD
        # 禁用浏览器内容嗅探
        self.content_type_nosniff = settings.SECURE_CONTENT_TYPE_NOSNIFF
        # 设置默认的xss过滤，如果检测到攻击，浏览器阻止页面加载
        self.xss_filter = settings.SECURE_BROWSER_XSS_FILTER
        # 重定向开关
        self.redirect = settings.SECURE_SSL_REDIRECT
        # 重定向地址，不配置就是原地址
        self.redirect_host = settings.SECURE_SSL_HOST
        # 不启用重定向的正则匹配
        self.redirect_exempt = [re.compile(r) for r in settings.SECURE_REDIRECT_EXEMPT]
        self.get_response = get_response

    # 检查重定向
    def process_request(self, request):
        path = request.path.lstrip("/")
        if (self.redirect and not request.is_secure() and
                not any(pattern.search(path)
                        for pattern in self.redirect_exempt)):
            host = self.redirect_host or request.get_host()
            return HttpResponsePermanentRedirect(
                "https://%s%s" % (host, request.get_full_path())
            )

    # 设置默认响应头
    def process_response(self, request, response):
        # 告诉浏览器只能通过https访问当前资源
        if (self.sts_seconds and request.is_secure() and
                'Strict-Transport-Security' not in response):
            sts_header = "max-age=%s" % self.sts_seconds
            if self.sts_include_subdomains:
                sts_header = sts_header + "; includeSubDomains"
            if self.sts_preload:
                sts_header = sts_header + "; preload"
            response['Strict-Transport-Security'] = sts_header

        # 禁用浏览器内容嗅探
        if self.content_type_nosniff:
            response.setdefault('X-Content-Type-Options', 'nosniff')

        # 遇到xss攻击，禁止加载页面
        if self.xss_filter:
            response.setdefault('X-XSS-Protection', '1; mode=block')

        return response
```
## 会话中间件
开启会话支持，`Session`支持中间件，加入这个中间件会在数据库(设置)中保存会话信息
```python
class SessionMiddleware(MiddlewareMixin):
    def __init__(self, get_response=None):
        self.get_response = get_response
        engine = import_module(settings.SESSION_ENGINE)
        # session 存储类
        self.SessionStore = engine.SessionStore

    def process_request(self, request):
        # 获取请求中的 session_key
        session_key = request.COOKIES.get(settings.SESSION_COOKIE_NAME)
        # 设置请求的session对象
        request.session = self.SessionStore(session_key)

    def process_response(self, request, response):
        # 保存、修改、空
        try:
            accessed = request.session.accessed
            modified = request.session.modified
            empty = request.session.is_empty()
        except AttributeError:
            pass
        else:
            # 如果请求中有 cookie 但是在本次会话中被删除，则删除响应中的 cookie
            if settings.SESSION_COOKIE_NAME in request.COOKIES and empty:
                response.delete_cookie(
                    settings.SESSION_COOKIE_NAME,
                    path=settings.SESSION_COOKIE_PATH,
                    domain=settings.SESSION_COOKIE_DOMAIN,
                )
            else:
                if accessed:
                    patch_vary_headers(response, ('Cookie',))
                # 如果cookie被改变或者每次响应都需要保存cookie
                if (modified or settings.SESSION_SAVE_EVERY_REQUEST) and not empty:
                    # 会话在浏览器关闭时过期
                    if request.session.get_expire_at_browser_close():
                        max_age = None
                        expires = None
                    # 设置过期时间
                    else:
                        max_age = request.session.get_expiry_age()
                        expires_time = time.time() + max_age
                        expires = http_date(expires_time)
                    # 设置并保存会话
                    if response.status_code != 500:
                        try:
                            request.session.save()
                        except UpdateError:
                            raise SuspiciousOperation(
                                "The request's session was deleted before the "
                                "request completed. The user may have logged "
                                "out in a concurrent request, for example."
                            )
                        response.set_cookie(
                            settings.SESSION_COOKIE_NAME,
                            request.session.session_key, max_age=max_age,
                            expires=expires, domain=settings.SESSION_COOKIE_DOMAIN,
                            path=settings.SESSION_COOKIE_PATH,
                            secure=settings.SESSION_COOKIE_SECURE or None,
                            httponly=settings.SESSION_COOKIE_HTTPONLY or None,
                            samesite=settings.SESSION_COOKIE_SAMESITE,
                        )
        return response
```
## 通用中间件
处理`URL`，比如`52pyc.cn`会自定处理成`www.52pyc.cn`。比如`/blog/111`会处理成`/blog/111/`自动加上反斜杠。还有禁用某些代理。
```python
class CommonMiddleware(MiddlewareMixin):
    response_redirect_class = HttpResponsePermanentRedirect

    # 检查前缀 www 和后缀 /,并判断是否需要重定向
    def process_request(self, request):
        # 检查被拒绝的用户代理
        if 'HTTP_USER_AGENT' in request.META:
            for user_agent_regex in settings.DISALLOWED_USER_AGENTS:
                if user_agent_regex.search(request.META['HTTP_USER_AGENT']):
                    raise PermissionDenied('Forbidden user agent')

        # 检查重定向，在子域名前面加上 www
        host = request.get_host()
        must_prepend = settings.PREPEND_WWW and host and not host.startswith('www.')
        redirect_url = ('%s://www.%s' % (request.scheme, host)) if must_prepend else ''

        # 检查路径重定向
        if self.should_redirect_with_slash(request):
            path = self.get_full_path_with_slash(request)
        else:
            path = request.get_full_path()

        # 如果需要重定向，返回重定向
        if redirect_url or path != request.get_full_path():
            redirect_url += path
            return self.response_redirect_class(redirect_url)

    # 检查是否需要路径重定向
    def should_redirect_with_slash(self, request):
        # 当前路径不存在并且在末尾加上斜杠后路径存在返回 True
        if settings.APPEND_SLASH and not request.path_info.endswith('/'):
            urlconf = getattr(request, 'urlconf', None)
            return (
                not is_valid_path(request.path_info, urlconf) and
                is_valid_path('%s/' % request.path_info, urlconf)
            )
        return False

    # 在路径后面加一个斜杠后返回，调试模式则抛出异常
    def get_full_path_with_slash(self, request):
        new_path = request.get_full_path(force_append_slash=True)
        # Prevent construction of scheme relative urls.
        new_path = escape_leading_slashes(new_path)
        if settings.DEBUG and request.method in ('POST', 'PUT', 'PATCH'):
            raise RuntimeError(
                "You called this URL via %(method)s, but the URL doesn't end "
                "in a slash and you have APPEND_SLASH set. Django can't "
                "redirect to the slash URL while maintaining %(method)s data. "
                "Change your form to point to %(url)s (note the trailing "
                "slash), or set APPEND_SLASH=False in your Django settings." % {
                    'method': request.method,
                    'url': request.get_host() + new_path,
                }
            )
        return new_path

    def process_response(self, request, response):
        # 如果响应状态为 404，检查斜杠路径
        if response.status_code == 404:
            if self.should_redirect_with_slash(request):
                return self.response_redirect_class(self.get_full_path_with_slash(request))

        # 设置协议的默认响应内容长度
        if not response.streaming and not response.has_header('Content-Length'):
            response['Content-Length'] = str(len(response.content))

        return response
```
## CSRF中间件
加入这个中间件在提交表单的时候必须加入`csrf_token`,`cookie`中也会生成一个`csrftoken`值，也会在请求头中加入一个`HTTP_X_CSRFTOKEN`放置`csrftoken`的值。
```python
class CsrfViewMiddleware(MiddlewareMixin):
    # 添加自定义属性避免多次检查，尤其是使用装饰器时
    def _accept(self, request):
        request.csrf_processing_done = True
        return None

    # 返回异常，记录日志
    def _reject(self, request, reason):
        response = _get_failure_view()(request, reason=reason)
        log_response(
            'Forbidden (%s): %s', reason, request.path,
            response=response,
            request=request,
            logger=logger,
        )
        return response

    def _get_token(self, request):
        # 如果启用了会话保存csrf token
        if settings.CSRF_USE_SESSIONS:
            try:
                return request.session.get(CSRF_SESSION_KEY)
            except AttributeError:
                raise ImproperlyConfigured(
                    'CSRF_USE_SESSIONS is enabled, but request.session is not '
                    'set. SessionMiddleware must appear before CsrfViewMiddleware '
                    'in MIDDLEWARE%s.' % ('_CLASSES' if settings.MIDDLEWARE is None else '')
                )
        # 如果没有使用会话保存，而是使用了用户保存
        else:
            try:
                cookie_token = request.COOKIES[settings.CSRF_COOKIE_NAME]
            except KeyError:
                return None

            csrf_token = _sanitize_token(cookie_token)
            if csrf_token != cookie_token:
                # Cookie token needed to be replaced;
                # the cookie needs to be reset.
                request.csrf_cookie_needs_reset = True
            return csrf_token

    def _set_token(self, request, response):
        # 如果使用会话保存token
        if settings.CSRF_USE_SESSIONS:
            if request.session.get(CSRF_SESSION_KEY) != request.META['CSRF_COOKIE']:
                request.session[CSRF_SESSION_KEY] = request.META['CSRF_COOKIE']
        else:
            response.set_cookie(
                settings.CSRF_COOKIE_NAME,
                request.META['CSRF_COOKIE'],
                max_age=settings.CSRF_COOKIE_AGE,
                domain=settings.CSRF_COOKIE_DOMAIN,
                path=settings.CSRF_COOKIE_PATH,
                secure=settings.CSRF_COOKIE_SECURE,
                httponly=settings.CSRF_COOKIE_HTTPONLY,
                samesite=settings.CSRF_COOKIE_SAMESITE,
            )
            # Set the Vary header since content varies with the CSRF cookie.
            patch_vary_headers(response, ('Cookie',))

    # 获取csrf token
    def process_request(self, request):
        csrf_token = self._get_token(request)
        if csrf_token is not None:
            # Use same token next time.
            request.META['CSRF_COOKIE'] = csrf_token

    # 在进入视图之前检查，跨域请求，令牌检查
    def process_view(self, request, callback, callback_args, callback_kwargs):
        # 如果csrf 已经检查过跳过检查
        if getattr(request, 'csrf_processing_done', False):
            return None

        # 直到 request.META["CSRF_COOKIE"] 被处理完才退出，这样 get_token 还可以工作
        if getattr(callback, 'csrf_exempt', False):
            return None

        # 对不安全的请求方法进行处理
        if request.method not in ('GET', 'HEAD', 'OPTIONS', 'TRACE'):
            # 不进行csrf检查
            if getattr(request, '_dont_enforce_csrf_checks', False):
                return self._accept(request)

            # 不允许其他地址引用，除非在允许列表中
            if request.is_secure():
                # 检查 referer(来源地址) 是否存在
                referer = request.META.get('HTTP_REFERER')
                if referer is None:
                    return self._reject(request, REASON_NO_REFERER)

                referer = urlparse(referer)

                # 确保 referer 符合标准(有效)
                if '' in (referer.scheme, referer.netloc):
                    return self._reject(request, REASON_MALFORMED_REFERER)

                # 确保来源地址是 https
                if referer.scheme != 'https':
                    return self._reject(request, REASON_INSECURE_REFERER)

                # cookie 有效的域
                good_referer = (
                    settings.SESSION_COOKIE_DOMAIN
                    if settings.CSRF_USE_SESSIONS
                    else settings.CSRF_COOKIE_DOMAIN
                )
                # 增加端口
                if good_referer is not None:
                    server_port = request.get_port()
                    if server_port not in ('443', '80'):
                        good_referer = '%s:%s' % (good_referer, server_port)
                else:
                    try:
                        good_referer = request.get_host()
                    except DisallowedHost:
                        pass

                # 创建一个列表包含所有允许的引用地址
                good_hosts = list(settings.CSRF_TRUSTED_ORIGINS)
                if good_referer is not None:
                    good_hosts.append(good_referer)

                # 检查引用的地址是否在允许的列表中
                if not any(is_same_domain(referer.netloc, host) for host in good_hosts):
                    reason = REASON_BAD_REFERER % referer.geturl()
                    return self._reject(request, reason)

            # 我们拒绝没有csrf_token的不安全提交
            csrf_token = request.META.get('CSRF_COOKIE')
            if csrf_token is None:
                return self._reject(request, REASON_NO_CSRF_COOKIE)

            # 检查令牌是否匹配
            request_csrf_token = ""
            if request.method == "POST":
                try:
                    request_csrf_token = request.POST.get('csrfmiddlewaretoken', '')
                except IOError:
                    pass

            # 不是POST方法或者POST方法请求令牌不存在，从请求头中获取令牌
            if request_csrf_token == "":
                request_csrf_token = request.META.get(settings.CSRF_HEADER_NAME, '')

            # 验证请求令牌和csrf_token是否一致
            request_csrf_token = _sanitize_token(request_csrf_token)
            if not _compare_salted_tokens(request_csrf_token, csrf_token):
                return self._reject(request, REASON_BAD_TOKEN)

        return self._accept(request)

    # 设置csrf cookie
    def process_response(self, request, response):
        # csrf cookie 不需要重置
        if not getattr(request, 'csrf_cookie_needs_reset', False):
            if getattr(response, 'csrf_cookie_set', False):
                return response
        # 如果没有使用csrf 配置
        if not request.META.get("CSRF_COOKIE_USED", False):
            return response

        # 设置csrf cookie
        self._set_token(request, response)
        response.csrf_cookie_set = True
        return response
```
## 验证中间件
会在每个请求对象到达`view`之前添加当前用户的`user`属性，可以在`view`中直接访问。
```python
def get_user(request):
    if not hasattr(request, '_cached_user'):
        request._cached_user = auth.get_user(request)
    return request._cached_user

class AuthenticationMiddleware(MiddlewareMixin):
    def process_request(self, request):
        assert hasattr(request, 'session'), (
            "The Django authentication middleware requires session middleware "
            "to be installed. Edit your MIDDLEWARE%s setting to insert "
            "'django.contrib.sessions.middleware.SessionMiddleware' before "
            "'django.contrib.auth.middleware.AuthenticationMiddleware'."
        ) % ("_CLASSES" if settings.MIDDLEWARE is None else "")
        # 设置用户，一个懒加载对象
        request.user = SimpleLazyObject(lambda: get_user(request))
```
## 消息中间件
展示一些后台信息给前端页面，如果想要使用还要在`INSTALLED_APPS`中添加`django.contrib.message`才能生效。
```python
class MessageMiddleware(MiddlewareMixin):
    def process_request(self, request):
        # 设置消息存储类
        request._messages = default_storage(request)

    def process_response(self, request, response):
        if hasattr(request, '_messages'):
            unstored_messages = request._messages.update(response)
            if unstored_messages and settings.DEBUG:
                raise ValueError('Not all temporary messages could be stored.')
        return response
```
## 点击劫持中间件
防止通过浏览器页面跨`Frame`出现`clickjacking`攻击出现
```python
class XFrameOptionsMiddleware(MiddlewareMixin):
    def process_response(self, request, response):
        # 如果已经在响应中则不需要设置
        if response.get('X-Frame-Options') is not None:
            return response

        # 如果使用了 @xframe_options_exempt 则不设置
        if getattr(response, 'xframe_options_exempt', False):
            return response

        # 设置响应同一站点下的页面
        response['X-Frame-Options'] = self.get_xframe_options_value(request,
                                                                    response)
        return response

    def get_xframe_options_value(self, request, response):
        return getattr(settings, 'X_FRAME_OPTIONS', 'SAMEORIGIN').upper()
```