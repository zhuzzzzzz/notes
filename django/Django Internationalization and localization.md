## Internationalization and Localization

国际化和本地化的目标是为了让网页应用为不同国家的访问者提供对应语言和格式的内容

Django 提供了关于文本翻译, 日期格式, 时间和数字, 时区的完善支持

总的来说, Django 完成以下两件事情

- 允许开发者定义 app 的哪些部分需要根据使用者的文化及所用的语言进行翻译排版.
- 用户根据他们的偏好来使用这些钩子以将 web app 本土化.

### Overview

#### internationalization

为软件的本土化做准备, 通常由开发者完成

#### localization

编写翻译和本地格式, 通常由翻译者完成

#### locale name

区域名称

#### language code

语言代码

#### message  file

对某种语言的包含所有翻译字符串的纯文本文件. 

#### translation string

翻译字符串

#### format file

为不同区域定义数据格式的 python 模块

## i18n

### translation

为了让 Django 项目可翻译, 必须在 python 代码和模板中为其添加一些钩子. 这些钩子即为翻译字符串(translation strings). 这些钩子会告诉 Django, 当这段文本具有这个语言下的可用翻译时, 这部分的文本应该被翻译为终端用户所使用的语言. 开发者的职责是将这些需要翻译的字符串标记出来, Django 只能翻译那些被标记的且具有对应翻译字符串条目的内容. 

Django 提供了提取翻译字符串到信息文件(message files)的功能. 翻译者可以通过这个文件提供翻译. 当信息文件准备完成后, 它需要被编译, 这部分的工作依赖于 GNU gettext toolset. 

这些都准备完成后, Django 将负责 web app 内容的动态翻译, 根据用户的语言偏好提供对应的翻译文本. 

Django 的 i18n 功能默认开启, 这将产生一部分系统资源开销, 若不需要使用, 需要显示设置 `USE_I18N = False` .

#### 标准翻译

```python
from django.http import HttpResponse
from django.utils.translation import gettext as _

def my_view(request):
    output = _("Welcome to my site.")
    return HttpResponse(output)
```

翻译函数可以接收占位符: 

```python
def my_view(request, m, d):
    output = _("Today is %(month)s %(day)s.") % {"month": m, "day": d}
    return HttpResponse(output)

# This technique lets language-specific translations reorder the placeholder text. For example, an English translation may be "Today is November 26.", while a Spanish translation may be "Hoy es 26 de noviembre." – with the month and the day placeholders swapped.
```

#### 为翻译提供注释

```python
def my_view(request):
    # Translators: This message appears on the home page only
    output = gettext("Welcome to my site.")
```

采用如上方式进行注释, 则注释内容将出现在对应的 `.po` 文件中

```
#. Translators: This message appears on the home page only
# path/to/python/file.py:123
msgid "Welcome to my site."
msgstr ""
```

#### 将字符串标记为 no-op

如果有一些需要以源语言存储的固定字符串（例如存储在数据库中的字符串，并可能在不同系统或用户之间传输），但希望在最终展示给用户时才进行翻译，就可以使用 `gettext_noop()`。

#### 复数化

```python
from django.http import HttpResponse
from django.utils.translation import ngettext


def hello_world(request, count):
    page = ngettext(
        "there is %(count)d object",
        "there are %(count)d objects",
        count,
    ) % {
        "count": count,
    }
    return HttpResponse(page)
```

#### 上下文标记

```python
from django.utils.translation import pgettext

month = pgettext("month name", "May")

# will appear in the .po file as:
msgctxt "month name"
msgid "May"
msgstr ""
```

#### 懒翻译

当字符串被用作字符内容时才进行翻译. 

#### 使用懒翻译

`gettext_lazy()` 调用的结果可以被用在 Django 项目中任何需要字符串的地方, 但不是所有 python 代码都支持. 比如当在 `request` 库中无法处理 `gettext_lazy` 对象: 

```python
body = gettext_lazy("I \u2764 Django")  # (Unicode :heart:)
requests.post("https://example.com/send", data={"body": body})

# 在使用前将其显示转化为字符串
requests.post("https://example.com/send", data={"body": str(body)})
```

#### 在模板中使用 i18n



#### 在 JavaScript 代码中使用 i18n



#### 在 urlpattern 中使用 i18n

