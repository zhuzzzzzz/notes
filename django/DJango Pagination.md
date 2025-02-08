### Pagination

Django 为管理分页数据同时提供了上层和底层方法. 

#### `Paginator` 类

*class* Paginator(*object_list*, *per_page*, *orphans=0*, *allow_empty_first_page=True*, *error_messages=None*)[[source\]](https://github.com/django/django/blob/stable/5.1.x/django/core/paginator.py#L27)

分页用到的所有方法都会用到 `Paginator` 类. 这个类负责处理所有将数据集拆分到页的工作. 

当进行求长和迭代时, 分页器的行为就像是一个以页构成的序列. 

#### 类属性

参考[链接](https://docs.djangoproject.com/en/5.1/ref/paginator/#paginator-class).

##### Paginator.object_list

Required. A list, tuple, `QuerySet`, or other sliceable object with a `count()` or `__len__()` method. For consistent pagination, `QuerySet`s should be ordered, e.g. with an [`order_by()`](https://docs.djangoproject.com/en/5.1/ref/models/querysets/#django.db.models.query.QuerySet.order_by) clause or with a default [`ordering`](https://docs.djangoproject.com/en/5.1/ref/models/options/#django.db.models.Options.ordering) on the model.

##### Paginator.per_page

Required. The maximum number of items to include on a page, not including orphans (see the orphans optional argument below).

##### Paginator.orphans

可选设置. 当你不希望最后一页仅有很少几条结果时使用. 默认值为0, 当剩余低于此项设置值的条目时, 这些剩余条目将被添加至最后一页. 

##### Paginator.allow_empty_first_page

可选. 是否允许第一页为空, 当此项设置为 False 且 `object_list` 为空, 将会抛出 `EmptyPage` 错误. 

##### Paginator.error_messages

用以覆盖分页器默认抛出的错误信息. 

#### 类方法

##### Paginator.get_page(number)

##### Paginator.page(number)

##### Paginator.get_elided_page_range(number, *, on_each_side=3, on_ends=2)

#### 其他类属性

##### Paginator.ELLIPSIS

用以表示省略页码的字符串, 默认为 "..."

##### Paginator.count

所有页中对象的总数

##### Paginator.num_pages

总共的页数

##### Paginator.page_range

#### `Page` 类

通常不用自己创建 `Page` 对象, 而是通过迭代分页器或使用 `Paginator.page()` 方法得到此对象. 

*class* Page(*object_list*, *number*, *paginator*)[[source\]](https://github.com/django/django/blob/stable/5.1.x/django/core/paginator.py#L181)

#### 类方法

##### Page.has_next()

##### Page.has_previous()

##### Page.has_other_pages()

##### Page.next_page_number()

##### Page.previos_page_number()

##### Page.start_index()

返回当页的第一个对象下标

##### Page.end_index()

返回当页的最后一个对象下标

#### 类属性

##### Page.object_list

当前页面的对象列表

##### Page.number

当前页面的页面号

##### Page.paginator

当前页对象关联的分页器对象

#### 示例

向分页器类传递一个对象列表, 以及每页的对象个数, 分页器类将提供访问每一页对象的方法. 

```python
>>> from django.core.paginator import Paginator
>>> objects = ["john", "paul", "george", "ringo"]
>>> p = Paginator(objects, 2)

>>> p.count
4
>>> p.num_pages
2
>>> type(p.page_range)
<class 'range_iterator'>
>>> p.page_range
range(1, 3)

>>> page1 = p.page(1)
>>> page1
<Page 1 of 2>
>>> page1.object_list
['john', 'paul']

>>> page2 = p.page(2)
>>> page2.object_list
['george', 'ringo']
>>> page2.has_next()
False
>>> page2.has_previous()
True
>>> page2.has_other_pages()
True
>>> page2.next_page_number()
Traceback (most recent call last):
...
EmptyPage: That page contains no results
>>> page2.previous_page_number()
1
>>> page2.start_index()  # The 1-based index of the first item on this page
3
>>> page2.end_index()  # The 1-based index of the last item on this page
4

>>> p.page(0)
Traceback (most recent call last):
...
EmptyPage: That page number is less than 1
>>> p.page(3)
Traceback (most recent call last):
...
EmptyPage: That page contains no results
```

#### 对 ListView 分页

`django.views.generic.list.ListView` 提供了内置的分页和显示功能. 只需设置视图类的 `paginate_by` 参数的值即可: 

```python
from django.views.generic import ListView

from myapp.models import Contact

class ContactListView(ListView):
    paginate_by = 2
    model = Contact
```

上述例子设置了每页的对象个数, 并会将 `paginator` 及 `page_obj` 对象添加至上下文. 为了在网页上实现分页浏览, 可以按如下编写模板: 

```python
{% for contact in page_obj %}
    {# Each "contact" is a Contact model object. #}
    {{ contact.full_name|upper }}<br>
    ...
{% endfor %}

<div class="pagination">
    <span class="step-links">
        {% if page_obj.has_previous %}
            <a href="?page=1">&laquo; first</a>
            <a href="?page={{ page_obj.previous_page_number }}">previous</a>
        {% endif %}

        <span class="current">
            Page {{ page_obj.number }} of {{ page_obj.paginator.num_pages }}.
        </span>

        {% if page_obj.has_next %}
            <a href="?page={{ page_obj.next_page_number }}">next</a>
            <a href="?page={{ page_obj.paginator.num_pages }}">last &raquo;</a>
        {% endif %}
    </span>
</div>
```

#### 在视图函数中使用 `Paginator`

```python
from django.core.paginator import Paginator
from django.shortcuts import render

from myapp.models import Contact


def listing(request):
    contact_list = Contact.objects.all()
    paginator = Paginator(contact_list, 25)  # Show 25 contacts per page.

    page_number = request.GET.get("page")
    page_obj = paginator.get_page(page_number)
    return render(request, "list.html", {"page_obj": page_obj})
```
