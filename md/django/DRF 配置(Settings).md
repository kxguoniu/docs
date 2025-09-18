[TOC]
## 摘要
`DRF`提供了配置参数可供我们在`Django`配置文件中设置全局默认的参数，也可以在视图中进行指定配置。更高级的也可以重写试图方法根据需要选择不同的配置。
## RENDERER(渲染器)
```python
# 配置文件
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',    #json 渲染器
        'rest_framework.renderers.BrowsableAPIRenderer',    #text/html 模板渲染器
    ]
}

# 视图中指定
from rest_framework.renderers import JSONRenderer
from rest_framework.views import APIView
class ExampleView(APIView):
    renderer_classes = [JSONRenderer]

    # 重写可以根据自己的需要返回不同的渲染器对象
    def get_renderers(self):
        # if else ...
        return [JSONRenderer()]
```

## PARSER(解析器)
```python
# 配置文件
REST_FRAMEWORK = {
    'DEFAULT_PARSER_CLASSES': [
        'rest_framework.parsers.JSONParser',
        'rest_framework.parsers.FormParser',
        'rest_framework.parsers.MultiPartParser'
    ]
}

# 视图中设置
from rest_framework.parsers import JSONParser
from rest_framework.views import APIView
class ExampleView(APIView):
    parser_classes = [JSONParser]

    # 重写根据需要选择解析器对象
    def get_parsers(self):
        return [JSONParser()]
```

## CONTENT_NEGOTIATION(内容协商)
```python
# 配置文件
REST_FRAMEWORK = {
    'DEFAULT_CONTENT_NEGOTIATION_CLASS': 'rest_framework.negotiation.DefaultContentNegotiation'
}

# 视图中设置
from rest_framework.negotiation import DefaultContentNegotiation
from rest_framework.views import APIView
class ExampleView(APIView):
    content_negotiation_class = DefaultContentNegotiation

    # 重写根据需要选择内容协商对象
    def get_content_negotiator(self):
        self._negotiator = DefaultContentNegotiation()
        return self._negotiator
```

## AUTHENTICATION(认证)
```python
# 配置文件
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',  #session认证
        'rest_framework.authentication.BasicAuthentication'     #基础认证
    ],
    # 未经过身份验证请求的 user与token
    'UNAUTHENTICATED_USER': 'django.contrib.auth.models.AnonymousUser',
    'UNAUTHENTICATED_TOKEN': None,
}

# 视图中设置
from rest_framework.authentication import BasicAuthentication, SessionAuthentication
from rest_framework.views import APIView
class ExampleView(APIView):
    authentication_classes = [BasicAuthentication, SessionAuthentication]

    # 根据需要返回不同的认证类对象
    def get_authenticators(self):
        return [BasicAuthentication(), SessionAuthentication()]
```

## PERMISSION(权限)
```python
# 配置文件
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.AllowAny',  #允许所有用户
    ]
}

# 视图中指定
from rest_framework.permissions import IsAuthenticated
from rest_framework.views import APIView
class ExampleView(APIView):
    # 经过认证的用户
    permission_classes = [IsAuthenticated]

    # 可以根据不同的条件返回不同的验证对象
    def get_permissions(self):
        return [IsAuthenticated()]
```

## THROTTLE(限流)
```python
# 配置文件
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        "rest_framework.throttling.AnonRateThrottle",   #匿名
        "rest_framework.throttling.UserRateThrottle",   #实名
        "rest_framework.throttling.ScopedRateThrottle"  #自定义
    ],
    'DEFAULT_THROTTLE_RATES': {
        # eg 100/s  100/m  100/h  100/d
        'user': None,   #实名
        'anon': None,   #匿名
    },
    # 获取请求地址时使用，不设置就使用请求的正式地址或者转发链
    # 如果设置了数值则表示转发最大数，如果超过了则使用组大限度转发地址
    'NUM_PROXIES': None,
}

# 视图中指定
from rest_framework.throttling import AnonRateThrottle, UserRateThrottle
from rest_framework,views import APIView
class ExampleView(APIView):
    throttle_classes = [AnonRateThrottle, UserRateThrottle]

    # 重写已根据不同情况使用不同的限流对象
    def get_throttles(self):
        return [AnonRateThrottle(), UserRateThrottle()]
```

## METADATA(元数据)
```python
# 配置文件
REST_FRAMEWORK = {
    'DEFAULT_METADATA_CLASS': 'rest_framework.metadata.SimpleMetadata'
}

# 为视图指定元数据类
from rest_framework.metadata import SimpleMetadata
from rest_framework.views import APIView
class ExampleView(APIView):
    metadata_class = SimpleMetadata
```