`i18n_patterns(*urls, prefix_default_language=True)`

在根 URLconf 中使用此函数, Django 将自动为使用此方法定义的 URL 模式前添加当前的激活语言代码

设置 `prefix_default_language=False` 将移除激活语言为默认语言(即 LANGUAGE_CODE 中定义的语言)下的 url 前缀. 

```python
from django.conf.urls.i18n import i18n_patterns
from django.urls import include, path

from about import views as about_views
from news import views as news_views
from sitemap.views import sitemap

urlpatterns = [
    path("sitemap.xml", sitemap, name="sitemap-xml"),
]

news_patterns = (
    [
        path("", news_views.index, name="index"),
        path("category/<slug:slug>/", news_views.category, name="category"),
        path("<slug:slug>/", news_views.details, name="detail"),
    ],
    "news",
)

urlpatterns += i18n_patterns(
    path("about/", about_views.main, name="about"),
    path("news/", include(news_patterns, namespace="news")),
)


## After defining these URL patterns, Django will automatically add the language prefix to the URL patterns that were added by the i18n_patterns function. Example:
>>> from django.urls import reverse
>>> from django.utils.translation import activate

>>> activate("en")
>>> reverse("sitemap-xml")
'/sitemap.xml'
>>> reverse("news:index")
'/en/news/'

>>> activate("nl")
>>> reverse("news:detail", kwargs={"slug": "news-slug"})
'/nl/news/news-slug/'


# With prefix_default_language=False and LANGUAGE_CODE='en', the URLs will be:
>>> activate("en")
>>> reverse("news:index")
'/news/'

>>> activate("nl")
>>> reverse("news:index")
'/nl/news/'
```

#### 翻译 url

```python
from django.conf.urls.i18n import i18n_patterns
from django.urls import include, path
from django.utils.translation import gettext_lazy as _

from about import views as about_views
from news import views as news_views
from sitemaps.views import sitemap

urlpatterns = [
    path("sitemap.xml", sitemap, name="sitemap-xml"),
]

news_patterns = (
    [
        path("", news_views.index, name="index"),
        path(_("category/<slug:slug>/"), news_views.category, name="category"),
        path("<slug:slug>/", news_views.details, name="detail"),
    ],
    "news",
)

urlpatterns += i18n_patterns(
    path(_("about/"), about_views.main, name="about"),
    path(_("news/"), include(news_patterns, namespace="news")),
)


###
>>> from django.urls import reverse
>>> from django.utils.translation import activate

>>> activate("en")
>>> reverse("news:category", kwargs={"slug": "recent"})
'/en/news/category/recent/'

>>> activate("nl")
>>> reverse("news:category", kwargs={"slug": "recent"})
'/nl/nieuws/categorie/recent/'
```

**大多数情况下, 最好只使用前一种情况, 以避免翻译导致的 url 冲突问题.**

#### 在模板中重定向到其他语言的 url

```python
{% load i18n %}

{% get_available_languages as languages %}

{% translate "View this category in:" %}
{% for lang_code, lang_name in languages %}
    {% language lang_code %}
    <a href="{% url 'category' slug=category.slug %}">{{ lang_name }}</a>
    {% endlanguage %}
{% endfor %}
```

### 如何创建语言文件

一旦应用的文本被标记需要翻译, 那么就需要编写对应的翻译条目

#### Message files

- 纯文本文件
- 一个文件代表一种语言
- 包含所有翻译条目
- 以 `.po` 为后缀

##### 创建 message file

```shell
django-admin makemessages -l de

# To reexamine all source code and templates for new translation strings and update all message files for all languages, run this:
django-admin makemessages -a
```

其中 `de` 为 locale name, For example, `pt_BR` for Brazilian Portuguese, `de_AT` for Austrian German or `id` for Indonesian.

此命令应在如下两个地方之一执行:

- Django 项目的根目录(包含 manage.py 的目录)
- Django app 的根目录

**执行此命令后, 会将项目目录内或app目录内所有被标记需要翻译的字符串提取出来, 在 `/locale/LANG/LC_MESSAGES` 目录下创建或更新 message file, 需要确保 `LOCALE_PATH` 正确配置. 以 `de` 为例, 文件将被创建在 `locale/de/LC_MESSAGES/django.po`**

