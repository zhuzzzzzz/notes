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

在列表页显示字段的设置。若不设置此项，默认展示对象 `__str()__` 方法返回的结果。