## 升级总体方案

- 去除blueapps包中的celery中的依赖
- 由于djcelery仅支持到celery3x版本，升级celery4x版本，使用django-celery-beat和django-celery-result包代替djcelery功能
- 升级主要存在的问题在于celery4x的时间流转与celery3x的不同，导致定时任务和异步任务执行异常，后续章节会详细介绍，[老用户升级](#老用户升级注意事项)和[新用户的使用注意事项](#新用户使用注意事项)
- celery4x解决了与async关键字的冲突，升级后可以使用python3.7及以上版本（目前仅测试到3.7.4）

## 升级后配置文件更改

- 升级后默认`USE_TZ=True`
- 替换djcelery依赖为django_celery_beat和django_celery_results

升级前：conf/default.py

```python 

...
# 国际化配置
LOCALE_PATHS = (os.path.join(BASE_DIR, 'locale'),)  # noqa

TIME_ZONE = 'Asia/Shanghai'
LANGUAGE_CODE = 'zh-hans'
....

if IS_USE_CELERY:
    INSTALLED_APPS = locals().get('INSTALLED_APPS', [])
    import djcelery
    INSTALLED_APPS += (
        'djcelery',
    )
    djcelery.setup_loader()
    CELERY_ENABLE_UTC = False
    CELERYBEAT_SCHEDULER = "djcelery.schedulers.DatabaseScheduler"

```

升级后：conf/default.py

```python

...
# 国际化配置
LOCALE_PATHS = (os.path.join(BASE_DIR, 'locale'),)  # noqa

USE_TZ = True
TIME_ZONE = 'Asia/Shanghai'
LANGUAGE_CODE = 'zh-hans'
....

if IS_USE_CELERY:
    INSTALLED_APPS = locals().get('INSTALLED_APPS', [])
    INSTALLED_APPS += (
        'django_celery_beat',
        'django_celery_results'
    )
    CELERY_ENABLE_UTC = False
    CELERYBEAT_SCHEDULER = "django_celery_beat.schedulers.DatabaseScheduler"

```

## celery4新特性

### 系统级别的更新

1. 不再对windows 平台提供支持。windwos平台目前主要发现两个主要的问题，其他的基本功能都是正常的。

   - 一个是需要加入 eventlet才可以使用。
   - windwos平台下无法同时启动多个worker，主要原因是启动多个worker 依赖resource这个库，这个库只支持linux 和 unix 平台。

2. 不再支持 Python 2.6 以及 python 3.3

3. 不再支持Jpython。

4. **celery4.x 支持的最低Django 版本为1.8,官方建议从1.9开始。**

5. celery4 将会是最后一个支持python2的版本，celery5最低支持的python版本为python3.5

   ### 功能级别的更新

6. 使用了全新的设置名

   ，具体表现在celery4.x版本的配置缩写全都为小写，但是为了考虑到对于celery3.x 版本的支持，所以大写的配置名仍然是可以使用的.例如:

   ```
   CELERY_TIMEZONE = 'Asia/Shanghai' # 默认时区是UTC 旧
   timezone = 'Asia/Shanghai' #默认时区是UTC 新
   ```

   关于celery大小写的升级，celery 提供了全新的命令来批量去迁移配置：

   ```
   celery upgrade settings [filename]
   ```

   具体的映射内容官方文档中有说明：
   https://docs.celeryproject.org/en/4.0/userguide/configuration.html#conf-old-settings-map

7. json 现在作为默认的序列化器。

   日常使用会出现使用json无法序列化的情况，如果仍然需要使用pickle作为序列化器，需要增加如下配置:

   ```python
   task_serializer = 'pickle'
   result_serializer = 'pickle'
   accept_content = {'pickle'}
   ```

8. 使用Task作为基类的任务不会被自动注册了

   ，新版本的celery4.x 需要使用装饰器装饰Task 才会被自动注册。

   ```python
   # 创建普通任务
   @app.task()
   def add(x, y):
       print("计算2个值的和: %s %s" % (x, y))
       return x + y
   ```

9. **优先级发生了改变，现在优先级的顺序为 0-9 依次升高**。主要是为了与AMQP 中的工作方式保持一致。

10. **命令行参数发生了改变**。celeryd -> celery worker , celerybeat-> celery beat,celeryd-multi-> celery multi

11. Auto-discover 目前支持 Django app 配置

12. 支持RabbitMQ 优先级.

13. 新增了broker_read_url和broker_write_url设置选项，这样就可以为用于消费/发布的连接提供单独的代理url。

14. 增加了对RabbitMQ扩展的支持.

    ```python
    # 以浮动秒为单位设置队列过期时间。
    Queue(expires=20.0)
    # 设置队列消息实时浮动秒。
    Queue(message_ttl=30.0)
    # 将队列最大长度（消息数）设置为 int。
    Queue(max_length=1000)
    # 将队列最大长度（消息大小总计（以字节为单位）设置为 int。
    Queue(max_length_bytes=1000)
    # 将队列声明为基于消息字段路由消息的优先级队列。priority
    Queue(max_priority=10)
    ```

15. 对`Amazon SQS transport` `Apache QPid transport` 提供了支持。

16. 对redis哨兵提供了支持。

    ```
    sentinel://0.0.0.0:26379;sentinel://0.0.0.0:26380/...
    ```

17. 新的定时任务写法。

    ```python
    from blueapps.core.celery.celery import app
    from django_celery_beat.tzcrontab import TzAwareCrontab
    
    
    @app.on_after_configure.connect
    def setup_periodic_tasks(sender, **kwargs):
        # Calls test('hello') every 10 seconds.
        sender.add_periodic_task(10.0, test.s('hello'), name='add every 10')
    
        # Calls test('world') every 30 seconds
        sender.add_periodic_task(30.0, test.s('world'), expires=10)
    
        # Executes every Monday morning at 7:30 a.m.
        sender.add_periodic_task(
            crontab(hour=7, minute=30, day_of_week=1,tz=timezone.get_current_timezone()),
            test.s('Happy Mondays!'),
        )
    
    @app.task
    def test(arg):
        print(arg)
    ```

18. AsyncResult 新增回调方法then ，需要结合gevent模块使用。

    ```python
    import gevent.monkey
    monkey.patch_all()
    
    import time
    from blueapps.core.celery.celery import app
    
    @app.task
    def add(x, y):
        return x + y
    
    def on_result_ready(result):
        print('Received result for id %r: %r' % (result.id, result.result,))
    
    add.delay(2, 2).then(on_result_ready)
    
    time.sleep(3)  # run gevent event loop for a while.
    ```

19. 支持redis ssl 链接方式

### 不再支持的部分

1. 移除了celery.task.http，emails.app.mail_admins 和 celery.contrib.batches
2. 不再支持使用 Django ORM, SQLAlchemy, CouchDB, IronMQ, Beanstalk 作为 broker。
3. **不再支持rabbitMQ 作为backend储存结果。**

## 老用户升级注意事项

- `USE_TZ=False`的老用户升级后，使用MysSQL backend时，会存在DB中无法写入带有时区的配置内容，具体报错为`ValueError: MySQL backend does not support timezone-aware datetimes when USE_TZ is False`

  解决方案：

  - 方案1（推荐）：在config/default.py文件中添加`USE_TZ = True`开启django时区
  - 方案2：在config/default.py中添加`DJANGO_CELERY_BEAT_TZ_AWARE = False`关闭celery时区感知

- Windows老用户来说，celery4.4已经不支持Windows平台，无法启动多worker，只能通过`-p eventlet`启动单worker，evenlet包需通过pip安装后使用

  ```python
  # eventlet安装命令
  pip install eventlet==0.26.1
  # windows用户启动worker命令
  celery -A blueapps.core.celery worker -l info -P eventlet
  ```

- djcelery不支持celery4.4，升级后通过django-celery-beat和django-celery-result代替djcelery的功能，因此djcelery提供的celery启动命令已经不支持，新的beat，worker启动命令为

  ```python
  # beat启动命令
  celery -A blueapps.core.celery beat -l info
  # worker启动命令，windows用户请加上 -P eventlet参数
  celery -A blueapps.core.celery worker -l info 
  ```
  
  > 注意：这里`-A blueapps.core.celery`用于指定使用开发框架内部封装好的celery app，可用于加载/conf/default.py中的配置等功能，如果配置使用其他app，会导致无法加载配置及task的自动发现

### 升级后异步任务需要修改的配置

- `T.delay(arg, kwarg=value)`配置：升级后无影响

- `apply_async`中的`eta`参数中使用`datetime.datetime.now()`配置的异步任务，由于没有时区在新版本中会视作UTC时间执行，同理`datetime()`生成的时间对象如果没有时区信息，也会被当作UTC时间执行

  例如，所在时区为Asia/Shanghai，当前时间为12:00+0800，会被当作成12:00+00:00执行，即延迟8小时候执行
  解决方案：

  - 使用`django.utils.timezone.now()`代替`datetime.datetime.now()`获取当前时间

  - 使用`django.utils.timezone.make_aware()`使得`datetime`生成时间对象携带时区信息

  ```python
  # 配置用户当前时间12:00+08:00，延迟当前时间2s执行，
  # 错误配置：由于没有时区，使用UTC时区，实际会到12:00+00：00，延迟2s执行
  T.apply_async(eta=datetime.datetime.now() + timedelta(seconds=2))
  # 正确配置：
  T.apply_async(eta=django.utils.timezone.now() + timedelta(seconds=2))
  
  
  # 配置用户2020年9月21日12:02+08:00执行
  # 错误配置：由于没有时区，使用UTC时区，实际会在2020年9月21日12:02+00:00执行
  T.apply_async(eta=datetime.datetime(2020, 9, 21, 12, 0) + timedelta(seconds=2))
  # 正确配置：
  T.apply_async(eta=django.utils.timezone.make_aware(datetime.datetime(2020, 9, 16, 17, 38)) + timedelta(seconds=2))
  ```

### 定时任务配置

- 使用`crontab`配置的定时任务，由于没有时区，会按照`CELERY_TIMEZONE`配置的时区（默认配置UTC）指定定时任务，

  解决方案：使用`TzAwareCrontab`配置，添加`tz=timezone.get_current_timezone()`参数，即可以用户所在时区执行定时任务

  ```python
  from celery.task import periodic_task
  from django.utils import timezone
  from celery.schedules import crontab
  from django_celery_beat.tzcrontab import TzAwareCrontab
  
  # 配置用户所在时区每天10:00执行定时任务
  # 错误配置：由于没有配置定时任务时区，使用UTC时区，实际会在10:00+00:00执行
  @periodic_task(run_every=celery.schedule.crontab(minute="0", hour="10",tz=timezone.get_current_timezone()))
  def crontab_timezone():
      logger.info("test timezone")
      return True
  
  # 正确配置
  @periodic_task(run_every=django_celery_beat.tzcrontab.TzAwareCrontab(minute="0", hour="10",tz=timezone.get_current_timezone()))
  def crontab_timezone():
      logger.info("test timezone")
      return True
  ```

- 升级后定时任务`CrontabSchedule`表新增`timezone`字段，创建定时任务时，可以通过指定`timezone`字段的时区来配置定时任务执行所在的时区

  
  示例代码：home_application/celery_utils.py
  
  ```python
  from django.utils import timezone
  from django_celery_beat.models import CrontabSchedule, PeriodicTask
  # 用户实时添加定时任务
  def add_period_task(task, name=None, minute='*', hour='*',
                      day_of_week='*', day_of_month='*',
                      month_of_year='*', args="[]", kwargs="{}", tz=None):
      """
      @summary: 添加一个周期任务
      @param task: 该task任务的模块路径名, 例如celery_sample.crontab_task
      @param name: 用户定义的任务名称, 具有唯一性
      @note: PeriodicTask有很多参数可以设置，这里只提供简单常用的
      """
      cron_param = {
          'minute': minute,
          'hour': hour,
          'day_of_week': day_of_week,
          'day_of_month': day_of_month,
          'month_of_year': month_of_year,
          'timezone': timezone.get_current_timezone() if tz is None else tz
      }
      if not name:
          name = task
      try:
          cron_schedule = CrontabSchedule.objects.get(**cron_param)
      except CrontabSchedule.DoesNotExist:
          cron_schedule = CrontabSchedule(**cron_param)
          cron_schedule.save()
      try:
          PeriodicTask.objects.create(
              name=name,
              task=task,
              crontab=cron_schedule,
              args=args,
              kwargs=kwargs,
          )
      except Exception as e:
          return False, '%s' % e
      else:
          return True, ''
  
  # 用户通过定时任务id编辑定时任务
  def edit_period_task_by_id(task_id, minute='*', hour='*',
                             day_of_week='*', day_of_month='*',
                             month_of_year='*', args="[]", kwargs="{}", tz=None):
      """
      @summary: 更新一个周期任务
      @param name: 用户定义的任务名称, 具有唯一性
      """
      try:
          period_task = PeriodicTask.objects.get(id=task_id)
      except PeriodicTask.DoesNotExist:
          return False, 'PeriodicTask.DoesNotExist'
      cron_param = {
          'minute': minute,
          'hour': hour,
          'day_of_week': day_of_week,
          'day_of_month': day_of_month,
          'month_of_year': month_of_year,
          'timezone': timezone.get_current_timezone() if tz is None else tz
      }
      try:
          cron_schedule = CrontabSchedule.objects.get(**cron_param)
      except CrontabSchedule.DoesNotExist:
          cron_schedule = CrontabSchedule(**cron_param)
          cron_schedule.save()
      period_task.crontab = cron_schedule
      period_task.args = args
      period_task.kwargs = kwargs
      period_task.save()
      return True, ''
  ```
  
  hoeme_application/views.py
  
  ```python
  
  def save_task(request):
      """
      创建/编辑周期性任务 并 运行
      """
      tz = request.POST.get('tz')
      if tz:
          timezone.activate(tz)
      periodic_task_id = request.POST.get('periodic_task_id', '0')
      params = request.POST.get('params', {})
      try:
          params = json.loads(params)
          add_param = params.get('add_task', {})
      except Exception:
          msg = u"参数解析出错"
          logger.error(msg)
          result = {'result': False, 'message': msg}
          return JsonResponse(result)
      task_args1 = add_param.get('task_args1', '0')
      task_args2 = add_param.get('task_args2', '0')
      # 任务参数
      try:
          task_args1 = int(task_args1)
          task_args2 = int(task_args2)
          task_args_list = [task_args1, task_args2]
          task_args = json.dumps(task_args_list)
      except Exception:
          task_args = '[0,0]'
          logger.error(u"解析任务参数出错")
      #  周期参数
      minute = add_param.get('minute', '*')
      hour = add_param.get('hour', '*')
      day_of_week = add_param.get('day_of_week', '*')
      day_of_month = add_param.get('day_of_month', '*')
      month_of_year = add_param.get('month_of_year', '*')
      # 创建周期任务时，任务名必须唯一
      now = int(time.time())
      task_name = "{}_{}".format(TASK, now)
      if periodic_task_id == '0':
          # 创建任务并运行
          res, msg = add_period_task(TASK, task_name, minute, hour,
                                     day_of_week, day_of_month,
                                     month_of_year, task_args)
      else:
          # 修改任务
          res, msg = edit_period_task_by_id(periodic_task_id, minute, hour,
                                            day_of_week, day_of_month,
                                            month_of_year, task_args)
      return JsonResponse({'result': res, 'message': msg})
  
  ```

### DB数据迁移

注意：

- 由于升级后djcelery被废弃，使用django-celery-beat和django-celery-results代替，因此需要数据表迁移
- 升级后无法迁移的表说明
  - djcelery包中TaskState表仅用于记录tasks任务状态的，不影响task任务的执行，由于django-celery-beat包中该表结构变动过大，因此无法迁移
  - djcelery包中WorkerState表用于记录celery worker状态，在django-celery-beat包中被移除
- 迁移命令的执行时机应该在celery4，django-celery-beat，django-celery-results包安装并配置好，并且执行migrate命令之后，例如，配置到bin/postcompile中
- 迁移前会检查目标数据库是否为空，不为空则迁移失败
- 如果是带时区迁移，旧的crontab配置依然为无时区时间，如`crontab(minute="57", hour="10")`，beat重启后会以`CELERY_TIMEZONE`配置的时区刷新crontab表中的timezone，可能会导致定时任务时区执行问题，因此需要修改crontab为TzAwareCrontab，示例代码[见定时任务配置小节](#定时任务配置)


迁移命令

```python
python manage.py migrate_from_djcelery 
# 可选参数: -tz  指定之前celery运行的时区，不指定默认使用UTC时区 例如：-tz Asia/Shanghai
```

**社区版和企业版用户注意**，请使用如下migrations文件在迁移线上数据库

使用方法：

1. 将下面文件放到任意app的migrations文件夹下，如：home_application/migrations/
2. 迁移命令：执行`python manaage.py migrate`应用该migration文件即可
3. 通过第11行handle函数的tz参数指定指定之前celery运行的时区

```python
# migrations文件
# -*- coding: utf-8 -*-
from django.db import migrations

from blueapps.contrib.bk_commands.management.commands import migrate_from_djcelery


def migrate_db_from_djcelery(*arg, **kwargs):
    migrate_command = migrate_from_djcelery.Command()
    #修改tz指定时区参数
    migrate_command.handle(tz='Asia/Shanghai')


class Migration(migrations.Migration):
    # dependencies中填写最后一个执行的migrations
    dependencies = []

    operations = [
        migrations.RunPython(migrate_db_from_djcelery),
    ]
```

## 新用户使用注意事项

- 默认的时区配置

  ```python
  # default.py
  TIME_ZONE = 'Asia/Shanghai'
  USE_TZ=True
  CELERY_ENABLE_UTC = False
  ```

- beat和worker启动命令

  ```python
  # beat启动命令
  celery -A blueapps.core.celery beat -l info
  # worker启动命令，windows用户请加上 -P eventlet参数
  celery -A blueapps.core.celery worker -l info
  ```

  > 注意：
  >
  > 1. 这里`-A blueapps.core.celery`用于指定使用开发框架内部封装好的celery app，可用于加载/conf/default.py中的配置等功能，如果配置使用其他app，会导致无法加载配置及task的自动发现

  **社区版和企业版用户注意**，由于PaaS平台的限制，只能使用下面的启动方式

  ```python
  # worker启动命令
  python manage.py celery worker -l info
  # worker启动命令
  python manage.py celery beat -l info
  ```

  > 注意：
  >
  > 1. 这种启动命令是从djcelery中迁移过来，
  > 2. 由于django orm只能允许单线程操作数据库句柄，通过`python manage.py`启动的worker与django不是同一线程因此无法修改
  > 3. 使用了`patch_thread_ident`的猴子补丁，用于patch掉django校验orm多线程操作数据的逻辑，会将worker的线程ID
  
- 异步任务配置

  ```python
  # 配置用户当前时间12:00+08:00，延迟当前时间2s执行，
  T.apply_async(eta=django.utils.timezone.now() + timedelta(seconds=2))
  
  
  # 配置用户2020年9月21日12:02+08:00执行
  T.apply_async(eta=django.utils.timezone.make_aware(datetime.datetime(2020, 9, 16, 17, 38)) + timedelta(seconds=2))
  ```

- 定时任务配置

  ```python
  from celery.task import periodic_task
  from django.utils import timezone
  from django_celery_beat.tzcrontab import TzAwareCrontab
  
  # 配置用户所在时区每天10:00执行定时任务
  @periodic_task(run_every=django_celery_beat.tzcrontab.TzAwareCrontab(minute="0", hour="10",tz=timezone.get_current_timezone()))
def crontab_timezone():
      logger.info("test timezone")
      return True
  ```
  
  

