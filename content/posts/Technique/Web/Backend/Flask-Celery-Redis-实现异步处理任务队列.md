---
title: Flask + Celery + Redis 实现异步处理任务队列
date: 2019-03-31 16:40:16
tags:
  - celery
  - flask
  - python
  - redis
  - task queue
---

最近正在尝试编写网站，其中一个功能是帮助用户从某个 URL 抓取信息 —— 也就是爬虫 。但众所周知，爬虫程序由于 I/O 阻塞通常会消耗较长的时间，无法第一时间对用户请求作出响应 。因此我们会希望让爬虫任务在后台处理，等到执行成功/失败后再将结果返回给用户；而**任务队列**就作为 Web 后端接口与爬虫处理程序之间的一个中介，负责传输任务的具体内容和执行结果 。这篇博客不会具体阐述 Flask Web 开发或 Python 爬虫的相关技术，而将重点聚焦于使用流行的开源异步任务处理框架 `Celery` 实现一个任务队列的基本功能 。

（我才不是出于咕咕咕的负罪感连更博客的）

<!--more-->

## 准备工作

### 安装 Redis

`Redis` （https://redis.io/ ）是一个开源的 Key-Value 数据库，通常运行在内存中，也可以存储可持久化数据 。Redis 拥有极高的性能（110000 read/s，81000 write/s ）与丰富的内置数据结构（包括可以作为队列使用的列表数据结构），近年来变得十分流行 。这里我选择使用 Redis 作为任务队列的消息中间件，也就是使用它来作为任务/结果传输的载体 。