默认情况下 `django-admin makemessages` 会检查具有 `.html` `.txt` `.py` 后缀的所有文件. 如果你想要覆盖这个默认设置, 可以使用 ``--extension` 或 `-e` 选项来指定要检查的文件后缀: 

```shell
django-admin makemessages -l de -e txt

# Separate multiple extensions with commas and/or use -e or --extension multiple times:
django-admin makemessages -l de -e html,txt -e xml
```

##### message file 的结构

```python
_("Welcome to my site.")

# …then django-admin makemessages will have created a .po file containing the following snippet – a message:

#: path/to/python/module.py:23
msgid "Welcome to my site."
msgstr ""
```

- `msgid` 为需要翻译的字符串, 出现在源代码中, 不要改动它
- `msgstr` 需要填入翻译后的字符串, 确保其由引号包裹

- 对于过长的字符串, 可以保持第一对引号为空, 后续使用多行引号包裹的字符串进行拼接.

##### 编译 message file

在创建 message file 后, 每次修改它时, 都需要将其重新编译, 以变成 `gettext` 使用的更高效的格式. 

```shell
django-admin compilemessages
```

执行命令为所有 `.po` 文件生成 `.mo` 文件, 后者为 `gettext` 所使用的二进制文件. 

##### 注释 % 号 

参考[链接](https://docs.djangoproject.com/en/5.1/topics/i18n/translation/#troubleshooting-gettext-incorrectly-detects-python-format-in-strings-with-percent-signs)

##### 为 JavaScript 源代码创建 message files

```shell
django-admin makemessages -d djangojs -l de
```

### 其他杂项

#### 设置语言偏好的重定向视图

`django.views.i18n.set_language()` 

`set_language(request)` 视图可以设置用户的语言偏好, 并重定向至指定 URL, 默认情况下会返回先前的页面, 通过如下方式启用:

```python
# adding the following line to your URLconf:
path("i18n/", include("django.conf.urls.i18n")),
```

这个视图需要通过 POST 方法调用, 并在请求中附带 `language` 参数. 若会话支持启用, 视图将在用户的会话中保存语言选择, 并会将语言选择保存在名为 `django_language` 的 cookie 中(名称可通过 `LANGUAGE_COOKIE_NAME` 修改). 

在设置语言选择后, Django 在请求的数据中寻找 `next` 参数, 若找到则将其视作安全 URL (不指向其他主机, 并使用安全机制)并执行重定向. 若未找到, Django 会将用户重定向至 `Referer` 请求头, 若该请求头也未设置, 会将用户重定向至主页. 但具体还取决于用户的请求属性:

- 若请求接收 HTML 内容, 则会依次执行以上策略
- 若请求不接受 HTMYou may want to set the active language for the current session explicitly. Perhaps a user’s language preference is retrieved from another system, for example. You’ve already been introduced to [`django.utils.translation.activate()`](https://docs.djangoproject.com/en/5.1/ref/utils/#django.utils.translation.activate). That applies to the current thread only. To persist the language for the entire session in a cookie, set the [`LANGUAGE_COOKIE_NAME`](https://docs.djangoproject.com/en/5.1/ref/settings/#std-setting-LANGUAGE_COOKIE_NAME) cookie on the response:L 内容, 则仅执行到 `next` 这一步, 否则将返回 204 状态码(No Content)

如下模板可设置使用 `set_language` 视图: 

```python
{% load i18n %}

<form action="{% url 'set_language' %}" method="post">{% csrf_token %}
    <input name="next" type="hidden" value="{{ redirect_to }}">
    <select name="language">
        {% get_current_language as LANGUAGE_CODE %}
        {% get_available_languages as LANGUAGES %}
        {% get_language_info_list for LANGUAGES as languages %}
        {% for language in languages %}
            <option value="{{ language.code }}"{% if language.code == LANGUAGE_CODE %} selected{% endif %}>
                {{ language.name_local }} ({{ language.code }})
            </option>
        {% endfor %}
    </select>
    <input type="submit" value="Go">
</form>
```

#### 显式设置当前语言

`django.utils.translation.activate()` 仅会在当前线程应用语言偏好设置, 要将会话中的语言设置持久化, 应设置响应的 cookie : `LANGUAGE_COOKIE_NAME` 

```python
from django.conf import settings
from django.http import HttpResponse
from django.utils import translation

