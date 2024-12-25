## django-admin

[文档链接](https://docs.djangoproject.com/en/5.1/ref/django-admin/#usage)

### django-admin 和 manage.py

django-admin 是 django 的命令行管理工具。

每个 django 项目将 自动创建一个 manage.py 脚本，他的功能与 django-admin 基本相同，不同的是，django-admin 需要预先设置DJANGO_SETTINGS_MODULE 环境变量以指向具体的项目，而 manage.py 脚本中自动设置了此项

通常情况下对于单个 django 项目，使用 manage.py 进行管理会方便得多。如果需要在多个 Django 设置文件中切换，那么可以使用 django-admin 并设置 DJANGO_SETTINGS_MODULE 环境变量，或在命令中使用 ```--settings``` 选项

### Usage

使用 django-admin 与使用 manage.py 或是 ```python -m django``` 的作用相同：

```shell
django-admin <command> [options]
manage.py <command> [options]
python -m django <command> [options]
```

获取帮助：

``` shell
django-admin help
django-admin help <command>
```

### Available commands

#### check

使用系统的检查框架来检查整个 Django  项目中存在的一些普遍性问题：

``` shell
django-admin check [app_label [app_label ...]]
```

#### compilemessages

关于 Internationalization and localization。

#### createcachetable

关于 Django's cache framework。

#### dbshell

运行设置中指定的 ENGINE 数据库的命令行客户端，使用设置文件中的 USER，PASSWORD 等参数作为连接参数。

```shell
django-admin dbshell [--database DATABASE] -- ARGUMENTS
```

所有在 `--` 符号后的参数将被传递至数据库的命令行客户端作为数据库命令的输入参数。

#### diffsettings

显示当前的设置文件与 Django 的默认设置文件之间的差别。

#### dumpdata

将数据库中的应用数据导出至标准输出，此命令可以作为 ```loaddata``` 命令的输入，可选数据的导出格式及显示的缩进空格数等参数。

```shell
django-admin dumpdata [app_label[.ModelName] [app_label[.ModelName] ...]]
```

#### flush

关于清除数据库数据。

#### inspectdb

检查由 NAME 设置的数据库中的数据表结构并输出相关的 Django 模型配置代码到标准输出。 可以使用此命令对现有的数据库进行分析，从而快速创建其所需要的数据库模型的定义代码。

#### loaddata

将数据导入至数据库。

#### makemessages

关于 前文命令 ```compilemessages``` 

#### makemigrations

根据检测到的模型修改来创建数据库的迁移文件。

#### migrate

将数据库的状态与当前的数据模型和迁移文件同步，即修改数据库的表结构以映射当前的模型定义。

#### optimizemigration

#### runserver

在本地启动一个轻量级的 web 服务器用于开发。用于开发的服务器将自动重载 Python 代码，因此当代码发生修改时无需重启服务器。需要注意的是，例如添加文件等的一些操作将不会触发自动重载，需要手动重启服务器以生效。

可使用 pywatchman 和 watchman 来通过内核信号触发服务器的自动重载，对一些大型的项目，这会降低代码更改的响应时间，提供更具鲁棒性的改动检测，有更好的性能体验和更低的能耗。

默认情况下，开发服务器不提供静态文件服务。如果需要配置 Django 提供静态媒体服务，参考 [How to manage static files (e.g. images, JavaScript, CSS)](https://docs.djangoproject.com/en/5.1/howto/static-files/)

#### sendtestemail

通过项目设置文件中的已配置项来发送测试邮件，以确认是否能够正常通过 Django 发送邮件。

#### shell

启动 Python 的可交互解释器。

#### showmigrations

显示项目中的所有迁移文件及相关的迁移状态。

#### sqlflush

显示执行 flush 命令将会执行的 SQL 代码。

#### sqlmigrate

为指定的迁移文件展示其 SQL 代码。

#### sqlsequencereset

显示重置序列将会执行的 SQL 代码。

sequences被某些数据库引擎用来获取自增字段的下一个可用序号。

#### squashmigrations

用于合并迁移文件，简化迁移历史。

#### startapp

创建 Django app 目录结构。

#### startproject

创建 Django 项目目录结构。

#### test

为所有已安装的 app 执行测试。

#### testserver

使用给定文件的数据运行 Django 开发服务器。

### Commands provided by applications

#### django.contrib.auth

##### changepassword

##### createsuperuser

#### django.contrib.contenttypes

##### remove_stable_contenttypes

#### django.contrib.gis

##### originspect

#### django.contrib.sessions

##### clearsessions

#### django.contrib.staticfiles

##### collectstatic

##### findstatic



