### ListView

#### as_view()

视图在 urlpattern 中的调用入口. 返回完成初始化的类方法 dispatch() 方法作为闭包. 即当对应 url 模式匹配时执行如下操作: 

- 创建对应的视图类, 将 urlpattern 中对视图的设置参数做初始化, 将其设置为视图类属性, 调用类的 setup() 方法. 
- 即当对应 url 模式匹配时, 调用创建类视图的 dispatch() 方法 

#### setup()

```python
    def setup(self, request, *args, **kwargs):
        """Initialize attributes shared by all view methods."""
        if hasattr(self, "get") and not hasattr(self, "head"):
            self.head = self.get
        self.request = request
        self.args = args
        self.kwargs = kwargs
```

#### dispatch()

```python
    def dispatch(self, request, *args, **kwargs):
        # Try to dispatch to the right method; if a method doesn't exist,
        # defer to the error handler. Also defer to the error handler if the
        # request method isn't on the approved list.
        if request.method.lower() in self.http_method_names:
            handler = getattr(
                self, request.method.lower(), self.http_method_not_allowed
            )
        else:
            handler = self.http_method_not_allowed
        return handler(request, *args, **kwargs)
```

**dispatch() 方法首先检查当前请求对象的方法是否可执行, 调用与之对应的方法执行.** 

**例如接收到 GET 请求, 则调用视图类内定义的 get() 执行.** 

#### get()

- 调用 `get_queryset()` 方法获取对象列表, 将其赋值给 `self.object_list`
- 调用 `get_context_data()` 方法获取上下文数据
- 调用 `render_to_response()` 方法呈现模板返回响应

#### get_queryset()

- 若定义了 `queryset` , 则调用 `queryset.all()`. 否则若定义了 `model` , 则调用 `self.model._default_manager.all()` 

#### get_context_data()

- 若设置了分页的页面大小, 则自动启用分页, 调用 `self.paginate_queryset()` 以获取分页信息
- 将分页器, 当前页, 对象列表等变量整合为模板上下文.  并调用父类定义的 `get_context_data()` 获取其他上下文信息

#### paginate_queryset()

```python
    def paginate_queryset(self, queryset, page_size):
        """Paginate the queryset, if needed."""
        paginator = self.get_paginator(
            queryset,
            page_size,
            orphans=self.get_paginate_orphans(),
            allow_empty_first_page=self.get_allow_empty(),
        )
        page_kwarg = self.page_kwarg
        page = self.kwargs.get(page_kwarg) or self.request.GET.get(page_kwarg) or 1
        try:
            page_number = int(page)
        except ValueError:
            if page == "last":
                page_number = paginator.num_pages
            else:
                raise Http404(
                    _("Page is not “last”, nor can it be converted to an int.")
                )
        try:
            page = paginator.page(page_number)
            return (paginator, page, page.object_list, page.has_other_pages())
        except InvalidPage as e:
            raise Http404(
                _("Invalid page (%(page_number)s): %(message)s")
                % {"page_number": page_number, "message": str(e)}
            )
```

#### render_to_response()

```python
    def render_to_response(self, context, **response_kwargs):
        """
        Return a response, using the `response_class` for this view, with a
        template rendered with the given context.

        Pass response_kwargs to the constructor of the response class.
        """
        response_kwargs.setdefault("content_type", self.content_type)
        return self.response_class(
            request=self.request,
            template=self.get_template_names(),
            context=context,
            using=self.template_engine,
            **response_kwargs,
        )
```

### DetailView

