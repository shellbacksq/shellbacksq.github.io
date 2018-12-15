# Python logging模块
[TOC] 
> 起因: 看到一篇文章[PyCon 2018: 利用logging模块轻松地进行Python日志记录](https://juejin.im/post/5b13fdd0f265da6e0b6ff3dd)(本文也主要参考了这篇文章), 里面有几句话引起了我的注意:
> 1. 开发过程中程序出现了问题,我们可以去debug,但是程序开发完毕,在生产环境中,我们如何追踪可能出现的问题, 这时候就需要日志记录.
> 2. 很多人会使用 print 语句输出一些运行信息，然后再在控制台观察，这样其实是非常不规范的. (其实我就是print流的...)

不难看出logging模块的出现是为了在程序运行时输出运行时信息,**有`bug`时可以定位`bug`,没有`bug`也可以用来跟踪输出**,这在追踪模型输出,评估模型好坏时也特别有用,`automl`就是这个思路.

## 1. 80% USAGE PART

### 1.1 基本语句
可以看出简单使用的话可以这样写:
```python
import logging
# log级别,格式,保存位置
logging.basicConfig(
   level= logging.DEBUG,
   format = '%(asctime)s : %(levelname)s : %(message)s',# 运行时间,日志级别,日志内容.
   filename= "test.log" #要想打印到控制台就不用这句,basicConfig下不能兼得.
)   
logger = logging.getLogger(__name__)
# 下面这些写在代码里面,可以出现在任何想出现的位置.
logger.debug('debug message')
logger.info("info message")
logger.warn('warn message')
logger.error("error message")
logger.critical('critical message')

# 更加典型的使用场景,记录错误和异常.
try:
    result = 10 / 0
except Exception:
    logger.error('Faild to get result', exc_info=True)
logger.info('Finished')
```

一些解释:
- basicConfig 配置了 level 信息和 format 信息,`asctime`,`levelname`,`message`分别代表运行实践,日志级别和日志信息.
- basicConfig更多的参数如:
  - filename：即日志输出的文件名，使用了它,日志信息就不会被控制台打印出来.
  - filemode：这个是指定日志文件的写入方式，有两种形式，一种是 w，一种是 a，分别代表清除后写入和追加写入。
  - format：指定日志信息的输出格式，即上文示例所示的参数，详细参数可以参考： https://docs.python.org/3/library/logging.html?highlight=logging%20threadname#logrecord-attributes
  - datefmt：指定时间的输出格式。
    - 重要的有:
    - %(levelno)s：打印日志级别的数值。
    - %(levelname)s：打印日志级别的名称。
    - %(pathname)s：打印当前执行程序的路径，其实就是sys.argv[0]。
    - %(filename)s：打印当前执行程序名。
    - %(funcName)s：打印日志的当前函数。
    - %(lineno)d：打印日志的当前行号。
    - %(asctime)s：打印日志的时间。
    - %(message)s：打印日志信息。

  - style：如果 format 参数指定了，这个参数就可以指定格式化时的占位符风格，如 %、{、$ 等。
  - level：指定日志输出的类别，程序会输出大于等于此级别的信息。
  - stream：在没有指定 filename 的时候会默认使用 StreamHandler，这时 stream 可以指定初始化的文件流。
  - handlers：可以指定日志处理时所使用的 Handlers，必须是可迭代的。
- 接下来声明了一个 Logger 对象，它就是日志输出的主类,在初始化的时候我们传入了模块的名称，这里直接使用 `__name__` 来代替了，就是模块的名称.
- 运行结果
  ```python
  2018-06-03 13:42:43,526 - __main__ - DEBUG - debug message
  2018-06-03 13:42:43,526 - __main__ - INFO - debug message
  2018-06-03 13:42:43,526 - __main__ - WARNING - Warning exists
  2018-06-03 13:42:43,526 - __main__ - ERROR - Warning exists
  2018-06-03 13:42:43,526 - __main__ - CRITICAL - Warning exists

   ```
---


### 1.2 如何在程序中使用

**不能做到的**
- 一边输出到控制台,一边写入到日志文件.

## 2. 深入理解logging模块
所以如何更加专业地使用logging模块呢?
**我们先定下几个目标,然后逐渐实现它:**
1. 一边输出到控制台,一边写入到日志文件,将不同级别的日志发送到不同的地方.
2. 实现可配置的日志,可以有一个模板,我们不必从头开始
3. 自动化的日志搜集和分析


### 2.1 Python中logging模块的结构

现在回到问题的一开始,如何设计这样一个模块它要能完成如下任务:
- 它要包含一定运行时信息,包括时间,代码位置等等.
- 它要能保存到本地或者某台服务器上.
- 它最好速度要快,不影响运行
- 它最好分为一定等级,这样便于定位不同信息.
  
粗略能想到这些,那么`python`中的`logging`模块是怎样实现的呢?
在 https://docs.python.org/3/library/logging.html 中有以下几个主要的对象:
- **Logger**     主类 https://docs.python.org/3/library/logging.html#logger-objects
- **Handler**    指定打印或存放位置  https://docs.python.org/3/library/logging.html#handler-objects
- **Formatter**  指定日志的格式 https://docs.python.org/3/library/logging.html#formatter-objects
  
具体来说`Logger`用来创建一个日志记录的对象,它是所有操作的入口,`Handler`和`Formatter`对象最后都会传给它. 而在`Logger`创建一个日志对象后, 使用`Handler`来指定日志保存的位置,可以是控制台,本地文件,远程服务器,甚至是某个邮箱. 设置好保存的位置,接下来就可以定制日志的样式了,这里很重要,因为我们最终看到的日志,就是在这里设定的, 我们指定`Formatter`来定制样式.

**整个使用流程见下面这个图**:
![logging]({{site.url}}/assets/logging flow.png)
![](../assets/logging&#32;flow.png)

一个分开`Logger`,`Handler`,`Formatter`使用的简单例子.

```python
logging.basicConfig和logger后面再设置有什么区别.

import logging

logger = logging.getLogger(__name__) # Logger
logger.setLevel(level=logging.INFO)
handler = logging.FileHandler('output.log') # Handler
formatter = logging.Formatter\
('%(asctime)s - %(name)s - %(levelname)s - %(message)s') # Formatter

handler.setFormatter(formatter)
logger.addHandler(handler)

logger.info('This is a log info')
logger.debug('Debugging')
logger.warning('Warning exists')
logger.info('Finish')

```

结合上面的流程图和上面代码中的这两行:
```python
handler.setFormatter(formatter)
logger.addHandler(handler)
```
可以看出`Formatter`传给`Handler`,`Handler`带着自身和`Formatter`的信息又传给了`Logger`, 最后完成了一个完整的日志记录过程.
那么我们来一个一个分析`Formatter`,`Handler`,`Logger`.

#### 2.1.1 [Logger](https://docs.python.org/3/library/logging.html#logger-objects)
`Logger`一般不会单独使用,而是使用模块级别的函数`logging.getLogger(name)`,常用的记录器对象的方法分为两类：配置和发送消息。
- `Logger.setLevel()`
- `Logger.debug()`....
日志级别如下:

|Level	|Numeric value|
|:---|:---|
|CRITICAL|	50|
|ERROR	|40|
|WARNING|	30|
|INFO	2|0|
|DEBUG	|10|
|NOTSET|	0|

还有一些方法如:
- Logger.addHandler()和Logger.removeHandler()从记录器对象中添加和删除处理程序对象。处理器详见Handlers。
- Logger.addFilter()和Logger.removeFilter()从记录器对象添加和删除过滤器对象。

#### 2.1.2 [Handler](https://docs.python.org/3/library/logging.html#handler-objects)
- SteamHandler             日志输出到流   #logging自带
- FileHandler              日志输出到文件 #logging自带
- RotatingFileHandler       按照大小自动分割日志文件   #超出最大容量时
- TimedRotatingFileHandler 按照时间自动分割日志文件 
- SocketHandler            远程输出日志到TCP/IP sockets
- DatagramHandler          远程输出日志到UDP sockets
- SMTPHandler              远程输出日志到邮件地址
- MemoryHandler            日志输出到内存中的指定buffer
- HTTPHandler              通过"GET"或者"POST"远程输出到HTTP服务器。

```python
import logging
from logging.handlers import HTTPHandler
import sys

logger = logging.getLogger(__name__)
logger.setLevel(level=logging.DEBUG)

# StreamHandler
stream_handler = logging.StreamHandler(sys.stdout)
stream_handler.setLevel(level=logging.DEBUG)
logger.addHandler(stream_handler)

# FileHandler
file_handler = logging.FileHandler('output.log')
file_handler.setLevel(level=logging.INFO)
formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
file_handler.setFormatter(formatter)
logger.addHandler(file_handler)

# HTTPHandler
http_handler = HTTPHandler(host='localhost:8001', url='log', method='POST')
logger.addHandler(http_handler)

# SMTPHandler
smtp_handler=SMTPHandler(mailhost=("smtp.163.com", 25), fromaddr='sunqiang42@163.com',
                                  toaddrs=['sq11235@163.com',],
                                  subject="logging to email test",
                                  credentials=('sunqiang42@163.com', '***'),
                                  )
logger.addHandler(smtp_handler)

# Log
logger.info('This is a log info')
logger.debug('Debugging')
logger.warning('Warning exists')
logger.info('Finish')
```
上述代码在运行后会在:
- 控制台打印日志信息
- 本地保存日志信息
- HTTP Server接收到日志信息
- Email 接受日志信息

这里我们就完成了几个任务:
1. 同时输出不同格式的信息给不同的对象
2. 可以同时拥有多个logger对象,不同的信息内容给不同的对象,例如只有ERROR信息才发邮件.
3. 另外每个 Handler 还可以设置 level 信息，最终输出结果的 level 信息会取 Logger 对象的 level 和 Handler 对象的 level 的交集。

#### 2.1.3 [Formatter](https://docs.python.org/3/library/logging.html#formatter-objects)

Formatter和前面[基本语句](#1.1)差不多,不在赘述.

### 2.2 配置共享

我们希望在一个地方定义好了logging配置,然后在别的模块里复用就行了.比如说在`main.py`函数里定义好了level,formatter,handler等信息,然后如果`main.py`中调用别的模块,在该模块里面也能使用`main.py`中的logger配置.

```python
# main.py
import logging
import core

logger = logging.getLogger('main')
logger.setLevel(level=logging.DEBUG)

# Handler
handler = logging.FileHandler('result.log')
handler.setLevel(logging.INFO)
formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
handler.setFormatter(formatter)
logger.addHandler(handler)

logger.info('Main Info')
logger.debug('Main Debug')
logger.error('Main Error')
core.run()# main中可能要调用core的信息
```
这就设置好了模板.然后在core中:
```python
# core.py
import logging
logger = logging.getLogger('main.core')# 在core.py的开头只需要写上这两句就可以了.

def run():
    logger.info('Core Info')
    logger.debug('Core Debug')
    logger.error('Core Error')
```
### 2.3 文件配置
把配置写到日志配置文件中去:(有.conf和.yaml多种方式,看起来还是.yaml更加直观.)
```yaml
# logging.yaml
formatters:
  brief:
    format: "%(asctime)s - %(message)s"
  simple:
    format: "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
handlers:
  console:
    class : logging.StreamHandler
    formatter: brief
    level   : INFO
    stream  : ext://sys.stdout
  file:
    class : logging.FileHandler
    formatter: simple
    level: DEBUG
    filename: debug.log
  error:
    class: logging.handlers.RotatingFileHandler
    level: ERROR
    formatter: simple
    filename: error.log
    maxBytes: 10485760
    backupCount: 20
    encoding: utf8
loggers:
  main.core:
    level: DEBUG
    handlers: [console, file, error]
root:
  level: DEBUG
  handlers: [console]
```
然后在程序中调用
```python
import logging
import logging.config

logging.config.fileConfig('logging.conf')
logger = logging.getLogger(__name__)
logger.debug('debug message')
logger.info("info message")
logger.warn('warn message')
logger.error("error message")
logger.critical('critical message')
```

可以看出日志的配置更加灵活,几句话概括就是不同的handler可以使用不同的formmter,不同的
logger可以使用不同的handler.

---
然后我们可以这样使用:
```python
# main_conf.py
import logging
import core
import yaml
import logging.config
import os


def setup_logging(default_path='config.yaml', default_level=logging.INFO):
    path = default_path
    if os.path.exists(path):
        with open(path, 'r', encoding='utf-8') as f:
            config = yaml.load(f)
            logging.config.dictConfig(config)
    else:
        logging.basicConfig(level=default_level)


def log():
    logging.debug('Start')
    logging.info('Exec')
    logging.info('Finished')


if __name__ == '__main__':
    yaml_path = 'config.yaml'
    setup_logging(yaml_path)
    log()
    core.run()
```

## 3. loguru
https://github.com/Delgan/loguru

**未完待续**!  


参考资料:
- [所有 Python 程序员必须要学会的「日志」记录。
](https://mp.weixin.qq.com/s/soddu10v40aDYpm_P6U3IQ)
- [使用logging管理爬虫
](https://zhuanlan.zhihu.com/p/32969467)
- [普通程序员怎么理解日志系统
](http://www.algorithmdog.com/loggingsystem)
- [PyCon 2018: 利用logging模块轻松地进行Python日志记录](https://juejin.im/post/5b13fdd0f265da6e0b6ff3dd#heading-3)
- [logging — Logging facility for Python¶
](https://docs.python.org/3/library/logging.html#handler-objects)
- [loguru](https://github.com/Delgan/loguru/blob/master/README.rst)
- [日志的艺术（The art of logging）
](http://blog.jobbole.com/113413/?utm_source=blog.jobbole.com&utm_medium=relatedPosts)

