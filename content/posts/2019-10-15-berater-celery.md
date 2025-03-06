---
title: "Celery 集成"
date: 2019-10-15T15:11:31+08:00
description: 在微信服务号后端项目中集成 Celery
---

这篇聊聊在微信服务号后端项目中集成 Celery。

## 起因

某天逛 V2ex 看到一个关于微信 access_token 的[讨论](https://www.v2ex.com/t/609041)。首先官方文档的介绍如下：

>建议公众号开发者使用中控服务器统一获取和刷新access_token，其他业务逻辑服务器所使用的access_token均来自于该中控服务器，不应该各自去刷新，否则容易造成冲突，导致access_token覆盖而影响业务。

之前，没有考虑到多进程的问题，`AccessToken` 如下：

```python
class AccessToken:
    token = ''
    last_time = 0
    
    @static_method
    def get():
        if time.time() - last_time < 7200:
            return AccessToken.token
        t = request_api_and_get_access_token()
        AccessToken.token = t
        return t
```

由于在生产环境下使用 gunicorn 启动了多个进程，上述逻辑出现问题。

基于原有形式考虑到的合理的方法是：将 token 与过期时间存储于 redis，进程每次尝试获取 token 时，首先阻塞式获取 redis 锁，若发现过期，即进行更新。

但为了尽量避免锁，且方便之后的业务扩展（即之前构想在 Flask 项目中也可以进行手动调用），当然同时也是为了了解新知识，决定使用官方推荐的方法。

在我们的单机环境下，中控服务器显然不太现实。一开始打算使用 Flask 周边的一些任务调度插件进行处理，常用的如 APScheduler，但 APScheduler 的配置方式我不是很喜欢，且它只能支持定时任务，灵活性太小，而 Celery 的功能较为全面，支持消息队列的形式，因此决定使用 Celery。

## 使用

### Demo

首先有一个定时任务，就是定时刷新 access_token 至 redis，这一时间间隔应略小于 2 小时。

```python
token_cache = MemoryCache('access_token', 7200)

cron = Celery(
    'tasks',
    broker=getenv('CELERY_BROKER_URL', 'redis://redis:6379/1'),
    backend=getenv('CELERY_RESULT_BACKEND', 'redis://redis:6379/1'))

@cron.on_after_configure.connect
def setup_periodic_tasks(**_):
    cron.add_periodic_task(2 * 60 * 60 - 5 * 60, refresh_access_token.s())

@cron.task
def refresh_access_token():
    token = get_access_token_directly(getenv('API_KEY'), getenv('API_SECRET'))
    print(f'token: {token}')
    if token:
        token_cache.set('', token=token)
    return token
```

 需要注意的地方有：

- 为了避免可能的问题， celery broker & backend 选择 redis 的 database 1，而非默认的 database 0；
- `cron.task`  装饰器用来定义一个任务，因为只有一个任务，且默认函数名即作为 name，故不指定其参数；
- 配置定时任务，可以使用配置文件的方式，但是由于我们的任务比较固定，且使用 docker 部署，因此配置文件的格式其实是没有什么优势的；而我本人又比较喜欢以代码格式进行配置，因此使用 `add_periodic_tasks` 函数进行配置定时任务。
- 至于定时任务的时间设置，默认的是使用浮点数进行配置，也可以使用 `crontab` 等方式，不过这些复杂的功能在这里使用并不会改进功能甚至不能实现目前的功能，而只是徒增学习成本，因此使用默认的方式。

同时，这一文件虽然在项目文件夹下，但 Celery 并不在 Flask 中启动（虽然有 flask_celery 库，但他的抽象功能比较弱，官方也并没有推荐这一插件，不进行考虑）。 相反，是在命令行中进行，具有比较高的独立性，在 docker-compose 中也将它独立出来，比较方便单独重启任务和查看日志。

### 调用

Flask 可以将该文件作为模块 import，[celery.task reference](http://docs.celeryproject.org/en/master/reference/celery.app.task.html) 中介绍了具体的调用方式和参数。

### 部署

目前 celery==4.3.0, kombu==4.6.5 版本会遇到 [InconsistencyError](https://github.com/celery/kombu/issues/236)，将 kombu 降级至 4.5.0 后解决。在 supervisord 配置文件中配置执行 command:

```bash
celery -A berater.misc.tasks.cron worker -B -E
```

monitor 决定使用 celery-flower，进行配置后可以有 dashboard 进行监控。

需要注意：

- Flower 最好在 worker 启动后再启动，避免 inspect 失败，会导致 dashboard 功能缺失；
- Flower dashboard 能够管理任务队列，需要进行[安全配置](https://flower.readthedocs.io/en/latest/auth.html)，最简单的使用环境变量配置 basic auth 即可；

- 对于 nginx 反向代理，Flower 支持设置 [url prefix](https://flower.readthedocs.io/en/latest/config.html#url-prefix)。command 如下：

```bash
celery -A berater.misc.tasks.cron flower --basic_auth=$CELERY_AUTH --url_prefix=flower
```

