[TOC]
## 摘要
View(视图)主要是根据用户的请求返回数据，用来展示用户可以看到的内容（eg：网页、图片、视频），也可以用来处理用户提交的数据，比如保存到数据库中，Django的视图（View）通常和URL路由一起工作。
## 模型
```python
class Books(models.Model):
    name = models.CharField(max_length=50)
    sums = models.IntegerField(default=0)
    price = models.FloatField(default=0.0)

    def __str__(self):
        return self.name

    class Meta:
        db_table = "books"
```

## 视图
### 方法视图
基于方法的视图优点是直接、容易理解，缺点是不便于继承和重用。
#### 示例
```python
def func_view(request, *args, **kwargs):
    request_method = request.method.lower()
    if request_method == "options":
        pass
    elif request_method == "get":
        pass
    elif request_method == "put":
        pass
    elif request_method == "post":
        pass
    elif request_method == "delete":
        pass
    else:
        pass
```

### 类视图
#### 源码解析
```python
class View:
    # 允许访问的http方法
    http_method_names = ['get', 'post', 'put', 'patch', 'delete', 'head', 'options', 'trace']

    # 构造函数，URLconf 中额外传递的关键参数
    def __init__(self, **kwargs):
        for key, value in kwargs.items():
            setattr(self, key, value)

    # 请求-响应的入口
    @classonlymethod
    def as_view(cls, **initkwargs):
        ...
        def view(request, *args, **kwargs):
            # 类对象实例化
            self = cls(**initkwargs)
            if hasattr(self, 'get') and not hasattr(self, 'head'):
                self.head = self.get
            self.setup(request, *args, **kwargs)
            if not hasattr(self, 'request'):
                raise AttributeError(
                    "%s instance has no 'request' attribute. Did you override "
                    "setup() and forget to call super()?" % cls.__name__
                )
            return self.dispatch(request, *args, **kwargs)
        view.view_class = cls
        view.view_initkwargs = initkwargs

        # 把类的名称和文档绑定在 view 方法上
        update_wrapper(view, cls, updated=())

        update_wrapper(view, cls.dispatch, assigned=())
        return view

    # 初始化所有视图方法共享的属性
    def setup(self, request, *args, **kwargs):
        self.request = request
        self.args = args
        self.kwargs = kwargs

    # 寻找与请求对应的处理方法
    def dispatch(self, request, *args, **kwargs):
        if request.method.lower() in self.http_method_names:
            handler = getattr(self, request.method.lower(), self.http_method_not_allowed)
        else:
            handler = self.http_method_not_allowed
        return handler(request, *args, **kwargs)

    # 返回状态 405，并把允许的请求方法发到响应头中
    def http_method_not_allowed(self, request, *args, **kwargs):
        logger.warning(
            'Method Not Allowed (%s): %s', request.method, request.path,
            extra={'status_code': 405, 'request': request}
        )
        return HttpResponseNotAllowed(self._allowed_methods())

    # 把允许的方法返回
    def options(self, request, *args, **kwargs):
        """Handle responding to requests for the OPTIONS HTTP verb."""
        response = HttpResponse()
        response['Allow'] = ', '.join(self._allowed_methods())
        response['Content-Length'] = '0'
        return response

    def _allowed_methods(self):
        return [m.upper() for m in self.http_method_names if hasattr(self, m)]
```
#### 示例
```python
from django.views import View
class class_view(View):
    def get(self, request, *args, **kwargs):
        pass
    def put(self, request, *args, **kwargs):
        pass
    def post(self, request, *args, **kwargs):
        pass
    def delete(self, request, *args, **kwargs):
        pass
```

