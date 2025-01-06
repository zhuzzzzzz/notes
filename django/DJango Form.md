## Django Form

表单用来接受用户的输入数据, Django 提供了一系列工具和库来帮助你构建表单, 用来从站点访问者处接收数据, 处理并返回响应

#### HTML 表单

在 HTML 中, 表单即为 `<form>...</form>` 中元素的集合, 允许访问者输入文, 选择选项, 操作对象和控件, 等等, 然后表单会把信息发送回服务器. 

##### GET 和 POST

任何有可能改变系统状态的请求, 例如会更改数据库的请求, 都应该使用 POST 方法.  GET 方法只适合哪些不影响系统状态的请求. 

GET 也不应用来处理需要加密的请求. 

### Django 如何处理表单

- 在呈现表单时, 负责准备和重构数据
- 创建 HTML 表单
- 接收并处理客户端提交的表单和数据

#### Form 类

如同 Django 中模型类定义表述对象的数据结构, 行为和呈现方式一样, 表单类描述了表单并决定了它如何运作和呈现. 如同模型类的字段与数据库字段相映射一样, 表单类的字段也会与 HTML 表单中 `<input>` 元素映射. 表单字段将作为 HTML 控件呈现给用户的浏览器, 每一种字段类型都有默认的控件类, 当然也可以根据需要进行修改. 

ModelForm 通过 Form 将模型类字段映射到 HTML 表单中 `<input>` 元素, Django Admin 是基于此实现的. 

#### 初始化, 处理和呈现表单

当需要 Django 呈现某个数据对象, 通常进行如下操作

- 在视图中获取数据(数据库查找)
- 将数据作为模板上下文参数传递
- 在构建 HTML 时使用模板变量呈现数据

#### 构建表单

##### 需要做的工作

- 需要构建如下表单, 这个表单会将数据发回至 action 字段定义的 `/your-name/` url 地址. 并提供标签, 输入框和提交按钮. 如果模板的上下文包含 `current_name`  变量, 则会预填充这个值至文本输入框. 

```html
<form action="/your-name/" method="post">
    <label for="your_name">Your name: </label>
    <input id="your_name" type="text" name="your_name" value="{{ current_name }}">
    <input type="submit" value="OK">
</form>
```

- 为实现上述表单, 我们需要构建一个视图来呈现包含此表单的模板, 并能够在模板上下文中提供 `current_name` 变量. 视图需要处理 POST 请求, 并映射至 `/your-name/` url 地址. 

##### 在 Django 中构建表单

编写如下 python 代码定义表单类, `label=` 变量将会呈现为 `<label>` 控件. `max_length=` 定义的数据长度值将在浏览器端校验, 并且当 Django 接收到数据口, 也将进行校验.

```python
from django import forms

class NameForm(forms.Form):
    your_name = forms.CharField(label="Your name", max_length=100)
```

以上表单将会被呈现为以下 HTML 代码: 

```html
<label for="your_name">Your name: </label>
<input id="your_name" type="text" name="your_name" maxlength="100" required>
```

其中并不包含 <form> 标签及提交按钮, 因此需要在模板中自行提供这些元素. 

##### 表单的 `is_valid()` 方法

执行此方法, 若所有的表单字段的数据都有效, 则返回 `True` 并将表单数据另存为 `cleaned_data` 类属性. 

##### 在 Django 中构建视图

表单数据通常会发送回通过 GET 请求创建它的视图, 因此视图中需要处理 GET 请求和 POST 请求. 

```python
from django.http import HttpResponseRedirect
from django.shortcuts import render

from .forms import NameForm

def get_name(request):
    # if this is a POST request we need to process the form data
    if request.method == "POST":
        # create a form instance and populate it with data from the request:
        form = NameForm(request.POST)
        # check whether it's valid:
        if form.is_valid():
            # process the data in form.cleaned_data as required
            # ...
            # redirect to a new URL:
            return HttpResponseRedirect("/thanks/")

    # if a GET (or any other method) we'll create a blank form
    else:
        form = NameForm()

    return render(request, "name.html", {"form": form})
```

