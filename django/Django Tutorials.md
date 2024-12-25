## Django Tutorials

### 创建项目

[文档链接](https://docs.djangoproject.com/en/5.1/intro/tutorial01/)

```shell
mkdir djangoproject
```

```shell
django-admin startproject mysite djangotutorial
```

``````
djangotutorial/
    manage.py
    mysite/
        __init__.py
        settings.py
        urls.py
        asgi.py
        wsgi.py
``````

- `mysite/asgi.py`: An entry-point for ASGI-compatible web servers to serve your project. See [How to deploy with ASGI](https://docs.djangoproject.com/en/5.1/howto/deployment/asgi/) for more details.
- `mysite/wsgi.py`: An entry-point for WSGI-compatible web servers to serve your project. See [How to deploy with WSGI](https://docs.djangoproject.com/en/5.1/howto/deployment/wsgi/) for more details.

```shel
python manage.py startapp polls
```

#### 使用 include() 包含定义在其他 app 内的 URLconf

``` python
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path("polls/", include("polls.urls")),
    path("admin/", admin.site.urls),
]
```

#### 默认安装的项目 APP 

- [`django.contrib.admin`](https://docs.djangoproject.com/en/5.1/ref/contrib/admin/#module-django.contrib.admin) – The admin site. 管理站点。
- [`django.contrib.auth`](https://docs.djangoproject.com/en/5.1/topics/auth/#module-django.contrib.auth) – An authentication system. 认证系统。
- [`django.contrib.contenttypes`](https://docs.djangoproject.com/en/5.1/ref/contrib/contenttypes/#module-django.contrib.contenttypes) – A framework for content types.  
- [`django.contrib.sessions`](https://docs.djangoproject.com/en/5.1/topics/http/sessions/#module-django.contrib.sessions) – A session framework. 
- [`django.contrib.messages`](https://docs.djangoproject.com/en/5.1/ref/contrib/messages/#module-django.contrib.messages) – A messaging framework. 
- [`django.contrib.staticfiles`](https://docs.djangoproject.com/en/5.1/ref/contrib/staticfiles/#module-django.contrib.staticfiles) – A framework for managing static files.

某些应用将使用至少一张数据库表，因此在使用它们前应在数据库中生成这些表。运行 `migrate` 命令以执行此操作。若不需要使用某些 APP，在执行 `migrate` 命令前应删除相关应用的注册行。

#### 模型与数据库表

- 数据库的表名自动生成格式：`AppName_ModelName`，也可以重写这个行为。
- 数据表中自动增加主键。
- 外键的字段名称将会自动添加 `_id` 后缀。
- `migrate` 命令将执行所有未实施的迁移(django 通过数据库中一个名为 django_migrations 的特殊表来记录哪些迁移已经被执行)。

### Django Admin

[文档链接](https://docs.djangoproject.com/en/5.1/intro/tutorial02/#introducing-the-django-admin)

管理站点不是为访客准备的，而是为站点管理员准备的。管理站点提供了统一的接口给管理员来编辑内容。

#### 为模型注册管理接口

```python
from django.contrib import admin
from .models import Question

admin.site.register(Question)
```

- 访问管理页面出现的编辑表单由 Django 根据数据库模型自动生成而来
- 不同的模型字段类型知道如何将自己展示在 admin 页面中，Django 会自动适应为合适的 HTML 输入控件
- 
