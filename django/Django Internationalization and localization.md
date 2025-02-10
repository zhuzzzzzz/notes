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

### 杂项

#### 设置语言的重定向视图