user_language = "fr"
translation.activate(user_language)
response = HttpResponse(...)
response.set_cookie(settings.LANGUAGE_COOKIE_NAME, user_language)
```

通常情况下使用二者, 在当前线程改变语言设置, 并将其设置在 cookie 中以将设置持续化应用至后续的请求上. 

#### 在视图和模板之外使用翻译

尽管 Django 提供了视图和模板中关于 i18n 的丰富工具集, 但它们的用途不局限在 Django 框架中. Django 的翻译机制可以用于翻译 Django 支持的任意语言的任意文本. 你可以加载一个翻译类, 然后选择想要的翻译语言激活去翻译任意文本, 但别忘了结束翻译时恢复原来的语言, 翻译是基于每个线程单独进行的, 当前线程下的翻译可能会受到影响. 

```python
from django.utils import translation

def welcome_translated(language):
    cur_language = translation.get_language()
    try:
        translation.activate(language)
        text = translation.gettext("welcome")
    finally:
        translation.activate(cur_language)
    return text
```

为了自动帮助恢复语言, 还可以基于 with 调用翻译, 这将在退出时自动回复调用时的语言设置: 

```python
from django.utils import translation

def welcome_translated(language):
    with translation.override(language):
        return translation.gettext("welcome")
```

#### 翻译相关的 cookie 设置

##### [LANGUAGE_COOKIE_NAME](https://docs.djangoproject.com/en/5.1/ref/settings/#std-setting-LANGUAGE_COOKIE_NAME)

##### [LANGUAGE_COOKIE_AGE](https://docs.djangoproject.com/en/5.1/ref/settings/#std-setting-LANGUAGE_COOKIE_AGE)

##### [LANGUAGE_COOKIE_DOMAIN](https://docs.djangoproject.com/en/5.1/ref/settings/#std-setting-LANGUAGE_COOKIE_DOMAIN)

##### [LANGUAGE_COOKIE_HTTPONLY](https://docs.djangoproject.com/en/5.1/ref/settings/#std-setting-LANGUAGE_COOKIE_HTTPONLY)

##### [LANGUAGE_COOKIE_PATH](https://docs.djangoproject.com/en/5.1/ref/settings/#std-setting-LANGUAGE_COOKIE_PATH)

##### [LANGUAGE_COOKIE_SAMESITE](https://docs.djangoproject.com/en/5.1/ref/settings/#std-setting-LANGUAGE_COOKIE_SAMESITE)

##### [LANGUAGE_COOKIE_SECURE](https://docs.djangoproject.com/en/5.1/ref/settings/#std-setting-LANGUAGE_COOKIE_SECURE)

### 应用说明

#### Django 翻译与 gettext

#### Django 如何发现语言偏好设置

应用级的语言偏好设置: `LANGUAGE_CODE` . 若想在母语下使用 Django , 仅需设置此变量并确保对应语言的 message file和相关编译文件存在即可. Django 使用此设置作为默认翻译语言: 当没有更好的翻译匹配选择时作为最后的默认语言. 

若要让不同语言的用户可以使用他们自己的语言, 需要使用 `LocaleMiddleware.LocaleMiddleware` 中间件, 以启用基于请求数据的语言选择功能. 中间件设置顺序: 

- 确保这是最先被安装的中间件之一
- 需要在 `SeesionMiddleware` 中间件之后, 因为 `LocaleMiddleware` 会使用会话数据. 
- 需要在 `CommonMiddleware` 中间件之前, 因为后者需要激活的语言设置来解析请求的 URL
- 若使用了 `CacheMiddleware` , 则需在其之后

关于更多, 参考[中间件文档](https://docs.djangoproject.com/en/5.1/topics/http/middleware/)

`LocaleMiddleware` 以如下算法来决定用户的语言偏好:

1. 查看请求 URL 的语言前缀. 这仅会在 root URLconf 使用了 `i18n_pattern` 功能的情况下生效. 
2. 根据 cookie 中 `LANGUAGE_COOKIE_NAME` 的设置决定
3. 根据 HTTP 头部 `Accept-Language` 设置决定. 
4. 使用全局设置 `LANGUAGE_CODE`

注:

- 语言偏好必须是标准的格式

- 若只提供了 Django 基础语言的翻译而未提供子语言的翻译, 则使用基础语言.  For example, if a user specifies `de-at` (Austrian German) but Django only has `de` available, Django uses `de`.

- 只有在 `LANGUAGE` 设置中列出的语言才可以使用. 

  ```python
  LANGUAGES = [
      ("de", _("German")),
      ("en", _("English")),
  ]
  ```

- 一旦中间件决定了用户的语言偏好, 则会在请求中设置为 `request.LANGUAGE_CODE` 变量, 可以在视图代码中使用此变量

  ```python
  from django.http import HttpResponse
  
  def hello_world(request, count):
      if request.LANGUAGE_CODE == "de-at":
          return HttpResponse("You prefer to read Austrian German.")
      else:
          return HttpResponse("You prefer to read another language.")
  ```

- 在没有中间件的情况下, 仅使用 `settings.LANGUAGE_CODE`设置, 在启用中间件的情况, 则使用 `request.LANGUAGE` . 注意区别

#### Django 如何发现翻译条目

运行时 Django 将在内存中构建统一的翻译条目. 将通过以下算法确定翻译条目的顺序和优先级: 

1. 在 `LOCALE_PATHS` 中定义的目录具有最高的优先级
2. 根据 `INSTALLED_APP` 设置, 在已安装的每个 APP 中依次寻找 `locale` 目录. 
3. 使用 Django 提供的基础翻译 `django/conf/locale` 

#### Django 项目使用英语作为默认语言



## l10n

当格式化被启用时, Django 可以使用本地化的格式来解析表单中的数据, 时间和数字. 

要使某个表单字段本地化输入输出数据, 可以使用其 `localize` 参数:

```python
class CashRegisterForm(forms.Form):
    product = forms.CharField()
    revenue = forms.DecimalField(max_digits=4, decimal_places=2, localize=True)
