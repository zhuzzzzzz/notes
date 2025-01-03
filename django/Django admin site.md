## Django admin site

Django 的一个强大特性就是它的自动管理接口。通过从模型中读取元数据，提供一个快速的、以模型为中心的接口。通过这个接口，受信任的用户可以管理站点内容。管理功能推荐作为仅限于内部组织的管理工具使用，这项功能的开发目的不是为构建整个前端。

本篇关于如何激活、使用及自定义 Django 的 admin 管理接口

[文档链接](https://docs.djangoproject.com/en/5.1/ref/contrib/admin/)

### Overview

#### 如何使用 admin

- 默认情况下 admin 功能在使用 `startproject` 创建项目时默认启用

- 若未使用默认的项目模板，参考[链接](https://docs.djangoproject.com/en/5.1/ref/contrib/admin/#overview)
- 默认情况下，登录到管理站点的用户需要将其 `is_staff` 属性设置为 `True`
- 确定哪些应用模型需要通过管理站点进行编辑，并将其注册到 admin

### ModelAdmin 对象

#### ModelAdmin 类

通过继承方式自定义 ModelAdmin 类并注册

``` python
from django.contrib import admin
from myapp.models import Author

class AuthorAdmin(admin.ModelAdmin):
    pass

admin.site.register(Author, AuthorAdmin)
```

使用默认的 ModelAdmin类并注册

```python
from django.contrib import admin
from myapp.models import Author

admin.site.register(Author)
```

#### 使用装饰器注册

```python
from django.contrib import admin
from .models import Author

@admin.register(Author)
class AuthorAdmin(admin.ModelAdmin):
    pass
```

也可以同时为多个模型类注册，如果需要使用自定义的 AdminSite，将其作为 `site` 参数传递、

```python
from django.contrib import admin
from .models import Author, Editor, Reader
from myproject.admin_site import custom_admin_site

@admin.register(Author, Reader, Editor, site=custom_admin_site)
class PersonAdmin(admin.ModelAdmin):
    pass
```

使用装饰器后，应使用 `super().__init__(*args, **kwargs)` 方法来调用父类的初始化函数，不应使用  `super(PersonAdmin, self).__init__(*args, **kwargs)` 来调用，这是因为装饰器可能会改变类的一些上下文属性。

#### ModelAdmin 支持的选项

##### ModelAdmin.actions

在修改页面提供的一系列操作的列表，详见[Admin actions](https://docs.djangoproject.com/en/5.1/ref/contrib/admin/actions/)

##### ModelAdmin.actions_on_top

##### ModelAdmin.actions_on_bottom

##### ModelAdmin.actions_selection_counter

控制是否在动作按钮附近显示一个选择计数器。默认 `actions_selection_counter = True`

##### ModelAdmin.date_hierarchy

设置此字段为模型中的某个 `DateField` 或  `DateTimeField`，以启用基于该日期字段的时间导航。

```python
date_hierarchy = "pub_date"

# You can also specify a field on a related model using the __ lookup
date_hierarchy = "author__pub_date"
```

##### ModelAdmin.empty_value_display

设置空字段(None，空字符串等)的默认显示值。默认值是 `-` 。

```python
from django.contrib import admin

class AuthorAdmin(admin.ModelAdmin):
    empty_value_display = "-empty-"
```

为某些字段设置：

```python
from django.contrib import admin

class AuthorAdmin(admin.ModelAdmin):
    list_display = ["name", "title", "view_birth_date"]

    @admin.display(empty_value="???")
    def view_birth_date(self, obj):
        return obj.birth_date
```

##### ModelAdmin.exclude

设置表单中将除外哪些字段

##### ModelAdmin.fields

设置表单中显示哪些字段，控制字段的显示顺序，也可以对其进行分组。`fields` 字段可以包含 `readonly_fields` 中设置的字段作为只读显示

`fields` 字段与 `list_display` 字段类似，区别在于 `fields` 字段不接受可调用对象和基于外键的字段查询( `__` )

将多个字段整合到一个元组中以实现在单行中显示多个字段：

```python
class FlatPageAdmin(admin.ModelAdmin):
    fields = [("url", "title"), "content"]
```

若未设置 `fields` 或 `fieldsets`，Django 将自动根据定义顺序展示非 `AutoField` 及 `editable=True` 的字段

##### ModelAdmin.fieldsets

通过该字段控制 admin "add" 和 "change" 页面的布局。

`fieldsets` 是一个二元素元组的列表，每一个二元素元组代表 admin 表单页的一个字段集合，格式：`(name, field_options)` 

```python
from django.contrib import admin

class FlatPageAdmin(admin.ModelAdmin):
    fieldsets = [
        (
            None,
            {
                "fields": ["url", "title", "content", "sites"],
            },
        ),
        (
            "Advanced options",
            {
                "classes": ["collapse"],
                "fields": ["registration_required", "template_name"],
            },
        ),
    ]
```

##### ModelAdmin.filter_horizontal

##### ModelAdmin.filter_vertical

##### ModelAdmin.form

提供自定义的 ModelForm 来重写 add/change 页面默认的表单行为，可通过 `ModelAdmin.get_form()` 来获取默认表单，在此基础上自定义表单，而不是从头开始写一个新的

##### ModelAdmin.formfield_overrides

快速覆盖某些字段的配置属性

##### ModelAdmin.inlines

##### ModelAdmin.list_display

参考[链接](https://docs.djangoproject.com/en/5.1/ref/contrib/admin/#django.contrib.admin.ModelAdmin.list_display)

在列表页显示字段的设置。若不设置此项，默认展示对象 `__str()__` 方法返回的结果。

##### ModelAdmin.list_display_links

使用这个字段控制 `list_display` 中哪些字段可以直接链接到对象的 `change` 页面。默认情况下，自动将 `list_display` 中的第一列链接。将此字段设置为 `None` 将不链接任何字段。

##### ModelAdmin.list_editable

设置可以直接在 `change list` 页面进行更改的字段。所设置的字段必须要在 `list_display` 中已设置且不能在 `list_display_links` 已设置

##### ModelAdmin.list_filter

设置一个 `list_filter` 来激活更改列表页面的右侧侧边栏的过滤器

最简单的应用是设置需要过滤字段的列表或元组，关于更高级的功能选项配置，参考[链接](https://docs.djangoproject.com/en/5.1/ref/contrib/admin/filters/#modeladmin-list-filters)

##### ModelAdmin.list_max_show_all

设置一个值，当 admin 更改列表页将要展示的实际对象个数小于这个值时，将提供一个展示全部对象的链接，点击这个链接，将无视分页情况在当前页展示所有对象。

##### ModelAdmin.list_per_page

设置在更改列表页每页将展示的对象个数

##### ModelAdmin.list_selected_related

设置此项以控制 Django 使用外键相关的查询 `select_related()` , 以节约数据库的查询时间

可以设置为 `Ture` , `False` , `list` , `tuple` . 

- 当设置为 `True` 时, `select_related()` 总会被调用
- 当设置为 `False` 时, Django 只会在 `list_display()` 中包含外键时调用 `select_related()`
- 当设置为元组或列表, 将直接作为 `select_related` 的参数

```python
class ArticleAdmin(admin.ModelAdmin):
    list_select_related = ["author", "category"]
```

will call `select_related('author', 'category')`.

##### Model Admin.ordering

设置管理页面的对象列表排序, 同对象模型中的 `ordering` 参数

##### Model Admin.paginator

分页器

##### ModelAdmin.prepopulated_fields

##### ModelAdmin.preserve_filters

默认情况下, 已经应用的过滤器在创建, 编辑, 删除一个对象后会仍然保留, 如果在这些操作后要清空过滤设置, 设置此项为 `False` 

##### ModelAdmin.show_facets

控制是否在过滤器面板上显示记数

启用此项将会增加管理更改列表页面的数据库查询量, 与过滤器的数量成比例关系, 当数据库较大时, 会有一定程度的性能影响

##### ModelAdmin.radio_fields

设置哪些字段在更改页面使用单选框, 只有外键字段和选项字段可以设置

##### ModelAdmin.autocomplete_fields

关于外键字段和多对多关系字段的自动补全. 当涉及到的数据数量过多时, 通过 Django 提供的选择框去选择目标对象是十分麻烦的, 此时可以设置自动补全. 

- 使用此字段必须要设置 `search_fields` 字段, 自动补全将利用这个进行搜索. 
- 为了避免数据泄露, 用户必须要有相关对象在管理站的的 view 或 change 权限
- 自动补全的排序和分页结果由相关对象的 `get_ordering` 和 `get_paginator` 方法控制
- 同前 `ordering` 字段, 此字段引发的排序在数据库数量较大时也会产生性能问题, 这种情况下可以重写 `Model Admin.get_search_results()` 方法来实现基于文本排序的搜索

##### ModelAdmin.raw_id_fields

关于外键字段和多对多关系字段的主键查找. 当涉及到的数据数量过多时, 通过 Django 提供的选择框去选择目标对象是十分麻烦的, 此时可以设置此项, 使用对应的主键值设置字段. 

##### ModelAdmin.readonly_fields

设置只读字段

##### ModelAdmin.save_as

设置 `save_as = True` 后, `Save and add another` 按钮将会被替换为 `Save as new` 

##### ModelAdmin.save_as_continue

设置 `save_as = True` 后, 对象保存后的重定向默认为到新创建的对象, 设置此项为 `False` 则重定向至更改列表页

##### ModelAdmin.save_on_top

##### ModelAdmin.search_fields

参考[链接](https://docs.djangoproject.com/en/5.1/ref/contrib/admin/#django.contrib.admin.ModelAdmin.search_fields)

设置此项激活管理更改列表页面上方的搜索框. 这些设置的字段必须是某种形式的文本字段, 例如 `CharField` 或 `TextField`

Django 会将输入的字符串拆分为单词并返回所有包含每个单词的搜索结果, 例如当设置 `search_fields = ['fisrt_name', 'last_name']` 时, 当用户搜索 `jhon lennon`, Django 将执行以下 SQL 语句

```sql
WHERE (first_name ILIKE '%john%' OR last_name ILIKE '%john%')
AND (first_name ILIKE '%lennon%' OR last_name ILIKE '%lennon%')
```

当用户搜索 `'jhon lennon'` 时, 则执行以下 SQL 语句

```sql
WHERE (first_name ILIKE '%john winston%' OR last_name ILIKE '%john winston%')
```

##### Model Admin.search_help_text

设置此项以显示搜索框帮助

##### ModelAdmin.show_full_result_count

此项默认设置为 `True` , 以显示一个过滤后结果类似于 `99 results (103 total)` 

##### ModelAdmin.sortable_by

设置可以通过哪些字段来执行排序

##### ModelAdmin.view_on_site

设置此项将在对象的更改页面上展示 `在站点上查看` 按钮, 将链接到用户能够访问到的站点页面. 默认此项设置为 `True` 

也可将此项设置为可调用函数, 此时这个可调用函数会接受模型类实例作为输入参数, 你需要在管理模型中定义这个方法以返回某个链接: 

```python
from django.contrib import admin
from django.urls import reverse

class PersonAdmin(admin.ModelAdmin):
    def view_on_site(self, obj):
        url = reverse("person-detail", kwargs={"slug": obj.slug})
        return "https://example.com" + url
```