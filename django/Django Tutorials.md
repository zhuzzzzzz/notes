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

#### CSRF

Cross Site Requset Forgeries

Django 默认启用了 CSRF 防护机制，保护应用免受 CSRF 攻击。以下是 Django 在 CSRF 防护方面的一些关键点：

1. **中间件启用**：Django 默认启用了 `CsrfViewMiddleware`，它会拦截所有的 POST 请求，检查请求中是否包含有效的 CSRF token。

2. **表单中的 CSRF Token**：Django 的模板系统提供了 `{% csrf_token %}` 标签，可以在每个表单中插入一个隐藏的 CSRF token。

3. **视图装饰器**：对于某些视图，你可以通过装饰器显式地启用或禁用 CSRF 防护。例如：

   ```python
   from django.views.decorators.csrf import csrf_exempt
   
   @csrf_exempt
   def my_view(request):
       # 该视图不启用 CSRF 防护
   ```

4. **Cookie 配置**：Django 会在响应中设置 `CSRF_COOKIE`，并在用户请求中检查 `CSRF_TOKEN`，从而验证请求是否合法。

#### HttpResponseRedirect

应总是在表单的 Post 请求成功后返回一个 Http 重定向。

### Generic Views

Web 开发的通常任务：根据 URL 中传递的参数从数据库中获取数据，加载模板并返回经过呈现后的模板。

#### ListView

- 默认使用 `<app name>/<model name>_list.html`模板

#### DetailView

- 默认使用 `<app name>/<model name>_detail.html`模板

#### Generic Views 获取数据

每一个类视图都需要明确将操作哪一个类的模型数据，或者通过指定 `model` 属性来实现，或者通过定义 `get_queryset()` 类方法来实现

### Automated Testing

#### 创建测试以暴露 bug

将添加测试代码至应用：`polls/test.py` ，测试系统将寻找所有名称以 test 开头的文件。

```python
import datetime

from django.test import TestCase
from django.utils import timezone

from .models import Question


class QuestionModelTests(TestCase):
    def test_was_published_recently_with_future_question(self):
        """
        was_published_recently() returns False for questions whose pub_date
        is in the future.
        """
        time = timezone.now() + datetime.timedelta(days=30)
        future_question = Question(pub_date=time)
        self.assertIs(future_question.was_published_recently(), False)
```

#### 运行测试

```shell
python manage.py test polls
```

将执行如下操作：

1. 寻找 `polls` 应用中的测试文件
2. 在测试文件中寻找 `django.test.Testcase` 的子类
3. 生成测试用数据库
4. 寻找以 `test` 开头的测试方法
5. 执行测试方法

#### 视图测试

`setup_test_environment()`

安装一个模板渲染器，以检查响应中的一些附加变量，例如响应的上下文变量 `response.context`，需要注意的是此方法不会生成一个测试数据库，而是会在当前数据库下进行。

使用 `shell` 进行测试：

```python
from django.test.utils import setup_test_environment
setup_test_environment()
from django.test import Client
# create an instance of the client for our use
client = Client()
```

编写测试文件：

```python
def create_question(question_text, days):
    """
    Create a question with the given `question_text` and published the
    given number of `days` offset to now (negative for questions published
    in the past, positive for questions that have yet to be published).
    """
    time = timezone.now() + datetime.timedelta(days=days)
    return Question.objects.create(question_text=question_text, pub_date=time)


class QuestionIndexViewTests(TestCase):
    def test_no_questions(self):
        """
        If no questions exist, an appropriate message is displayed.
        """
        response = self.client.get(reverse("polls:index"))
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, "No polls are available.")
        self.assertQuerySetEqual(response.context["latest_question_list"], [])

    def test_past_question(self):
        """
        Questions with a pub_date in the past are displayed on the
        index page.
        """
        question = create_question(question_text="Past question.", days=-30)
        response = self.client.get(reverse("polls:index"))
        self.assertQuerySetEqual(
            response.context["latest_question_list"],
            [question],
        )

    def test_future_question(self):
        """
        Questions with a pub_date in the future aren't displayed on
        the index page.
        """
        create_question(question_text="Future question.", days=30)
        response = self.client.get(reverse("polls:index"))
        self.assertContains(response, "No polls are available.")
        self.assertQuerySetEqual(response.context["latest_question_list"], [])

    def test_future_question_and_past_question(self):
        """
        Even if both past and future questions exist, only past questions
        are displayed.
        """
        question = create_question(question_text="Past question.", days=-30)
        create_question(question_text="Future question.", days=30)
        response = self.client.get(reverse("polls:index"))
        self.assertQuerySetEqual(
            response.context["latest_question_list"],
            [question],
        )

    def test_two_past_questions(self):
        """
        The questions index page may display multiple questions.
        """
        question1 = create_question(question_text="Past question 1.", days=-30)
        question2 = create_question(question_text="Past question 2.", days=-5)
        response = self.client.get(reverse("polls:index"))
        self.assertQuerySetEqual(
            response.context["latest_question_list"],
            [question2, question1],
        )
```

#### 测试思想

- 任何要添加进软件的功能，都需要同时添加其测试
- When testing, more is better
- In testing redundancy is a *good* thing
- 根据类或者视图来安排测试类
- 不同的测试条件安排不同的测试方法
- 测试名称直观描述其作用