### 列表类视图
列表类视图我们继承`BaseListView`这个类，`ListView`继承了`BaseListView`和一个`django`模板类，我们不需要用到模板所以就不使用它。这个类只为我们实现了默认的`get`方法，其他方法我们根据需要自行添加。
#### 源码分析
```python
class MultipleObjectMixin(ContextMixin):
    # 是否允许列表为空
    allow_empty = True
    # 查询集合(优先) 与下面的模型二选一
    queryset = None
    # 模型
    model = None
    # 分页 每页的数量
    paginate_by = None
    # 分页的最大数量
    paginate_orphans = 0
    # 模板使用的上下文对象名称
    context_object_name = None
    # 分页使用的类
    paginator_class = Paginator
    # 请求时携带的指定页面的参数
    page_kwarg = 'page'
    # 排序 str or list or set
    ordering = None

    # 获取查询集合并排序
    def get_queryset(self):
        # 指定查询集合或者使用模型的的集合
        if self.queryset is not None:
            queryset = self.queryset
            if isinstance(queryset, QuerySet):
                queryset = queryset.all()
        elif self.model is not None:
            queryset = self.model._default_manager.all()
        else:
            raise ImproperlyConfigured(
                "%(cls)s is missing a QuerySet. Define "
                "%(cls)s.model, %(cls)s.queryset, or override "
                "%(cls)s.get_queryset()." % {
                    'cls': self.__class__.__name__
                }
            )
        # 排序参数
        ordering = self.get_ordering()
        if ordering:
            if isinstance(ordering, str):
                ordering = (ordering,)
            queryset = queryset.order_by(*ordering)
        # 返回可迭代的查询集
        return queryset

    # 获取查询集排序使用的字段
    def get_ordering(self):
        return self.ordering

    # 如果需要，对查询集分页
    def paginate_queryset(self, queryset, page_size):
        # 分页对象
        paginator = self.get_paginator(
            queryset, page_size, orphans=self.get_paginate_orphans(),
            allow_empty_first_page=self.get_allow_empty())
        # 取出请求参数的值
        page_kwarg = self.page_kwarg
        page = self.kwargs.get(page_kwarg) or self.request.GET.get(page_kwarg) or 1
        try:
            # 指定的第几页
            page_number = int(page)
        except ValueError:
            if page == 'last':
                page_number = paginator.num_pages
            else:
                raise Http404(_("Page is not 'last', nor can it be converted to an int."))
        try:
            # 获取指定页的对象
            page = paginator.page(page_number)
            return (paginator, page, page.object_list, page.has_other_pages())
        except InvalidPage as e:
            raise Http404(_('Invalid page (%(page_number)s): %(message)s') % {
                'page_number': page_number,
                'message': str(e)
            })

    # 每页的数量
    def get_paginate_by(self, queryset):
        return self.paginate_by

    # 对查询集进行分页处理
    def get_paginator(self, queryset, per_page, orphans=0,
                      allow_empty_first_page=True, **kwargs):
        return self.paginator_class(
            queryset, per_page, orphans=orphans,
            allow_empty_first_page=allow_empty_first_page, **kwargs)

    # 获取最大的分页数
    def get_paginate_orphans(self):
        return self.paginate_orphans

    # 分页是否允许为空
    def get_allow_empty(self):
        return self.allow_empty

    # 模板使用的上下文对象名称
    def get_context_object_name(self, object_list):
        if self.context_object_name:
            return self.context_object_name
        elif hasattr(object_list, 'model'):
            return '%s_list' % object_list.model._meta.model_name
        else:
            return None

    # 处理查询集，返回 contex 对象
    def get_context_data(self, *, object_list=None, **kwargs):
        queryset = object_list if object_list is not None else self.object_list
        page_size = self.get_paginate_by(queryset)
        context_object_name = self.get_context_object_name(queryset)
        # 需要分页
        if page_size:
            paginator, page, queryset, is_paginated = self.paginate_queryset(queryset, page_size)
            context = {
                'paginator': paginator,
                'page_obj': page,
                'is_paginated': is_paginated,
                'object_list': queryset
            }
        # 不需要分页，返回完整查询集
        else:
            context = {
                'paginator': None,
                'page_obj': None,
                'is_paginated': False,
                'object_list': queryset
            }
        if context_object_name is not None:
            context[context_object_name] = queryset
        context.update(kwargs)
        return super().get_context_data(**context)

class BaseListView(MultipleObjectMixin, View):
    # 开始处理请求
    def get(self, request, *args, **kwargs):
        # 获取查询集列表并排序
        self.object_list = self.get_queryset()
        # 是否允许为空列表
        allow_empty = self.get_allow_empty()
        if not allow_empty:
            if self.get_paginate_by(self.object_list) is not None and hasattr(self.object_list, 'exists'):
                is_empty = not self.object_list.exists()
            else:
                is_empty = not self.object_list
            if is_empty:
                raise Http404(_("Empty list and '%(class_name)s.allow_empty' is False.") % {
                    'class_name': self.__class__.__name__,
                })
        # 处理查询集(分页等)
        context = self.get_context_data()
        # 不使用模板则需要子类重写这个方法
        return self.render_to_response(context)
```
#### 示例
```python
from django.views.generic.list import BaseListView
class class_list(BaseListView):
    allow_empty = True
    # 查询集和模型二选一，查询集优先
    queryset = Books.objects.all()
    model = Books
    # 每页的数量
    paginate_by = 10
    # 最大的分页数
    paginate_orphans = 0
    # 请求指定页面的参数
    page_kwarg = "page"
    ordering = ["id"]

    #方法一：重写 render_to_response 方法，因为我们不使用模板所以需要自己序列化返回的对象
    def render_to_response(self, context):
        # 分页对象
        paginator = context["paginator"]
        # 当前选择页对象
        page_obj = context["page_obj"]
        # 是否还有其他页
        is_paginated = context["is_paginated"]
        # 当前页包含的所有对象的列表， 是一个 queryset 对象
        object_list = context["object_list"]
        # 对象需要自己序列化
        result = []
        for one in object_list:
            result.append({"name": one.name, "sums":one.sums, "price":one.price})
        return JsonResponse(result, safe=False)

    #方法二：重写 get 方法，自己处理请求
    def get(self, request, *args, **kwargs):
        pass
```

### 其他类视图
`django`还提供了许多通用的基于类的视图，来帮我们简化视图代码的编写。

- CreateView
- UpdateView
- DetailView
- DeleteView
- FormView