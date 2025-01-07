## Django Model

### Meta 元数据选项

模型的元数据是模型中除字段以外的任何属性. 例如排序选项 `ordering` , 数据库表名 `db_table` , 用户友好的单数和复数名称     `verbose_name` `verbose_name_plural`.  这些属性都不是必须的, 设置 `Meta` 类也不少必须的. 

通过类内的 `Meta` 类设置模型的元数据. 例如: 

```python
from django.db import models

class Ox(models.Model):
    horn_length = models.IntegerField()

    class Meta:
        ordering = ["horn_length"]
        verbose_name_plural = "oxen"
```

`Meta` 类中支持的所有属性参考[链接](https://docs.djangoproject.com/en/5.1/ref/models/options/)

### 模型属性

#### objects

模型中最重要的属性就是模型管理器, 作为进行数据库查找操作的接口, 被 Django 用以从数据库中获取数据. 如果没有自定义管理器, 默认的名称就是 `objects` . 管理器只能通过模型类获取, 而不能通过模型的实例对象获取. 

### 模型方法

##### \_\_str\_\_()

##### get_absolute_url()

##### save()

##### delete()

### 模型继承

在 Django 中主要有三种模型继承方式

1. 使用抽象模型. 父模型列出公有信息. 

2. 多表继承, 父模型和子模型都会拥有自己的数据库表
3. 代理模型, 只是希望暂时在 Python 层面修改模型的行为, 而非修改模型的字段

#### 基于抽象基类的继承

抽象模型不会创建数据库表, 也没有模型管理器, 不可以被初始化或被保存, 只是作为其他模型的基类使用, 抽象模型定义的字段会被子类所共享. 

```python
from django.db import models

class CommonInfo(models.Model):
    name = models.CharField(max_length=100)
    age = models.PositiveIntegerField()

    class Meta:
        abstract = True

class Student(CommonInfo):
    home_group = models.CharField(max_length=5)
```

##### 抽象基类继承中的 Meta 类属性

当抽象基类创建时, Django 会将你定义的类内 `Meta` 类作为属性. 如果某个子类没有定义自己的 `Meta` 类, 则其将继承父类的 `Meta` 类, 如果子类需要扩展父类的 `Meta` 类, 则需继承父类的 `Meta` 类, 如下:

```python
from django.db import models

class CommonInfo(models.Model):
    # ...
    class Meta:
        abstract = True
        ordering = ["name"]

class Student(CommonInfo):
    # ...
    class Meta(CommonInfo.Meta):
        db_table = "student_info"
```

- 抽象类的子类在继承时, Django 会对 `Meta` 类进行一些调整: 抽象类的子类中的 `Meta` 类会自动设置 `abstract=False` . 这意味着抽象类的子类不会自动成为抽象类. 若其仍需保持为抽象类, 则需显式在子类中设置 `abstract=False` .

- 某些包含在抽象基类中的元属性是无意义的, 例如定义数据库表名的 `db_table` .

- 出于 Python 的继承机制, 当某个子类继承多个抽象父类时, 只有第一个父类的元数据将被继承. 如果需要继承多个父类的元数据, 需要以如下方式显式定义: 

  ```python
  from django.db import models
  
  class CommonInfo(models.Model):
      name = models.CharField(max_length=100)
      age = models.PositiveIntegerField()
  
      class Meta:
          abstract = True
          ordering = ["name"]
  
  class Unmanaged(models.Model):
      class Meta:
          abstract = True
          managed = False
  
  class Student(CommonInfo, Unmanaged):
      home_group = models.CharField(max_length=5)
  
      class Meta(CommonInfo.Meta, Unmanaged.Meta):
          pass
  ```

##### 抽象基类中定义的 `related_name` 和 `related_query_name`

如果使用基于外键或多对多关系的反向查找, 必须要指定一个唯一的反向名称和查找名称. 在不指定 `related_name `时, 默认的名称将是子类的类型加上 `_set` 后缀, 这可能会导致冲突. 

当多个子类继承的抽象基类中定义了  `related_name` 和 `related_query_name` 时, 所有子类都会拥有相同的该字段属性, 这将导致不唯一的反向名称和查找名称.

要解决这个问题, 可以在抽象基类中参考如下设置: 

```python
from django.db import models

class Base(models.Model):
    m2m = models.ManyToManyField(
        OtherModel,
        related_name="%(app_label)s_%(class)s_related",
        related_query_name="%(app_label)s_%(class)ss",
    )

    class Meta:
        abstract = True
```

#### 多表继承

多表继承的情况下, 每个模型都有自己单独的数据库表, 可以独立创建或查询. 继承关系会在子类和父类关系中通过自动创建的一对一字段引入外键关系. 

```python
place_ptr = models.OneToOneField(
    Place,
    on_delete=models.CASCADE,
    parent_link=True,
    primary_key=True,
)
```

##### 多表继承中的 Meta 类属性

在多表继承中, 子类继承父类的 `Meta` 类是没有意义的. 所有已经应用到父类当中的元属性再应用到子类将会导致一些冲突行为. 

因此, 子类模型无法访问其父类的 `Meta` 类. 但在某些有限的情况下,  子类会继承父类的行为. 例如当子类没有定义 `ordering` 元属性或 `get_latest_by` 元属性时, 其将继承父类的相关属性. 若无需这样的自动继承特性, 需显式地将其取消: 

```python
class ChildModel(ParentModel):
    # ...
    class Meta:
        # Remove parent's ordering effect
        ordering = []
```

##### 多表继承和反向关联

多表继承会隐式使用一对一字段来连接子模型和父模型, 因此可以从父模型访问到子模型的数据. 然而, 这使用的是外键查找或多对多关系查找默认的 `related_name` 进行的. 如果把这些外键关系设置在子类中, 有冲突时必须设置 `related_name` 属性. 例如:

```python
class Supplier(Place):
    customers = models.ManyToManyField(Place)
    
'''
Reverse query name for 'Supplier.customers' clashes with reverse query
name for 'Supplier.place_ptr'.

HINT: Add or change a related_name argument to the definition for
'Supplier.customers' or 'Supplier.place_ptr'.  
'''

# Adding related_name to the customers field as follows would resolve the error: models.ManyToManyField(Place, related_name='provider').
```

#### 代理模型

```python
from django.db import models

class Person(models.Model):
    first_name = models.CharField(max_length=30)
    last_name = models.CharField(max_length=30)

class MyPerson(Person):
    class Meta:
        proxy = True

    def do_something(self):
        # ...
        pass
```

- 代理模型返回的对象是基于原模型的. 代理模型一般用来对原模型的一些元属性和方法进行自定义更新, 而不是用来完全替代原模型的. 
- 代理模型必须要继承非抽象的数据模型类, 且只能继承一个类. 代理模型可以继承任意多个共享相同非抽象父类的代理模型.

##### 代理模型管理器

若未为代理模型指定模型管理器, 则其将继承父类模型的管理器. 当为代理模型指定模型管理器, 其成为默认管理器, 但父类管理器仍可使用. 

#### 多父类继承

与 Python 的继承方式一样, Django 的模型可以继承多个父类. 第一个父类的同名属性将被使用, 例如 `Meta` 类. 

一般情况下不需要同时继承多个父类, 多父类继承的一个主要用途是为了添加混入类 `mix-in` , 以为子类添加一些额外的字段或方法. 尽量保持继承结构的简单性和直接性以便于代码维护. 

当继承多个父类模型时, 需要注意若父类模型都没有指定主键字段时, Django 会自动添加一个名为 `id` 的主键字段, 这样多个父类的主键字段会在子类产生冲突, 显示指定主键字段以避免冲突: 

```python
class Article(models.Model):
    article_id = models.AutoField(primary_key=True)
    ...

class Book(models.Model):
    book_id = models.AutoField(primary_key=True)
    ...

class BookReview(Book, Article):
    pass
```

#### 继承中的一些问题

通常在 Python 的类继承中, 继承父类的子类中可以重写父类的任何属性. 但在 Django 中, 这经常是不被允许的. 

如果一个非抽象类的父类模型定义了一个字段 `author` , 那么你在继承的时候就不能创建另一个 `author` 字段. (这是因为在非抽象类的继承中, 将通过一对一字段将子类的数据库表连接至父类, 若定义了相同字段, 则可能会产生冲突). 

值得注意的是, 在抽象类的继承中是可以重写某些字段的, 也可以通过设置 `field_name = None` 来移除某些字段. 

模型管理器继承自抽象基类, 若此时继承的子类中重写了某些父类基类中管理器引用的字段, 可能会导致 bug . 

### 在 App 中管理 Models 模块

默认情况下 App 内只生成一个 `models.py` 模块文件, 如果有多个模块文件, 可以将他们用不同的文件管理起来. 如下:

1. 创建一个 `models` 文件夹, 在文件夹中创建 `__init__.py` 文件以及不同的模块文件, 并且在 `__init__.py` 文件中导入这些模块文件

   ```python
   # myapp/models/__init__.py
   
   from .organic import Person
   from .synthetic import Robot
   ```

2. 使用 `from .models import *` 以导入所有模块中定义的模型