##### 构建模板

```html
<form action="/your-name/" method="post">
    {% csrf_token %}
    {{ form }}
    <input type="submit" value="Submit">
</form>
```

#### 更多信息

##### 表单和跨站点请求伪造保护(CSRF)

##### 绑定或未绑定的表单实例

可通过 `is_bound` 属性来判断表单是否绑定

- 未绑定的表单实例通常不包含任何数据, 当呈现给用户时, 通常是空的或仅包含默认值
- 绑定的表单通常会含有提交的数据, 因此需要判断数据是否有效. 如果提交了无效数据, 表单会包含内敛的错误信息以告诉用户需要修正哪些数据

### 使用表单模板

将表单填至模板只需要将表单实例作为上下文参数传递给模板即可, 如果传递的表单实例为 `form` , 那么 `{{ form }}` 将自动呈现其中的 `<label>` 元素和 `<input>` 元素. 但使用者仍需自行处理 `<form>` 标签和提交按钮. 

#### 复用表单模板

将表单以片段化方式在表单中呈现: 

```html
# In your template:
{{ form }}

# In form_snippet.html:
{% for field in form %}
    <div class="fieldWrapper">
        {{ field.errors }}
        {{ field.label_tag }} {{ field }}
    </div>
{% endfor %}
```

可以自定义 `FORM_RENDERER` 以应用全局的表单模板: 

```python
from django.forms.renderers import TemplatesSetting

class CustomFormRenderer(TemplatesSetting):
    form_template_name = "form_snippet.html"

FORM_RENDERER = "project.settings.CustomFormRenderer"
```

或在某个表单类中定义表单模板:

```python
class MyForm(forms.Form):
    template_name = "form_snippet.html"
    ...
```

或在呈现视图时提供表单模板: 

```python
def index(request):
    form = MyForm()
    rendered_form = form.render("form_snippet.html")
    context = {"form": rendered_form}
    return render(request, "index.html", context)
```

#### 表单字段中的常用属性

如果需要对每个字段使用相同的 HTML 设置, 可以使用以下循环来进行: 

```html
{% for field in form %}
    <div class="fieldWrapper">
        {{ field.errors }}
        {{ field.label_tag }} {{ field }}
        {% if field.help_text %}
          <p class="help" id="{{ field.auto_id }}_helptext">
            {{ field.help_text|safe }}
          </p>
        {% endif %}
    </div>
{% endfor %}
```

要获取完整的属性或方法列表, 参考 [BoundField](https://docs.djangoproject.com/en/5.1/ref/forms/api/#django.forms.BoundField)

##### {{ field.errors }}

输出一个 `<ul class="errorlist">` 标签, 其中包含此字段所有的数据验证错误. 此标签还可以进行循环以自定义呈现方式: `{% for error in field.errors %}` .

##### {{ field.field }}

表单的字段实例. 可以用来进一步获取字段属性, 如 `{{ char_field.field.max_length }}`

##### {{ field.help_text }}

与字段相关的帮助文本

##### {{ field.html_name }}

将被用来作为输入元素名称, 如果游表单前缀, 将加入表单前缀

##### {{ field.id_for_label }}

将要被用来作为id. 

##### {{ field.is_hidden }}

表单字段为隐藏字段时, 此字段为 `True` .

##### {{ field.label }}

字段的标签

##### {{ field.label_tag }}

包装好的 HTML <tag> 标签的字段标签. 自动在后面添加表单的 `label_suffix` 值, 默认 `label_suffix = '':"`.

```html
<label for="id_email">Email address:</label>
```

##### {{ field.legend_tag }}

##### {{ field.use_fieldset }}

字段是否设置为 `fieldset` 

##### {{ field.value }}

