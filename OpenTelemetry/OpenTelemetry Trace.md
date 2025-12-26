## OpenTelemetry Trace链路数据采集

### Python 环境

参考[链接](https://www.cnblogs.com/MrVolleyball/p/19206221).

#### 1. 环境准备

```shell
# 安装 OpenTelemetry-sdk
pip3 install opentelemetry-sdk opentelemetry-exporter-otlp opentelemetry-api

# 安装数据展示 jaeger UI
docker pull docker.m.daocloud.io/jaegertracing/all-in-one:latest

docker run -d --name jaeger \
  -e COLLECTOR_OTLP_ENABLED=true \
  -p 16686:16686 \
  -p 4317:4317 \
  -p 4318:4318 \
  docker.m.daocloud.io/jaegertracing/all-in-one:latest
  
# 访问 http://127.0.0.1:16686 显示 jaeger UI
```

#### 2. web 服务文件

##### 1. web 服务 s1

```python
# s1.py
# curl http://localhost:10000
import tornado.httpserver as httpserver
import tornado.web
from tornado.ioloop import IOLoop
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.resources import SERVICE_NAME, Resource
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator
import requests
import os


JAEGER_ADDR=os.environ.get('JAEGER_ADDR', "http://localhost:4318/v1/traces")
S2_ADDR=os.environ.get('S2_ADDR', "http://localhost:20000")

trace.set_tracer_provider(
    TracerProvider(resource=Resource.create({SERVICE_NAME: "s1"}))
)
tracer = trace.get_tracer(__name__)
span_processor = BatchSpanProcessor(OTLPSpanExporter(endpoint=JAEGER_ADDR))
trace.get_tracer_provider().add_span_processor(span_processor)


class TestFlow(tornado.web.RequestHandler):
    def get(self):
        views()
        self.finish('hello world 1\n')

def views():
    span = tracer.start_span("s1-span")
    span.set_attribute("name", "wilson")
    span.set_attribute("addr", "cd")
    ctx = trace.set_span_in_context(span)
    get_db(ctx)
    headers = {}
    TraceContextTextMapPropagator().inject(headers, context=ctx)
    requests.get(S2_ADDR, headers=headers)
    span.end()

def get_db(parent_ctx):
    span = tracer.start_span("s1-span-db", context=parent_ctx)
    span.end()

def applications():
    urls = []
    urls.append([r'/', TestFlow])
    return tornado.web.Application(urls)

def main():
    app = applications()
    server = httpserver.HTTPServer(app)
    server.bind(10000, '0.0.0.0')
    server.start(1)
    IOLoop.current().start()


if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt as e:
        IOLoop.current().stop()
    finally:
        IOLoop.current().close()
```

##### 2. web 服务 s2

```python
# s2.py
# curl http://localhost:20000
import tornado.httpserver as httpserver
import tornado.web
from tornado.ioloop import IOLoop
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.resources import SERVICE_NAME, Resource
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator
import os


JAEGER_ADDR=os.environ.get('JAEGER_ADDR', "http://localhost:4318/v1/traces")

trace.set_tracer_provider(
    TracerProvider(resource=Resource.create({SERVICE_NAME: "s2"}))
)
tracer = trace.get_tracer(__name__)
span_processor = BatchSpanProcessor(OTLPSpanExporter(endpoint=JAEGER_ADDR))
trace.get_tracer_provider().add_span_processor(span_processor)


class TestFlow(tornado.web.RequestHandler):
    def get(self):
        ctx = TraceContextTextMapPropagator().extract(self.request.headers)
        span = tracer.start_span("s2-span", context=ctx)
        span.end()
        self.finish('hello world 2\n')

def applications():
    urls = []
    urls.append([r'/', TestFlow])
    return tornado.web.Application(urls)

def main():
    app = applications()
    server = httpserver.HTTPServer(app)
    server.bind(20000, '0.0.0.0')
    server.start(1)
    IOLoop.current().start()


if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt as e:
        IOLoop.current().stop()
    finally:
        IOLoop.current().close()
```

##### 3. 基于装饰器实现 Python 代码注入

```python
# decorator-s1.py
import tornado.httpserver as httpserver
import tornado.web
from tornado.ioloop import IOLoop
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.resources import SERVICE_NAME, Resource
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.trace import get_tracer


trace.set_tracer_provider(
    TracerProvider(resource=Resource.create({SERVICE_NAME: "decoration-s1"}))
)
span_processor = BatchSpanProcessor(OTLPSpanExporter(endpoint="http://127.0.0.1:4318/v1/traces"))
trace.get_tracer_provider().add_span_processor(span_processor)


def traced(name):
    def decorator(func):
        def wrapper(*args, **kwargs):
            tracer = get_tracer(__name__)
            with tracer.start_as_current_span(name):
                return func(*args, **kwargs)
        return wrapper
    return decorator

class TestFlow(tornado.web.RequestHandler):
    def get(self):
        views()
        self.finish('hello world')

@traced("phase-1")
def views():
    views_2()
    headers = {}
    inject(headers)
    requests.get("http://127.0.0.1:20000", headers=headers)

@traced("phase-2")
def views_2():
    pass

def applications():
    urls = []
    urls.append([r'/', TestFlow])
    return tornado.web.Application(urls)

def main():
    app = applications()
    server = httpserver.HTTPServer(app)
    server.bind(10000, '0.0.0.0')
    server.start(1)
    IOLoop.current().start()


if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt as e:
        IOLoop.current().stop()
    finally:
        IOLoop.current().close()
```

#### 3. OpenTelemetry Instrumentation

##### 1. 安装工具

```shell
pip3 install opentelemetry-instrumentation opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install

# 查看支持的自动采集
pip3 list | grep instrumentation
```

##### 2. web 服务文件

```python
# auto-s1.py
from tornado.ioloop import IOLoop
import tornado.httpserver as httpserver
import tornado.web
import redis
import pymysql
import requests

class TestFlow(tornado.web.RequestHandler):
    def get(self):
        views()
        self.finish('hello world')

def views():
    get_redis()
    get_db()
    get_s2()

def get_redis():
    r = redis.StrictRedis(host='localhost', port=6379, db=0)
    r.set('name', 'hello')

def get_db():
    db = pymysql.connect(host='localhost', user='root', password='123456')
    cursor = db.cursor()
    cursor.execute("SELECT VERSION()")
    data = cursor.fetchone()
    db.close()

def get_s2():
    requests.get('http://127.0.0.1:20000')

def applications():
    urls = []
    urls.append([r'/', TestFlow])
    return tornado.web.Application(urls)

def main():
    app = applications()
    server = httpserver.HTTPServer(app)
    server.bind(10000, '0.0.0.0')
    server.start(1)
    IOLoop.current().start()


if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt as e:
        IOLoop.current().stop()
    finally:
        IOLoop.current().close()
```

```python
# auto-s2.py
from tornado.ioloop import IOLoop
import tornado.httpserver as httpserver
import tornado.web

class TestFlow(tornado.web.RequestHandler):
    def get(self):
        views()
        self.finish('hello world')

def views():
    pass

def applications():
    urls = []
    urls.append([r'/', TestFlow])
    return tornado.web.Application(urls)

def main():
    app = applications()
    server = httpserver.HTTPServer(app)
    server.bind(20000, '0.0.0.0')
    server.start(1)
    IOLoop.current().start()


if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt as e:
        IOLoop.current().stop()
    finally:
        IOLoop.current().close()
```

##### 3. 通过自动注入启动 web 服务

```shell
opentelemetry-instrument \
    --traces_exporter otlp \
    --metrics_exporter none \
    --service_name auto-s1 \
    --exporter_otlp_endpoint http://localhost:4317 \
    python3 auto-s1.py
    
opentelemetry-instrument \
    --traces_exporter otlp \
    --metrics_exporter none \
    --service_name auto-s2 \
    --exporter_otlp_endpoint http://localhost:4317 \
    python3 auto-s2.py
```

#### 4. 基于 k8s 部署

##### 0. ctr 镜像管理(基于 containerd 容器运行时)

```shell
# 列出命名空间下所有镜像
sudo ctr -n k8s.io images ls
# 拉取镜像至指定命名空间
sudo ctr -n k8s.io images pull --user admin:admin registry.iasf/s1:v1
# 删除指定命名空间镜像
sudo ctr -n k8s.io images remove registry.iasf/s1:v1
```

##### 1. jaeger

```yaml
# jaeger.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: jaeger
  name: jaeger
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
      - image: docker.m.daocloud.io/jaegertracing/all-in-one:latest
        imagePullPolicy: Always
        name: jaeger
      dnsPolicy: ClusterFirst
      restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: jaeger-service
  name: jaeger-service
  namespace: default
spec:
  ports:
  - name: port-4317
    port: 4317
    protocol: TCP
    targetPort: 4317
  - name: port-4318
    port: 4318
    protocol: TCP
    targetPort: 4318
  - name: port-16686
    port: 16686
    protocol: TCP
    targetPort: 16686
  selector:
    app: jaeger
  type: NodePort
```

##### 2. s1

```dockerfile
# s1.dockerfile
# 编译时使用 --no-cache 选项，否则可能产生无法识别的镜像依赖问题
FROM python:3.10-slim

WORKDIR /opt
RUN pip3 install tornado opentelemetry-api opentelemetry-sdk opentelemetry-exporter-otlp -i https://pypi.tuna.tsinghua.edu.cn/simple
ADD s1.py /opt
CMD ["python3", "s1.py"]
```

```yaml
# s1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: s1
  name: s1
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: s1
  template:
    metadata:
      labels:
        app: s1
    spec:
      containers:
      - env:
        - name: S2_ADDR
          value: http://s2-service:20000
        - name: JAEGER_ADDR
          value: http://jaeger-service:4318/v1/traces
        image: s1:v1
        imagePullPolicy: Always
        name: s1
      dnsPolicy: ClusterFirst
      restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: s1-service
  name: s1-service
  namespace: default
spec:
  ports:
  - name: s1-port
    port: 10000
    protocol: TCP
    targetPort: 10000
  selector:
    app: s1
  type: NodePort
```

##### 3. s2

```dockerfile
# s2.dockerfile
# 编译时使用 --no-cache 选项，否则可能产生无法识别的镜像依赖问题
From python:3.10-slim

WORKDIR /opt
RUN pip3 install tornado opentelemetry-api opentelemetry-sdk opentelemetry-exporter-otlp -i https://pypi.tuna.tsinghua.edu.cn/simple
ADD s2.py /opt
CMD ["python3", "s2.py"]
```

```yaml
# s2.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: s2
  name: s2
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: s2
  template:
    metadata:
      labels:
        app: s2
    spec:
      containers:
      - env:
        - name: JAEGER_ADDR
          value: http://jaeger-service:4318/v1/traces
        image: s2:v1
        imagePullPolicy: Always
        name: s2
      dnsPolicy: ClusterFirst
      restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: s2-service
  name: s2-service
  namespace: default
spec:
  ports:
  - name: s2-port
    port: 20000
    protocol: TCP
    targetPort: 20000
  selector:
    app: s2
  type: NodePort
```
