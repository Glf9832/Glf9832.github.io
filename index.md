# Celery总结

## 一、配置

### 1) 基本配置

schedule_test/celery_config.py
``` python
class BaseConfig(object):
broker_url = 'amqp://username:password@127.0.0.1:5672/' #使用rabbitmq作为消息代理
result_backend = 'redis://:password@127.0.0.1/0' #任务结果存入redis
task_serializer = 'msgpack' #任务序列化和反序列化使用msgpack方案
result_serializer = 'json' #读取任务结果要求性能不高，使用可读性更好的JSON
result_expires = 60*60*24 #任务任务过期时间
accept_content = ['json','msgpack'] #指点接受的内容类型
timezone = 'UTC' #设置时区
enable_utc = True #开启utc
imports = ['schedule.celery_tasks'] #导入任务模块
task_track_started = True #任务跟踪
beat_schedule = {
'test':{
'task':'schedule_test.celery_tasks.test_task_async',
'schedule':timedelta(seconds=10),
}
}

# crontab() 每分钟执行一次
# crontab(minute=0, hour=0) 每天凌晨十二点执行
# crontab(minute='*/15') 每十五分钟执行一次
# crontab(minute='*',hour='*', day_of_week='sun') 每周日的每一分钟执行一次
# crontab(minute='*/10',hour='3,17,22', day_of_week='thu,fri') 每周三，五的三点，七点和二十二点没十分钟执行一次

class DevConfig(BaseConfig):
pass
class TestConfig(BaseConfig):
pass
class ProductionConfig(BaseConfig):
pass

config = DevConfig()
```
schedule_test/celery_app.py
``` python
from celery import Celery
from schedule_test.celery_config import config as CONFIG

celery_app = Celery('test')
celery_app.config_from_object(CONFIG) #导入配置
```

schedule_test/celery_tasks.py
``` python
from schedule_test.celery_app import celery_app

@celery_app.task
def test_task_async():
return 'result info.'
```

### 2) 指定任务队列

schedule_test/celery_config.py
``` python
from kombu import Queue

class BaseConfig(object):
...
task_queues = (
Queue('default',exchange=Exchange('default'),routing_key='task.#'), # 路由键以“task.”开头的消息都进default队列
Queue('web_tasks',exchange=Exchange('web_tasks'),routing_key='web.#'), # 路由键以“web.”开头的消息都进web_tasks队列
Queue('beat_tasks',exchange=Exchange('beat_tasks'),routing_key='beat.#'), #路由键以“beat.”开头的消息都进beat_tasks队列
)
task_default_exchange = 'tasks' # 默认的交换机名字为tasks
task_default_exchange_type = 'topic' # 默认的交换类型为topic
task_default_routing_key = 'task.default' # 默认的路由键是task.default,这个路由键符合上面的default队列
task_routes = {
'schedule_test.celery_tasks.test_task_async':{ # test_task_async任务的消息指定进入beat_tasks队列
'queue':'beat_tasks',
'routing_key':'beat.test_task_async'
}
}
```

### 3) Django与Celery的使用

安装Django-celery-beat和Django-celery-results包

schedule_test/celery_config.py
``` python
class BaseConfig(object):
# result_expires = 60*60*24 # 此项在使用django-celery-results作为backend时无效
beat_scheduler = 'django_celery_beat.schedulers.DatabaseScheduler' # 指定django-celery-beat调度类
result_backend = 'django_celery_results.backends.database:DatabaseBackend' # 指定任务结果使用django-celery-results保存

```

schedule_test/celery_app.py
``` python
import os
from celery import Celery
from schedule_test.celery_config import config as CONFIG

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'project.settings')
celery_app = Celery('test')
celery_app.config_from_object(CONFIG) #导入配置
celery_app.autodiscover_tasks() # 自动发现任务
```

project/__init__.py
``` python
from __future__ import absolute_import, unicode_literals
from schedule_test.celery_app import celery_app
__all__ = ('celery_app',)
```

## 二、管理

### 1) Beat

#### 1、启动beat
``` shell
celery beat -A schedule_test.celery_app -l info
```

### 2) Worker

#### 1、启动worker
``` shell
celery multi start test -A schedule_test.celery_app -l info --pidfile=tmp/celery/celery_%n.pid --logfile=tmp/celery/celery_%n.log
```

#### 2、查看worker启动时的命令
``` shell
celery multi show test
```

#### 3、获取worker的节点名
``` shell
celery multi names test
```

#### 4、停止worker进程
``` shell
celery multi stop test
```

#### 5、重新启动worker
``` shell
celery multi restart test
```

#### 6、杀掉worker进程
``` shell
celery multi kill test
```

## 三、Flower监控

### 1) 配置
schedule_test\flower_config.py
``` python
debug = False
address = '0.0.0.0'
port = 5555 # 默认5555
auto_refresh = True
broker_api = 'http://guest:guest@127.0.0.1:15672/api/' # RabbitMQ management api
max_tasks = 10000 # 内存中保留最大的task数目 默认10000
persistent = True
db = 'tmp/flower/flower'
```

### 2）管理

#### 1、不指定配置文件启动
``` shell
flower -A schedule_test.celery_app --broker=redis://:password@127.0.0.1:6379/0
```

#### 2、指定配置文件启动
``` shell
flower --broker=redis://:password@127.0.0.1:6379/0 --conf=schedule_test/flower_config.py
```