```

### 在模板中控制本地化

#### Template tags

##### localize

在包含的块中启用或禁用模板变量的本地化. 

```python
{% load l10n %}

{% localize on %}
    {{ value }}
{% endlocalize %}

{% localize off %}
    {{ value }}
{% endlocalize %}
```

当本地化被禁用时, 关于 localization 的设置被默认应用. 详见[链接](https://docs.djangoproject.com/en/5.1/ref/settings/#settings-l10n)

#### Template filters

##### localize

对某个变量进行本地化.

```python
{% load l10n %}

{{ value|localize }}
```

##### unlocalize

对某个变量取消本地化. 对数字会返回其字符串表示形式. 

```python
{% load l10n %}

{{ value|unlocalize }}
```

#### 创建自定义的格式文件

Django 提供了很多地区的格式定义, 但有时也可能需要创建自定义的格式. 

要使用自定义格式, 需要首先指定格式文件的路径: 

```python
FORMAT_MODULE_PATH = [
    "mysite.formats",
    "some_app.formats",
]
```

文件不似乎直接放在目录内的, 而是放在由地区名称命名的目录内, 并且是作为 python 模块文件: 

```python
mysite/
    formats/
        __init__.py
        en/
            __init__.py
            formats.py
            
## where formats.py contains custom format definitions. For example:
THOUSAND_SEPARATOR = "\xa0"
```

#### 关于本地化格式的一些限制

有些地区使用上下文相关的数字格式, Django 的本地化系统无法自动处理. 

## Time zones

当时区相关的支持被启用, Django 将时间信息以 UTC 的形式存储在数据库中, 并在系统内部使用时区相关的时间日期对象, 并根据终端用户的时区在模板中和表单中进行时区转换. 

即使你的网站只在一个时区内提供服务, 将数据用 UTC 形式存储在数据库中也是一个很好的做饭. 理由是因为 DST (daylight saving time). 大多数国家都有 DST 系统, 在春天时钟会被调快而冬天会被调慢. 当你使用本地时间运行项目, 你可能会在时间调整时, 每两年遇到一些错误. 这对博客系统没有影响, 但如果你要通过时长向用户收费, 这可能会产生问题. 解决此问题的方法就是在代码中使用 UTC , 并让终端用户基于此使用他们的地区时间. 

时区支持默认开启, 设置 `USE_TZ=False` 来关闭他. (这是 Django 5.0 的特性, 更早的版本默认关闭.)

### 一些概念

#### Naive and aware datetime objects

Python 的 `datetime.datetime` 对象有一个 `tzinfo` 属性, 可以被用来存储时区信息, 以 `datetime.tzinfo` 子类的实例形式表示. 当这个属性被设置并且表述了一个偏移, 那么 `datetime` 对象就是 aware 状态, 否则是 naive 状态. 

可以使用 `is_aware()` 和 `is_naive()` 方法来确定状态. 

当时区支持被禁用, Django 使用 naive 状态的 `datetime` 对象作为本地时间. 这在大多数情况下够用了, 在这种情况下, 要获取当前时间: 

```python
import datetime

