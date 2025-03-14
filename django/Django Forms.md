## Django Forms

#### 绑定表单与未绑定表单

表单实例分为两种: 绑定的和未绑定的

- 当表单与某个数据集绑定, 就可以核实数据的有效性, 并且可以 作为 HTML 将数据呈现
- 若表单未绑定, 则不可以进行数据有效性的核实, 但依然可以作为空表单呈现为 HTML

#### 表单初始值

`Form.initial`

```python
>>> f = ContactForm(initial={"subject": "Hi there!"})
```

#### 获取表单数据

```python
>>> for row in f.fields.values():
...     print(row)
...
<django.forms.fields.CharField object at 0x7ffaac632510>
<django.forms.fields.URLField object at 0x7ffaac632f90>
<django.forms.fields.CharField object at 0x7ffaac3aa050>
>>> f.fields["name"]
<django.forms.fields.CharField object at 0x7ffaac6324d0>

>>> f.as_div().split("</div>")[0]
'<div><label for="id_subject">Subject:</label><input type="text" name="subject" maxlength="100" required id="id_subject">'
>>> f["subject"].label = "Topic"
>>> f.as_div().split("</div>")[0]
'<div><label for="id_subject">Topic:</label><input type="text" name="subject" maxlength="100" required id="id_subject">'
```

#### 不要改变 base_fields, 这将改变整个表单类, 并影响所有后面创建的实例

```python
>>> f.base_fields["subject"].label_suffix = "?"
>>> another_f = ContactForm(auto_id=False)
>>> another_f.as_div().split("</div>")[0]
'<div><label for="id_subject">Subject?</label><input type="text" name="subject" maxlength="100" required id="id_subject">'
```

### 将表单输出为 HTML

参考[链接](https://docs.djangoproject.com/en/5.1/ref/forms/api/#outputting-forms-as-html).

#### 默认呈现

当你打印一个表单时, 将使用如下方法和属性完成表单的默认呈现,

##### template_name

`Form.template_name` 

当表单要被转化为字符串时(通过 `print(form)` 或 `{{ form }}`)需要呈现的表单名称. 默认使用渲染器的 `form_template_name` 定义, 你可以覆盖它以设置整个表单子类的默认模板.  

##### render()

`Form.render(template_name=None, context=None, renderer=None)`

此方法由 `__str()__` 或 `Form.as_div()` `Form.as_table()` `Form.as_p()` `Form.as_ul()` 方法调用, 默认参数如下: 

- template_name: Form.template_name, 通过传递此参数可以为单次的表单渲染提供自定义模板. 
- context: Form.get_context() 方法的返回值
- renderer: Form.default_renderer 的返回值

##### get_context()

`Form.get_context()`

返回呈现表单所需要的上下文: 

- form: 绑定的表单
- fields: 所有绑定的字段, 隐藏字段除外
- hidden_fields: 所有隐藏的绑定字段
- errors: 所有非字段相关的或隐藏字段相关的错误

##### template_name_label

`Form.template_name_label` 

用来呈现字段 `<label>` 标签的模板. 

#### 输出样式

可以再站点范围, 表单范围, 实例范围内设置自定义的表单模板. 