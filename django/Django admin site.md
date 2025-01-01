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