## VERSIONING(版本控制)
```python
# 配置文件
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': "rest_framework.versioning.NamespaceVersioning",
    # 默认版本
    'DEFAULT_VERSION': None,
    # 允许的版本列表/集合
    'ALLOWED_VERSIONS': None,
    # 版本查询参数
    'VERSION_PARAM': 'version',
}

# 为视图指定版本控制类
from rest_framework.versioning import NamespaceVersioning
from rest_framework.views import APIView
class ExampleView(APIView):
    versioning_class = NamespaceVersioning
```

## EXCEPTION(异常处理)
```python
# 配置文件
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'rest_framework.views.exception_handler',
    'NON_FIELD_ERRORS_KEY': 'non_field_errors',
}

# 为视图设置异常处理程序
from rest_framework.views import exception_handler
from rest_framework.views import APIView
class ExampleView(APIView):
    # 重新设置视图的异常处理对象
    def get_exception_handler(self):
        return exception_handler
```

## SCHEMA(架构图)
```python
# 配置文件
REST_FRAMEWORK = {
    'DEFAULT_SCHEMA_CLASS': 'rest_framework.schemas.openapi.AutoSchema',
    'SCHEMA_COERCE_PATH_PK': True,
    'SCHEMA_COERCE_METHOD_NAMES': {
        'retrieve': 'read',
        'destroy': 'delete'
    },
}
```

## FILTER(过滤)
```python
# 配置文件
INSTALLED_APPS = [
    ...
    'django_filters',   #需要注册
]
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': [
        "django_filters.rest_framework.DjangoFilterBackend"
    ],
    # 请求的搜索参数和排序参数
    'SEARCH_PARAM': 'search',
    'ORDERING_PARAM': 'ordering',
}

# 视图中指定
from rest_framework.filters import SearchFilter, OrderingFilter
from rest_framework.generics import ListAPIView
class ExampleView(APIView):
    queryset = Books.objects.all()
    filter_backends = [SearchFilter, OrderingFilter]
    # 前缀，相等，搜索，正则，完整
    # 传递的搜索列表中的属性每一个线与这些or，然后结果之间and
    search_fields = ["^arg1", "=arg2", "@arg3", "$arg4", "arg5"]
    # 默认的排序列表
    ordering = ["agr1", "-arg2"]
    # 可以使用的排序字段，排序参数必须在这些字段值中
    ordering_fields = ["arg1", "arg2"]
```

## PAGINATION(分页)
```python
# 配置文件
REST_FRAMEWORK = {
    # 分页类与每页数目
    'DEFAULT_PAGINATION_CLASS': "rest_framework.pagination.PageNumberPagination",
    'PAGE_SIZE': 10
}

# 视图中指定
from rest_framework.pagination import PageNumberPagination
from rest_framework.generics import ListAPIView
class LargeResult(PageNumberPagination):
    # 默认的每页大小，如果不设置使用配置文件中的值
    page_size = 10
    # 查询参数设置的页面大小
    page_size_query_param = "page_size"
    # 每页最大的数量
    max_page_size = 1000
    # 查询第几页的参数
    page_query_param = "page"

class ExampleView(ListAPIView):
    pagination_class = LargeResult
```
## 其他配置
```python
REST_FRAMEWORK = {
    # 渲染器使用的日期和时间格式
    'DATE_FORMAT': ISO_8601,
    'DATE_INPUT_FORMATS': [ISO_8601],

    'DATETIME_FORMAT': ISO_8601,
    'DATETIME_INPUT_FORMATS': [ISO_8601],

    'TIME_FORMAT': ISO_8601,
    'TIME_INPUT_FORMATS': [ISO_8601],

    # 允许响应中使用unicode字符，设置false将转义为非ascii字符 eg：”\u2605“
    'UNICODE_JSON': True,
    # JSON响应返回紧凑表示形式，False 返回详细的表示形式。就是响应中有没有空格
    'COMPACT_JSON': True,
    # JSON渲染和解析只会遵循语法有效的JSON字符，排除(nan,inf,-inf)等，因为Java和PG的Json类型不支持这类数据
    # 设置False将允许呈现和解析，但需要我们特殊处理
    'STRICT_JSON': True,
    'COERCE_DECIMAL_TO_STRING': True,
    'UPLOADED_FILES_USE_URL': True,

    # 可以使用参数的方式指定响应类型   eg： http://localhost/api?format=json, http://localhost/api/ceshi.csv/
    'URL_FORMAT_OVERRIDE': 'format',
    'FORMAT_SUFFIX_KWARG': 'format',
    # 生成URL字段的密钥
    'URL_FIELD_NAME': 'url',
}
```