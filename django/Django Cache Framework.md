## Django Cache Framework

### 背景

对于动态网站来说, 每一次用户请求一个页面, web 服务器就需要执行所有类型的计算: 从数据库查询到呈现模板以创建用户需要的页面. 这将造成很大的服务器资源开销. 

对大多数 web 应用来说, 这个开销不是什么大事. 大多数站点的访问流量都很小. 但是对一些中型或大型流量的站点, 就非常有必要来尽可能控制这方面的开销. 

**这就引入了缓存机制**. 缓存避免了重复执行相同程序带来的资源消耗. 

Django 提供了一个可靠的缓存系统来保存动态页面, 以避免在请求时带来的重复资源消耗. 处于便捷性的考虑, Django 提供了不同层级的缓存粒度, 例如可以缓存特定视图的输出, 或仅仅缓存资源消耗大的部分, 或是缓存整个站点. 

Django 同样能够很好地与 "下游" 缓存配合使用, 比如 Squid 和基于浏览器的缓存. 这些是你不能直接控制的缓存类型, 但你可以通过HTTP头部向它们提供提示(hints), 说明你的网站的哪些部分应该被缓存, 以及如何缓存. 

### 使用缓存

缓存位置将极大程度地影响缓存性能, 缓存可以存在于三个地方: 

- 数据库
- 文件系统
- 内存

#### 基于 Memcahed 的缓存

