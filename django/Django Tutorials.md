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
- DataTimeField 会获得默认的 Javascript 快捷方式。

### View

#### url 请求处理方式

当有相关请求到达 Django 时，根据设置文件中的 `ROOT_URLCONF` 加载指定 的 url 模块，并在模块中寻找 `urlpatterns` 变量，并根据变量中的定义顺序遍历匹配其中的 url 项目。当匹配到指定的 url 字符时，会将剩余部分传给下一级的 `URLonf` 进一步处理。

#### 视图的原理

每一个视图都必须要实现两件事情：

- 对请求返回一个 `HttpResponse` 对象
- 某些意外情况下触发异常，如 `Http404`

视图可以调用任何  Python 库，做任何事情。例如从数据库中获取数据，使用 Django 的模板系统或第三方模板系统，可以生成 PDF 文件，输出 XML，动态创建压缩包，等等。

#### 模板系统

`TEMLATES` 设置项描述了 Django 如何加载和呈现模板。设置文件中默认使用 `DjangoTemplates` 后端系统。默认情况下该系统将在每一个安装 APP 中寻找 `templates` 目录。

#### 使用 URL 名称命名空间

在应用的 `polls/urls.py` 文件中设置 `app_name` 以设置命名空间。

```python
from django.urls import path

from . import views

app_name = "polls"
urlpatterns = [
    path("", views.index, name="index"),
    path("<int:question_id>/", views.detail, name="detail"),
    path("<int:question_id>/results/", views.results, name="results"),
    path("<int:question_id>/vote/", views.vote, name="vote"),
]
```

### Form

