[TOC]
## 摘要
`Django`的视图类`View`和视图函数在功能上并没有太大差别，`DRF`视图类继承自`Django`，提供了众多的类可以让我们在请求响应中进行组合，提高代码编写率，代码复用率。包括响应渲染类、请求解析类、用户验证类、请求限速类、权限检查类、内容协商类、版本控制类、查询筛选排序类、分页类、序列化类等等。
## API视图
`AOIView`是`DRF`最基本的视图类，它继承了`Django`的视图类`View`。
### 源码分析
#### APIView 类属性以及基础方法
- 四种异常(请求方法、请求限制、身份验证、权限验证)
- 三种参数(解析参数、渲染参数、异常参数)
- 两种视图(视图名称、视图描述)

```python
class APIView(View):
    # 响应渲染类
    renderer_classes = api_settings.DEFAULT_RENDERER_CLASSES
    # 请求解析类
    parser_classes = api_settings.DEFAULT_PARSER_CLASSES
    # 用户验证类
    authentication_classes = api_settings.DEFAULT_AUTHENTICATION_CLASSES
    # 请求限制类
    throttle_classes = api_settings.DEFAULT_THROTTLE_CLASSES
    # 权限检查类
    permission_classes = api_settings.DEFAULT_PERMISSION_CLASSES
    # 内容协商类
    content_negotiation_class = api_settings.DEFAULT_CONTENT_NEGOTIATION_CLASS
    # 元数据
    metadata_class = api_settings.DEFAULT_METADATA_CLASS
    # 版本控制类
    versioning_class = api_settings.DEFAULT_VERSIONING_CLASS

    # 配置对象
    settings = api_settings

    schema = DefaultSchema()

    # 重写父类的方法
    @classmethod
    def as_view(cls, **initkwargs):
        if isinstance(getattr(cls, 'queryset', None), models.query.QuerySet):
            def force_evaluation():
                raise RuntimeError(
                    'Do not evaluate the `.queryset` attribute directly, '
                    'as the result will be cached and reused between requests. '
                    'Use `.all()` or call `.get_queryset()` instead.'
                )
            cls.queryset._fetch_all = force_evaluation
        # 使用父类进行初始化
        view = super().as_view(**initkwargs)
        view.cls = cls
        view.initkwargs = initkwargs

        # csrf
        return csrf_exempt(view)

    # 允许的http 请求方法
    @property
    def allowed_methods(self):
        return self._allowed_methods()

    # 默认响应头设置
    @property
    def default_response_headers(self):
        headers = {
            'Allow': ', '.join(self.allowed_methods),
        }
        if len(self.renderer_classes) > 1:
            headers['Vary'] = 'Accept'
        return headers

    # 请求方法不允许，引发异常
    def http_method_not_allowed(self, request, *args, **kwargs):
        raise exceptions.MethodNotAllowed(request.method)

    # 权限检查不通过，抛出异常
    def permission_denied(self, request, message=None):
        if request.authenticators and not request.successful_authenticator:
            raise exceptions.NotAuthenticated()
        raise exceptions.PermissionDenied(detail=message)

    # 请求被限制，抛出异常
    def throttled(self, request, wait):
        raise exceptions.Throttled(wait)

    # 请求未经过身份验证，设置响应头
    def get_authenticate_header(self, request):
        authenticators = self.get_authenticators()
        if authenticators:
            return authenticators[0].authenticate_header(request)

    # 返回一个字典作为解析函数的参数
    def get_parser_context(self, http_request):
        return {
            'view': self,
            'args': getattr(self, 'args', ()),
            'kwargs': getattr(self, 'kwargs', {})
        }

    # 返回一个字典作为 render 的参数
    def get_renderer_context(self):
        return {
            'view': self,
            'args': getattr(self, 'args', ()),
            'kwargs': getattr(self, 'kwargs', {}),
            'request': getattr(self, 'request', None)
        }

    # 返回一个字典作为异常处理对象的参数
    def get_exception_handler_context(self):
        return {
            'view': self,
            'args': getattr(self, 'args', ()),
            'kwargs': getattr(self, 'kwargs', {}),
            'request': getattr(self, 'request', None)
        }

    # 返回视图的名称
    def get_view_name(self):
        func = self.settings.VIEW_NAME_FUNCTION
        return func(self)

    # 返回视图的描述
    def get_view_description(self, html=False):
        func = self.settings.VIEW_DESCRIPTION_FUNCTION
        return func(self, html)
```
#### API 策略实例化方法
渲染类、解析类、验证类、权限类、限流类、内容协商类等等。这些都是可以在子类中重写的方法，为不同的情况提供不同的类处理对象。
```python
# 检查请求中是否包含 json 风格的后缀
def get_format_suffix(self, **kwargs):
	if self.settings.FORMAT_SUFFIX_KWARG:
		return kwargs.get(self.settings.FORMAT_SUFFIX_KWARG)

# 返回视图使用的渲染器列表
def get_renderers(self):
	return [renderer() for renderer in self.renderer_classes]

# 返回用于解析的类对象列表
def get_parsers(self):
	return [parser() for parser in self.parser_classes]

# 返回用于验证的类对象列表
def get_authenticators(self):
	return [auth() for auth in self.authentication_classes]

# 返回用于检查权限的类对象列表， 默认设置允许所有
def get_permissions(self):
	return [permission() for permission in self.permission_classes]

# 返回节流对象列表
def get_throttles(self):
	return [throttle() for throttle in self.throttle_classes]

# 返回要使用的内容协商对象，如果没有设置则使用默认的类对象
def get_content_negotiator(self):
	if not getattr(self, '_negotiator', None):
		self._negotiator = self.content_negotiation_class()
	return self._negotiator

# 异常处理程序 对象
def get_exception_handler(self):
	return self.settings.EXCEPTION_HANDLER
```
#### API 策略实现方法
内容协商、登录检查、权限检查、限流检查、版本控制的实现。这些方法大多情况下是不需要子类重写也不需要在子类中明确调用。他们都是根据请求流程自动调用检查。
```python
# 根据请求允许的媒体类型返回一个优先级最高的媒体类型与其渲染类对象
def perform_content_negotiation(self, request, force=False):
	renderers = self.get_renderers()
	conneg = self.get_content_negotiator()

	try:
		return conneg.select_renderer(request, renderers, self.format_kwarg)
	except Exception:
		if force:
			return (renderers[0], renderers[0].media_type)
		raise

# 检查用户是否存在
def perform_authentication(self, request):
	request.user

# 检查权限
def check_permissions(self, request):
	for permission in self.get_permissions():
		if not permission.has_permission(request, self):
			self.permission_denied(
				request, message=getattr(permission, 'message', None)
			)

## 检查请求是否拥有对象的权限
def check_object_permissions(self, request, obj):
	for permission in self.get_permissions():
		if not permission.has_object_permission(request, self, obj):
			self.permission_denied(
				request, message=getattr(permission, 'message', None)
			)

# 检查请求限流，默认没有
def check_throttles(self, request):
	throttle_durations = []
	for throttle in self.get_throttles():
		if not throttle.allow_request(request, self):
			throttle_durations.append(throttle.wait())

	if throttle_durations:
		# Filter out `None` values which may happen in case of config / rate
		# changes, see #1438
		durations = [
			duration for duration in throttle_durations
			if duration is not None
		]

		duration = max(durations, default=None)
		self.throttled(request, duration)

## 版本控制
def determine_version(self, request, *args, **kwargs):
	if self.versioning_class is None:
		return (None, None)
	scheme = self.versioning_class()
	return (scheme.determine_version(request, *args, **kwargs), scheme)
```
#### 调度方法
请求流程：封装请求对象`->`内容协商`->`版本控制`->`用户`->`权限`->`限流`->`(操作)`->`(异常 or 结果)
```python
# 初始化包装请求对象
def initialize_request(self, request, *args, **kwargs):
	# 获取解析上下文
	parser_context = self.get_parser_context(request)

	return Request(
		request,
		parsers=self.get_parsers(),
		authenticators=self.get_authenticators(),
		negotiator=self.get_content_negotiator(),
		parser_context=parser_context
	)

# 处理方法开始之前需要做的事 (权限检查等)
def initial(self, request, *args, **kwargs):
	self.format_kwarg = self.get_format_suffix(**kwargs)

	# 内容协商，确定返回的媒体类型与其对应的渲染类对象
	neg = self.perform_content_negotiation(request)
	request.accepted_renderer, request.accepted_media_type = neg

	# Api 版本控制，默认为空
	version, scheme = self.determine_version(request, *args, **kwargs)
	request.version, request.versioning_scheme = version, scheme

	# 用户检查
	self.perform_authentication(request)
	# 权限检查
	self.check_permissions(request)
	# 限流检查
	self.check_throttles(request)

# 处理结果，返回最终的响应对象
def finalize_response(self, request, response, *args, **kwargs):
	# Make the error obvious if a proper response is not returned
	assert isinstance(response, HttpResponseBase), (
		'Expected a `Response`, `HttpResponse` or `HttpStreamingResponse` '
		'to be returned from the view, but received a `%s`'
		% type(response)
	)

	# 如果使用的是框架的 Response 对象
	if isinstance(response, Response):
		if not getattr(request, 'accepted_renderer', None):
			neg = self.perform_content_negotiation(request, force=True)
			request.accepted_renderer, request.accepted_media_type = neg

		# 设置框架响应对象的 处理器以及媒体类型和 参数
		response.accepted_renderer = request.accepted_renderer
		response.accepted_media_type = request.accepted_media_type
		response.renderer_context = self.get_renderer_context()

	# 在响应中添加新的变化头
	vary_headers = self.headers.pop('Vary', None)
	if vary_headers is not None:
		patch_vary_headers(response, cc_delim_re.split(vary_headers))

	# 设置响应头
	for key, value in self.headers.items():
		response[key] = value

	return response

# 处理异常，包装成一个结果
def handle_exception(self, exc):
	if isinstance(exc, (exceptions.NotAuthenticated,
						exceptions.AuthenticationFailed)):
		# WWW-Authenticate header for 401 responses, else coerce to 403
		auth_header = self.get_authenticate_header(self.request)

		if auth_header:
			exc.auth_header = auth_header
		else:
			exc.status_code = status.HTTP_403_FORBIDDEN

	exception_handler = self.get_exception_handler()

	context = self.get_exception_handler_context()
	response = exception_handler(exc, context)

	if response is None:
		self.raise_uncaught_exception(exc)

	response.exception = True
	return response

# 在请求中设置 没有捕捉到异常， 使用明文抛出
def raise_uncaught_exception(self, exc):
	if settings.DEBUG:
		request = self.request
		renderer_format = getattr(request.accepted_renderer, 'format')
		use_plaintext_traceback = renderer_format not in ('html', 'api', 'admin')
		request.force_plaintext_errors(use_plaintext_traceback)
	raise exc

# 和 Djanfo 相似的功能，但是增加了额外的开始，结束，和异常的钩子函数
def dispatch(self, request, *args, **kwargs):
	# 初始化参数
	self.args = args
	self.kwargs = kwargs
	# 包装请求
	request = self.initialize_request(request, *args, **kwargs)
	self.request = request
	self.headers = self.default_response_headers  # deprecate?

	try:
		self.initial(request, *args, **kwargs)

		# 获取请求方法并调用对应的方法处理
		if request.method.lower() in self.http_method_names:
			handler = getattr(self, request.method.lower(),
							  self.http_method_not_allowed)
		else:
			handler = self.http_method_not_allowed

		response = handler(request, *args, **kwargs)

	except Exception as exc:
		response = self.handle_exception(exc)

	self.response = self.finalize_response(request, response, *args, **kwargs)
	return self.response

# http options 方法
def options(self, request, *args, **kwargs):
	if self.metadata_class is None:
		return self.http_method_not_allowed(request, *args, **kwargs)
	data = self.metadata_class().determine_metadata(request, self)
	return Response(data, status=status.HTTP_200_OK)
```
### 示例
```python
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser
from rest_framework.negotiation import DefaultContentNegotiation
from rest_framework.authentication import SessionAuthentication
from rest_framework.permissions import IsAuthenticated
from rest_framework.throttling import AnonRateThrottle
from rest_framework.metadata import SimpleMetadata
from rest_framework.versioning import QueryParameterVersioning
from rest_framework.views import APIView
from rest_framework.response import Response
class DRF_APIView(APIView):
	renderer_classes = [JSONRenderer]
	parser_classes = [JSONParser]
	authentication_classes = [SessionAuthentication]
	throttle_classes = [AnonRateThrottle]
	permission_classes = [IsAuthenticated]
	content_negotiation_class = DefaultContentNegotiation
	metadata_class = SimpleMetadata
	versioning_class = QueryParameterVersioning

	def get(self, request, *args, **kwargs):
		...
		return Response(data=None, status=None, template_name=None, headers=None)

	def put(self, request, *args, **kwargs):
		...
	...
```
## 通用视图
通用视图继承自`APIView`,是对其功能的扩充，增加了序列化，筛选，分页功能。
### 源码分析
```python
class GenericAPIView(views.APIView):
    queryset = None
    serializer_class = None

    # 默认查找的字段
    lookup_field = 'pk'
    # url 传递过来的参数字段 pk 的别名
    lookup_url_kwarg = None

    # 筛选类
    filter_backends = api_settings.DEFAULT_FILTER_BACKENDS
    # 分页类
    pagination_class = api_settings.DEFAULT_PAGINATION_CLASS

    # 获取查询集
    def get_queryset(self):
        assert self.queryset is not None, (
            "'%s' should either include a `queryset` attribute, "
            "or override the `get_queryset()` method."
            % self.__class__.__name__
        )

        queryset = self.queryset
        if isinstance(queryset, QuerySet):
            # Ensure queryset is re-evaluated on each request.
            queryset = queryset.all()
        return queryset

    # 从查询集中获取对象，如果需要支持更多的参数，需要覆盖方法
    def get_object(self):
        queryset = self.filter_queryset(self.get_queryset())

        # Perform the lookup filtering.
        lookup_url_kwarg = self.lookup_url_kwarg or self.lookup_field

        assert lookup_url_kwarg in self.kwargs, (
            'Expected view %s to be called with a URL keyword argument '
            'named "%s". Fix your URL conf, or set the `.lookup_field` '
            'attribute on the view correctly.' %
            (self.__class__.__name__, lookup_url_kwarg)
        )

        filter_kwargs = {self.lookup_field: self.kwargs[lookup_url_kwarg]}
        # 获取对象病检查权限
        obj = get_object_or_404(queryset, **filter_kwargs)

        # May raise a permission denied
        self.check_object_permissions(self.request, obj)

        return obj

    # 获取序列化器对象
    def get_serializer(self, *args, **kwargs):
        serializer_class = self.get_serializer_class()
        kwargs['context'] = self.get_serializer_context()
        return serializer_class(*args, **kwargs)

    # 获取序列化类，可以覆盖以实现更多的序列化操作
    def get_serializer_class(self):
        assert self.serializer_class is not None, (
            "'%s' should either include a `serializer_class` attribute, "
            "or override the `get_serializer_class()` method."
            % self.__class__.__name__
        )

        return self.serializer_class

    # 提供给序列化程序的额外上下文
    def get_serializer_context(self):
        return {
            'request': self.request,
            'format': self.format_kwarg,
            'view': self
        }

    # 对查询接进行筛选
    def filter_queryset(self, queryset):
        for backend in list(self.filter_backends):
            queryset = backend().filter_queryset(self.request, queryset, self)
        return queryset

    # 视图分页类
    @property
    def paginator(self):
        if not hasattr(self, '_paginator'):
            if self.pagination_class is None:
                self._paginator = None
            else:
                self._paginator = self.pagination_class()
        return self._paginator

    # 返回分页的结果
    def paginate_queryset(self, queryset):
        if self.paginator is None:
            return None
        return self.paginator.paginate_queryset(queryset, self.request, view=self)

    # 返回一个分页的 Response， 包含上一页，下一页，当前页数等
    def get_paginated_response(self, data):
        """
        Return a paginated style `Response` object for the given output data.
        """
        assert self.paginator is not None
        return self.paginator.get_paginated_response(data)
```
### 示例
```python
from rest_framework.filters import SearchFilter, OrderingFilter
from rest_framework.pagination import PageNumberPagination
from rest_framework.generics import GenericAPIView
class ExampleView(GenericAPIView):
	queryset = Books.objects.all()
	serializer_class = BooksSerializer
	filter_backends = [SearchFilter, OrderingFilter]
	pagination_class = PageNumberPagination

	def list(self, request, *args, **kwargs):
		# 获取查询集
		queryset = self.get_queryset()
		# 筛选查询集并排序
		filterset = self.filter_queryset(queryset)
		# 对结果进行分页获取查询页对象
		pageset = self.paginate_queryset(filterset)
		# 对最终结果序列化
		serializer = self.get_serializer(pageset, many=True)
		# 返回最终结果
		return Response(serializer.data)

	def get(self, request, *args, **kwargs):
		result = self.get_object()
		serializer = self.get_serializer(result)
		return Response(serializer.data)
```
## 视图集合混入类
只用于继承，不单独使用，重写`as_view`方法增加了`actions`参数，并新增反转url的方法。
### 源码分析
```python
class ViewSetMixin:
    # 重写方法，因为传递了 actions 参数
    @classonlymethod
    def as_view(cls, actions=None, **initkwargs):
        # 名称和描述
        cls.name = None
        cls.description = None

        # viewset 的后缀
        cls.suffix = None

        # viewset 类型
        cls.detail = None

        # 设置基础名称允许反转 url
        cls.basename = None

        # actions must not be empty
        if not actions:
            raise TypeError("The `actions` argument must be provided when "
                            "calling `.as_view()` on a ViewSet. For example "
                            "`.as_view({'get': 'list'})`")

        # sanitize keyword arguments
        for key in initkwargs:
            if key in cls.http_method_names:
                raise TypeError("You tried to pass in the %s method name as a "
                                "keyword argument to %s(). Don't do that."
                                % (key, cls.__name__))
            if not hasattr(cls, key):
                raise TypeError("%s() received an invalid keyword %r" % (
                    cls.__name__, key))

        # 名称和后缀不能同时出现
        if 'name' in initkwargs and 'suffix' in initkwargs:
            raise TypeError("%s() received both `name` and `suffix`, which are "
                            "mutually exclusive arguments." % (cls.__name__))

        def view(request, *args, **kwargs):
            self = cls(**initkwargs)

            # 请求方法的映射
            self.action_map = actions

            # 绑定操作和方法
            for method, action in actions.items():
                handler = getattr(self, action)
                setattr(self, method, handler)

            if hasattr(self, 'get') and not hasattr(self, 'head'):
                self.head = self.get

            self.request = request
            self.args = args
            self.kwargs = kwargs

            # And continue as usual
            return self.dispatch(request, *args, **kwargs)

        update_wrapper(view, cls, updated=())
        update_wrapper(view, cls.dispatch, assigned=())

        view.cls = cls
        view.initkwargs = initkwargs
        view.actions = actions
        return csrf_exempt(view)

    # 初始化请求
    def initialize_request(self, request, *args, **kwargs):
        request = super().initialize_request(request, *args, **kwargs)
        method = request.method.lower()
        # 获取请求方法执行的 action
        if method == 'options':
            self.action = 'metadata'
        else:
            self.action = self.action_map.get(method)
        return request

    # 反转给定的url
    def reverse_action(self, url_name, *args, **kwargs):
        url_name = '%s-%s' % (self.basename, url_name)
        kwargs.setdefault('request', self.request)

        return reverse(url_name, *args, **kwargs)

    # 获取额外的 action 方法
    @classmethod
    def get_extra_actions(cls):
        return [method for _, method in getmembers(cls, _is_extra_action)]

    # 为额外的操作构建一个 url 映射
    def get_extra_action_url_map(self):
        action_urls = OrderedDict()

        # 如果没有设置 detail 则返回空
        if self.detail is None:
            return action_urls

        # 筛选
        actions = [
            action for action in self.get_extra_actions()
            if action.detail == self.detail
        ]

        for action in actions:
            try:
                url_name = '%s-%s' % (self.basename, action.url_name)
                url = reverse(url_name, self.args, self.kwargs, request=self.request)
                view = self.__class__(**action.kwargs)
                action_urls[view.get_view_name()] = url
            except NoReverseMatch:
                pass  # URL requires additional arguments, ignore

        return action_urls
```
## 视图集合
视图集合类继承自视图集合混入类，因此需要传递`actions`参数。
### 源码分析
```python
# 视图集合，不提供任何操作。没有筛选、分页、序列化功能
class ViewSet(ViewSetMixin, views.APIView):
    """
    The base ViewSet class does not provide any actions by default.
    """
    pass

# 通用视图集合，不提供任何操作。有筛选、分页、序列化功能
class GenericViewSet(ViewSetMixin, generics.GenericAPIView):
    """
    The GenericViewSet class does not provide any actions by default,
    but does include the base set of generic view behavior, such as
    the `get_object` and `get_queryset` methods.
    """
    pass

# 模型视图集合，在通用视图的基础上增加了增删改查的方法。
class ModelViewSet(mixins.CreateModelMixin,
                   mixins.RetrieveModelMixin,
                   mixins.UpdateModelMixin,
                   mixins.DestroyModelMixin,
                   mixins.ListModelMixin,
                   GenericViewSet):
    """
    A viewset that provides default `create()`, `retrieve()`, `update()`,
    `partial_update()`, `destroy()` and `list()` actions.
    """
    pass
```