`memcached` 是完全基于内存的缓存服务器, 某些站点使用其来减少数据库的查询数量. 其设置参考[链接](https://docs.djangoproject.com/en/5.1/ref/settings/#caches). 

基于内存的缓存作为一个守护进程运行, 并被分配了指定大小的 RAM, 提供一个添加, 获取, 和删除数据的快速接口. 所有数据都直接存储在内存中, 所以没有对于数据库或文件系统的资源开销. 

__使用方式: __

- 设置 `BACKEND` 为 `django.core.cache.backends.memcached.PyMemcacheCache` 或 `django.core.cache.backends.memcached.PyLibMCCache` (这两个选项分别依赖于 `pymemcache` 和 `pylibmc` 库)

- 设置 `LOCATION` 为 `IP:port` 指向内存缓存服务的 ip 地址及端口号, 或使用 `unix:path` 的格式来指定 Unix 套接字文件

  ``` python
  CACHES = {
      "default": {
          "BACKEND": "django.core.cache.backends.memcached.PyMemcacheCache",
          "LOCATION": "127.0.0.1:11211",
      }
  }
  
  CACHES = {
      "default": {
          "BACKEND": "django.core.cache.backends.memcached.PyMemcacheCache",
          "LOCATION": "unix:/tmp/memcached.sock",
      }
  }
  ```

**使用基于内存缓存的一个极佳特性是, 其可以在多个服务器中共享同一个缓存. 这意味着缓存守护进程可以运行在多个服务器上, 程序会将这些机器看作一个组提供一个单独的缓存, 而无需考虑每台机器上的缓存副本. **

```python
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.memcached.PyMemcacheCache",
        "LOCATION": [
            "172.19.26.240:11211",
            "172.19.26.242:11212",
            "172.19.26.244:11213",
        ],
    }
}
```

#### 基于 Redis 的缓存

Redis 是一个基于内存的数据库, 可以被用作缓存. 要在 Django 中使用需提前部署 Redis 服务. 

准备好 Redis 服务后, 需要安装 Redis 的 Python 插件. Django 天然支持的插件是 `redis-py` .

__使用方式: __

- 设置 `BACKEND` 为 `django.core.cache.backends.redis.RedisCache`

- 设置 `LOCATION` 指向 Redis 服务, 参考[redis-py 文档](https://redis-py.readthedocs.io/en/stable/connections.html#redis.connection.ConnectionPool.from_url).

  ```python
  CACHES = {
      "default": {
          "BACKEND": "django.core.cache.backends.redis.RedisCache",
          "LOCATION": "redis://127.0.0.1:6379",
      }
  }
  
  CACHES = {
      "default": {
          "BACKEND": "django.core.cache.backends.redis.RedisCache",
          "LOCATION": "redis://username:password@127.0.0.1:6379",
      }
  }
  
  # 使用多个服务器时, 写操作将在指定的首个服务器(leader)中进行, 读操作则会在其他服务器中随机执行. 
  CACHES = {
      "default": {
          "BACKEND": "django.core.cache.backends.redis.RedisCache",
          "LOCATION": [
              "redis://127.0.0.1:6379",  # leader
              "redis://127.0.0.1:6378",  # read-replica 1
              "redis://127.0.0.1:6377",  # read-replica 2
          ],
      }
  }
  ```

#### 基于数据库的缓存

Django 也可以将缓存数据存储在数据库, 如果你有一个快速的, 索引良好的数据库, 这种方式是最佳的. 

__使用方式: __

- 设置 `BACKEND` 为 `django.core.cache.backends.db.DatabaseCache`

- 设置 `LOCATION` 指向数据库表名. 

  ```python
  CACHES = {
      "default": {
          "BACKEND": "django.core.cache.backends.db.DatabaseCache",
          "LOCATION": "my_cache_table",
      }
  }
  ```

- 创建缓存表, 如果指定了多个数据库缓存, 则每个缓存都会创建一个表. 类似于 `migrate` 命令, 这个命令不会改变已存在的表, 只会创建不存在的表. 

  ``` shell
  python manage.py createcachetable
  ```

##### 多数据库

如果使用多个数据库作为缓存, 必须要设置数据库缓存表的路由策略. 

#### 基于文件系统的缓存

基于文件系统的缓存将每一个缓存值序列化并存储为一个单独的文件. 

**使用方式: **

- 设置 `BACKEND` 为 `django.core.cache.backends.filebased.FileBasedCache`

- 设置`LOCATION` 指向适当的目录, 必须为绝对路径. 所指向的目录需要在当前执行 django 的用户有读写权限.  

  ```python
  CACHES = {
      "default": {
          "BACKEND": "django.core.cache.backends.filebased.FileBasedCache",
          "LOCATION": "/var/tmp/django_cache",
      }
  }
  
  # Windows 操作系统
  CACHES = {
      "default": {
          "BACKEND": "django.core.cache.backends.filebased.FileBasedCache",
          "LOCATION": "c:/foo/bar",
      }
  }
  ```

#### 基于本地内存的缓存

如果在设置文件中没有指定另外的缓存系统, 这就是默认的缓存系统. 

**使用方式: **

```python
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.locmem.LocMemCache",
        "LOCATION": "unique-snowflake",
    }
}
```

`LOCATION` 字段被用来区分不同的缓存存储, 如果只有一个`locmem` 缓存, 可以忽略这个选项. 如果有多个本地缓存, 则需要为其命名加以区分. 

**使用 Least-recently-used (LRU) 作为数据剔除策略.**

**每个进程都会有自己私有的缓存实例, 这意味着不支持进程间的缓存, 也就是说基于本地内存的缓存将降低内存效率, 在生产环境中不是一个好的选择, 但在开发环境中不错. **

#### 假缓存

**Dummy caching 是一种专为开发环境设计的缓存机制, 它实现了缓存接口但不进行任何缓存操作. 这有助于开发者在开发或测试环境中避免受到缓存的干扰. 从而更轻松地调试和验证应用**

一般用在开发环境中, 不想要使用 cache 机制, 且又不想对已有的 cache 配置进行修改. 

```python
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.dummy.DummyCache",
    }
}
```

#### 自定义 cache 后端



### 缓存参数

`TIMEOUT` : 缓存的默认超时时间, 以秒计算. 默认 300 秒, 若设置 None , 则缓存永远不会过期; 若设置 0 , 缓存立即失效, 即不缓存.

`OPTIONS` : 缓存后端支持的配置项, 不同的后端支持不同的选项. 

例如 `locmem` , `filesystem` , `database` 支持的一些配置如下:

	-	`MAX_ENTRIES` : 允许的最大缓存条目, 默认值 300
	-	`CULL_FREQUENCY` : 当达到最大缓存条目时的剔除频率,  即剔除的比例为 `1/CULL_FREQUENCY` , 默认为 3 . 此值设置为 0 意味着当达到最大缓存条目时, 所有缓存都会被丢弃, 对一些数据库缓存来说, 这会更有执行效率但会降低缓存命中率. 

对 `Memcached` 和 `Redis` 后端, 配置项会作为关键字参数在传递给客户端, 以实现对客户端行为的更高级的控制. 

- `KEY_PREFIX` : 缓存键值的前缀.
- `VERSION` : 缓存键生存所使用的默认版本号
- `KEY_FUNCTION` : 可调用的函数路径以指定如何构建缓存的前缀, 版本和键值

**一些配置案例: **

```python
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.filebased.FileBasedCache",
        "LOCATION": "/var/tmp/django_cache",
        "TIMEOUT": 60,
        "OPTIONS": {"MAX_ENTRIES": 1000},
    }
}
```

```python
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.memcached.PyLibMCCache",
        "LOCATION": "127.0.0.1:11211",
        "OPTIONS": {
            "binary": True,
            "username": "user",
            "password": "pass",
            "behaviors": {
                "ketama": True,
            },
        },
    }
}
```

```python
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.memcached.PyMemcacheCache",
        "LOCATION": "127.0.0.1:11211",
        "OPTIONS": {
            "no_delay": True,
            "ignore_exc": True,
            "max_pool_size": 4,
            "use_pooling": True,
        },
    }
}
```

```python
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379",
        "OPTIONS": {
            "db": "10",
            "pool_class": "redis.BlockingConnectionPool",
        },
    }
}
```

### 站点缓存

参考[Order of MIDDLEWARE](https://docs.djangoproject.com/en/5.1/topics/cache/#order-of-middleware), 按如下顺序设置中间件: 

```python
MIDDLEWARE = [
    "django.middleware.cache.UpdateCacheMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.cache.FetchFromCacheMiddleware",
]
```

然后在配置文件中进行如下设置: 

- `CACHE_MIDDLEWARE_ALIAS` : 设置缓存存储的别名
- `CACHE_MIDDLEWARE_SECONDS` : 每个页面需要缓存的秒数
- `CACHE_MIDDLEWARE_KEY_PREFIX` : 如果一个 django 项目实例有多个网站, 或在多个环境中使用相同的配置, 配置此项为缓存添加前缀以避免缓存键的冲突

#### 缓存中间件

##### FetchFromCacheMiddleware

这个中间件缓存状态码为 200 的 GET 请求和 HEAD 请求. 相同 URL 请求参数不同的请求被视作不同的页分开缓存. 

HEAD 请求页面的响应头, 当指定 URL 相同的 GET 请求已被缓存, 则将返回该 GET 请求的响应头. 

##### UpdateCacheMiddleware

此中间件将为每一个 HttpResponse 设置请求头, 以控制下游缓存. 

- 根据 `CACHE_MIDDLEWARE_SECONDS` 设置 `Expires` 头部
- 根据 `CACHE_MIDDLEWARE_SECONDS` 设置 `Cache-Control` 头部

**注: **

如果一个视图设置了自己的缓存到期时间, 则页面会根据此时间进行缓存, 而不是默认的  `CACHE_MIDDLEWARE_SECONDS` . 使用 `django.views.decorators.cache` 提供的装饰器可以轻松设置缓存过期时间(使用 `cache_control()` 装饰器), 或禁用某个视图的缓存(使用 `never_cache()` 装饰器). 

若设置 `USE_I18N=True` 则生成的缓存键将包含当前使用的语言. 

若设置 `USE_TZ=True` 则缓存键也会包含当前时区.

### 视图缓存

视图装饰器 `django.views.decorators.cache.cache_page`  , 只接受一个位置参数, 即缓存时间. 

```python
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)
def my_view(request): ...
```

`cache_page()` 也可接受关键字参数, 如 `cache` , 以指定使用的缓存后端, 默认将使用 `default` . 

`key_prefix` , 与设置 `CACHE_MIDDLEWARE_KEY_PREFIX` 类似, **但二者会被拼接**. 

```python
@cache_page(60 * 15, cache="special_cache")
def my_view(request): ...

@cache_page(60 * 15, key_prefix="site1")
def my_view(request): ...
```

也可以在 URLconf 中对视图应用装饰器函数

```python
from django.views.decorators.cache import cache_page

urlpatterns = [
    path("foo/<int:code>/", cache_page(60 * 15)(my_view)),
]
```

### 模板缓存

使用模板标签中的 `cache` 标签以实现更精细化的缓存控制. 

`{% cache %}` 模板标签将缓存内容块一段时间, 其接受至少两个参数: 缓存时间和缓存名称, 缓存名称不要使用变量, 会直接作为缓存键的名称使用. 

```python
{% load cache %}
{% cache 500 sidebar %}
    .. sidebar ..
{% endcache %}
```

有时也许想要根据动态请求数据来缓存同一个片段的多个副本. 例如有时想要缓存每个用户自己的侧边栏, 则可传递额外的参数(可以是原始变量, 或是经过模板过滤器输出的变量)来区分这些缓存片段: 

```python
{% load cache %}
{% cache 500 sidebar request.user.username %}
    .. sidebar for logged in user ..
{% endcache %}
```

若设置 `USE_I18N=True` , 则每个站点的缓存中间件将考虑当前语言. 要在模板中使用这样的机制, 如下实现: 

```html
{% load i18n %}
{% load cache %}
{% get_current_language as LANGUAGE_CODE %}
{% cache 600 welcome LANGUAGE_CODE %}
    {% translate "Welcome to example.com" %}
{% endcache %}
```

缓存时间也可以使用模板变量确定. 

```python
{% cache 600 sidebar %} ... {% endcache %}
{% cache my_timeout sidebar %} ... {% endcache %}
```

##### 模板缓存使用的缓存系统

默认情况下, 缓存标签将会尝试使用名为 `template_fragments` 的缓存, 如果这样的缓存不存在, 会使用默认缓存 `default` . 你也可以通过 `using` 关键字参数人为指定使用的缓存, 关键字参数必须作为最后一个参数. 

```python
{% cache 300 local-thing ...  using="localcache" %}
```

##### 获取模板片段缓存的键值

可以通过这个函数来删除或重写某些缓存项. 

`django.core.cache.utils.make_template_fragment_key(fragment_name,vary_on=None)`

`fragment_name` 是 `cache` 模板标签的第一个参数, `vary_on` 是后续传递的参数列表. 

```python
>>> from django.core.cache import cache
>>> from django.core.cache.utils import make_template_fragment_key
# cache key for {% cache 500 sidebar username %}
>>> key = make_template_fragment_key("sidebar", [username])
>>> cache.delete(key)  # invalidates cached template fragment
True
```

### 缓存底层 API

参考[链接](https://docs.djangoproject.com/en/5.1/topics/cache/#the-low-level-cache-api). 

### 下游缓存