在类 Unix 系统上安装 Redis 相当方便，许多包管理器都提供了一键式安装，也可以去上面提到的官网下载；在 Windows 系统上安装可以试试[这个](https://github.com/MicrosoftArchive/redis) 。我个人的选择是通过 `docker` 安装：

```bash
docker pull redis
docker run --name redis -p 127.0.0.1:6379:6379 -d redis:latest
```

这样一个 Redis 容器实例就在你的宿主机上运行起来了，你可以通过 `redis://localhost:6379` 连接 。如果你需要通过 Python 操作 Redis，你还需要 `pip install redis` 。

完成以上步骤以后你已经可以实现一个简单的任务队列了 。例如：

`producer.py`

```python
import time
import redis

cnt = 1
conn = redis.Redis()

while True:
  conn.lpush('queue', cnt)
  cnt = cnt + 1
  time.sleep(1)
```

`consumer.py`

```python
import datetime
import redis

conn = redis.Redis()

while True:
  info = conn.blpop('queue')
  print('received info: ', info, 'at', datetime.datetime.now())
```

**output**

```
received info:  (b'queue', b'1') at 2019-03-31 18:06:56.335450
received info:  (b'queue', b'2') at 2019-03-31 18:06:57.337337
received info:  (b'queue', b'3') at 2019-03-31 18:06:58.339141
received info:  (b'queue', b'4') at 2019-03-31 18:06:59.341055
received info:  (b'queue', b'5') at 2019-03-31 18:07:00.343051
```

在两个终端里分别先后执行 `consumer.py` 与 `producer.py`，大致就能看到这样的输出了 。

### 安装 Celery

`Redis` 提供了消息中间件，而 `Celery` 的作用就是完善的异步任务管理机制 。事实上 Celery 还可以使用 `RabbitMQ` 等其他产品做为任务传输中间件，也能使用 memcached、许多 SQL 数据库以及很多其他产品来存储执行结果 —— 我们这里就使用 Redis 。

安装 Celery 也很简单：`pip install celery`；但注意如果你使用 Python 3.7 或以上版本，你应该从 [GitHub 仓库](https://github.com/celery/celery/) 安装开发版本 `pip install git+https://github.com/celery/celery.git` 。

## 具体实现

### Flask 项目结构

```
demo
├── __init__.py
└── task.py
```

任务队列主要的功能有两个：创建异步任务与检查任务运行状态 。因此相应地，我们在 `__init__.py` 中创建了 Flask app 与两个路由：`/create` 与 `/get/<task_id>`，为这两个功能提供了 API 接口；在 `task.py` 外中则创建了 Celery app 并定义了一个需要较长时间执行的任务 。

在 demo 文件夹所在的目录下按照如下步骤即可启动 Flask app：

```bash
export FLASK_APP=demo
export FLASK_ENV=development
flask run
```

在 demo 文件夹所在的目录下启动 Celery app（`-A` 指定 app 对象位置，`-l` 指定日志输出等级）：

```
celery worker -A demo.task -l info
```

### 创建 Celery 应用

`task.py`

```python
import time
from celery import Celery

app = Celery('default')
app.conf.update({
  'broker_url': 'redis://localhost:6379',
  'enable_utc': True,
  'result_backend': 'redis://localhost:6379',
  'timezone': 'Asia/Shanghai',
})

@app.task
def long_task(span:int):
  time.sleep(span)
  return True
```

首先是创建 Celery 实例，实例最好命名为 app，因为 Celery 默认从指定模块（这里是 `demo.task`）中加载 `app` 对象来启动 。

其次就是配置 。`broker_url` 指定了任务传输的中间件，`result_backend` 指定了于何处存储任务执行的结果 ；`enable_utc` 与 `timezone` 让 Celery 使用本地时间运行任务 。

最后通过装饰器 `@app.task` 创建任务模板 。这里是定义了一个接受整型参数并阻塞相应秒后返回 `True` 的任务模板（其实就是一个 `callable` 对象），我们通过这个模板来创建实际的任务 。

### Web API 接口

`__init__.py`

```python
from flask import Flask
from .task import long_task

app = Flask(__name__)

@app.route('/create')
def create():
  task_id = long_task.delay(10).task_id
  return task_id

@app.route('/get/<task_id>')
def get(task_id):
  result = long_task.AsyncResult(task_id)
  return result.status
```

-   创建任务

    通过 `long_task.delay` 即可创建异步任务，你只需传入参数，剩下的事 Celery 会替你处理 。上例中这个任务会阻塞 10 秒后返回 `True` 。注意 `long_task.delay` 返回的是一个 `AsyncResult` 对象，这个对象代表了一个异步任务的结果 。它有一系列有用的属性，例如：

    -   `task_id`：此任务的 uuid
    -   `status` / `state`：任务的状态，有 PENDING，FAILED，SUCCESS 等
    -   `result` / `info`：任务执行的返回结果，未完成时为 `None`

    更详细的属性/方法列表可以查看[官方文档](<http://docs.celeryproject.org/en/latest/reference/celery.result.html#celery.result.AsyncResult>) 。这里我们返回了 `task_id` 以便后续查看 。

-   查看任务执行状态

    你可能会想要把 `AsyncResult` 对象保存在 Flask 的 `session` 字典中一边后续调用，但遗憾的是由于 `session` 不是真正的 Python 字典且 `AsyncResult` 不能被直接 JSON 序列化，所以你不能这么做（这涉及到 `session` 的实现原理） 。但你可以保存 `task_id`（因为是字符串），然后再通过`long_task.AsyncResult(task_id)` 来获取 。当然我们此处并没有存进 `session`（用户记不住？管他呢/doge）。

最后，注意每一个任务都应该在完成后释放其占用的资源（通过 `result.forget()`，这里 `result` 也是 `AsyncResult` 对象）。你也应该在配置中设置 `result_expires` 字段，防止有漏网之鱼 。这点我在 demo 中没处理好（懒）。

### 测试结果

按照上面在项目结构中提到的方法分别启动 Flask 与 Celery（当然还有 Redis），然后在命令行通过 `curl` 测试：

```bash
$ curl localhost:5000/create
a59c7e0e-cba6-4b78-87b0-b24dca90c746
$ curl localhost:5000/get/a59c7e0e-cba6-4b78-87b0-b24dca90c746
SUCCESS
```

我手速不够快，所以没捕捉到中间的 PENDING 。Celery 日志会也有类似以下输出：

```
[2019-03-31 19:45:20,092: INFO/MainProcess] Connected to redis://localhost:6379//
[2019-03-31 19:45:20,104: INFO/MainProcess] mingle: searching for neighbors
[2019-03-31 19:45:21,134: INFO/MainProcess] mingle: all alone
[2019-03-31 19:45:21,154: INFO/MainProcess] celery@manjaro ready.
[2019-03-31 19:45:46,264: INFO/MainProcess] Received task: demo.task.long_task[a59c7e0e-cba6-4b78-87b0-b24dca90c746]
[2019-03-31 19:45:56,278: INFO/ForkPoolWorker-4] Task demo.task.long_task[a59c7e0e-cba6-4b78-87b0-b24dca90c746] succeeded in 10.012213802998303s: True
```

任务确实在 10 秒后完成并返回了 True 。

