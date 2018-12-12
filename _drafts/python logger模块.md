# Python logging模块
> 起因: 看到一篇文章[PyCon 2018: 利用logging模块轻松地进行Python日志记录](https://juejin.im/post/5b13fdd0f265da6e0b6ff3dd), 里面有几句话引起了我的注意:
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
   format = '%(asctime)s : %(levelname)s : %(message)s',
   filename= "test.log"
)   
# 下面这些写在代码里面
logging.debug('debug message')
logging.info("info message")
logging.warn('warn message')
logging.error("error message")
logging.critical('critical message')
```
或者再复杂一点把配置写到日志配置文件中去:
```python
# logging.conf
[loggers]
keys = root

[handlers]
keys = logfile

[formatters]
keys = generic

[logger_root]
handlers = logfile

[handler_logfile]
class = handlers.TimedRotatingFileHandler
args = ('test.log', 'midnight', 1, 10)
level = DEBUG
formatter = generic

[formatter_generic]
format = %(asctime)s %(levelname)-5.5s [%(name)s:%(lineno)s] %(message)s
```
然后在程序中调用
```python
import logging
import logging.config

logging.config.fileConfig('logging.conf')

logging.debug('debug message')
logging.info("info message")
logging.warn('warn message')
logging.error("error message")
logging.critical('critical message')
```

### 1.2 如何在程序中使用


## 2. 深入理解logging模块
所以如何使用logging模块呢? 一般来说一个工具从**主题思想到具体使用**中间有个桥梁叫**设计思路**. 而所有的这些东西我们都可以从 **存在性**,**唯一性**,这两个思维层面去考量. 存在性就是为什么我们需要它,当然就是前面所说的那些,也就是`print`所不能解决的问题. 那么唯一性就是如何设计这样一个模块,最合理最易用,当然存在性是相对的,目前的`logging`模块并不是不可替代的,事实上有很多第三方的`log`包可以提供更加便捷的操作,后面我们会提到.

现在回到问题的一开始,如何设计这样一个模块它要能完成如下任务:
- 它要包含一定运行时信息,包括时间,代码位置等等.
- 它要能保存到本地或者某台服务器上.
- 它最好速度要快,不影响运行
- 它最好分为一定等级,这样便于定位不同信息.
  
粗略能想到这些,那么`python`中的`logging`模块是怎样实现的呢?
在 https://docs.python.org/3/library/logging.html 中有以下几个主要的对象:
- **Logger** 即 Logger Main Class，是我们进行日志记录时创建的对象，我们可以调用它的方法传入日志模板和信息，来生成一条条日志记录，称作 Log Record。
- **Handler** 即用来处理日志记录的类，它可以将 Log Record 输出到我们指定的日志位置和存储形式等，如我们可以指定将日志通过 FTP 协议记录到远程的服务器上，Handler 就会帮我们完成这些事情。
- **Formatter**  实际上生成的 Log Record 也是一个个对象，那么我们想要把它们保存成一条条我们想要的日志文本的话，就需要有一个格式化的过程，那么这个过程就由 Formatter 来完成，返回的就是日志字符串，然后传回给 Handler 来处理。   
- **Filter** 另外保存日志的时候我们可能不需要全部保存，我们可能只需要保存我们想要的部分就可以了，所以保存前还需要进行一下过滤，留下我们想要的日志，如只保存某个级别的日志，或只保存包含某个关键字的日志等，那么这个过滤过程就交给 Filter 来完成。
- **LogRecord** 就代指生成的一条条日志记录。

- **LogAdapter**

整个使用流程见下面这个图:
![logging]({{site.url}}/assets/logging flow.png)
![](../assets/logging&#32;flow.png)

**未完待续**!