now = datetime.datetime.now()
```

当时区支持被启用, Django 使用 aware 状态的 `datetime` 对象. 若你的代码要创建 `datetime` 对象, 他们也应该是 aware 状态的. 在这种情况下, 要获取当前时间, 按如下操作: 

```python
from django.utils import timezone

now = timezone.now()
```

#### 默认时区和当前时区

- 默认时区指由 `TIME_ZONE` 设置定义的时区
- 当前时区指将要被呈现给用户的时区

通过 `activate()` 方法来将终端用户的实际时区设置为当前时区. 否则, 将会使用默认时区. 

#### 选择当前时区

在会话中保存时区信息: 

```python
# Add the following middleware to MIDDLEWARE:
import zoneinfo
from django.utils import timezone

class TimezoneMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        tzname = request.session.get("django_timezone")
        if tzname:
            timezone.activate(zoneinfo.ZoneInfo(tzname))
        else:
            timezone.deactivate()
        return self.get_response(request)
```

创建设置当前时区的视图及相关模板:

```python
from django.shortcuts import redirect, render

# Prepare a map of common locations to timezone choices you wish to offer.
common_timezones = {
    "London": "Europe/London",
    "Paris": "Europe/Paris",
    "New York": "America/New_York",
}

def set_timezone(request):
    if request.method == "POST":
        request.session["django_timezone"] = request.POST["timezone"]
        return redirect("/")
    else:
        return render(request, "template.html", {"timezones": common_timezones})
```

```python
{% load tz %}
{% get_current_timezone as TIME_ZONE %}
<form action="{% url 'set_timezone' %}" method="POST">
    {% csrf_token %}
    <label for="timezone">Time zone:</label>
    <select name="timezone">
        {% for city, tz in timezones %}
        <option value="{{ tz }}"{% if tz == TIME_ZONE %} selected{% endif %}>{{ city }}</option>
        {% endfor %}
    </select>
    <input type="submit" value="Set">
</form>
```

### 在表单中确定时区

当时区支持被启用, Django 会以当前时区自动解析表单中输入的时间日期, 并在 `cleaned_data` 中返回一个 aware 状态的 `datetime` 对象

### 在输出模板中确定时区

#### Template tags

##### localtime

在包含的块内, 启用或禁用对 aware 状态下 datetime 对象到当前时区的转换

```python
{% load tz %}

{% localtime on %}
    {{ value }}
{% endlocaltime %}

{% localtime off %}
    {{ value }}
{% endlocaltime %}
```

##### timezone

在包含的块内设置或取消当前时区. 

```python
{% load tz %}

{% timezone "Europe/Paris" %}
    Paris time: {{ value }}
{% endtimezone %}

{% timezone None %}
    Server time: {{ value }}
{% endtimezone %}
```

##### get_current_timezone

获取当前时区

```python
{% get_current_timezone as TIME_ZONE %}
```

除了这种方法外, 你也可以激活 `tz()` 的上下文处理器, 使用其返回的 `TIME_ZONE` 上下文变量. 

#### Template filters

这些过滤器接收 aware 和 naive 状态的时间. 默认将 naive 状态的时间使用本地时区考虑. 总是返回 aware 状态下的时间. 

##### localtime

将当前值转化到当前时区

```python
{% load tz %}

{{ value|localtime }}
```

##### utc

将当前值转化到 UTC

```python
{% load tz %}

{{ value|utc }}
```

##### timezone

将当前值转化到任意时区, 参数必须是 tzinfo 的子类实例或是时区名称

```python
{% load tz %}

{{ value|timezone:"Europe/Paris" }}
```

### 数据库迁移帮助

帮助将 Django 支持时区之前的项目迁移到当前版本. 

