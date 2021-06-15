+++
title = "Python的logging模块总结"
date = "2015-02-02"
slug = "2015/02/02/logging-module-of-python"
Categories = ["Python"]
+++

里面主要有这么几个类：logger handler formatter，其中logger类用于实例化logger对象，handler类负责指定log信息的发送目的地，比如你可能希望log信息发送到终端、文件甚至远程主机上，还支持日志轮转，比如`RotatingFileHandler`支持按文件轮转，`TimedRotatingFileHandler`支持按时间轮转，formatter类，顾名思义，负责log信息的格式的设定。

使用logging模块的基本流程为：先通过`logging.getlogger('loggername')`获取一个logger对象，然后根据需求为这个logger对象添加相应的handler和formatter，这样就算基本配置完了，这样就可以在你的程序里使用`logging.debug('debug info')`等类似语句来输出log信息了。

在实际开发中一般是通过文件来进行配置的，配置信息全写在一个配置文件中，然后在脚本里读取配置信息，一个简单的示例：
    import logging
    import logging.config

    logging.config.fileConfig('logging.conf')

    # create logger
    logger = logging.getLogger('simpleLogger')

    # use it
    logger.debug('debug info')
    logger.info('info message')
    logger.warn('warning info')
    logger.critical(''critical info')
配置文件如下，需要遵守一定的格式，具体要求见参考链接
    [loggers]
    keys=root, simpleLogger

    [handlers]
    keys=consoleHandler

    [formatters]
    keys=simpleFormatter

    [logger_root]
    level=DEBUG
    handlers=consoleHandler
    ...
    ...
    [formatter_simpleFormatter]
    format=%(asctime)s - %(name)s - %(levelname)s - %(message)s
    datefmt=

## 参考
- [Logging HOWTO](https://docs.python.org/2/howto/logging.html)
- [Logging configuration](https://docs.python.org/2/library/logging.config.html#logging-config-api)  这里详细讲了配置文件到底该怎么写
- [使用python的logging模块](http://kenby.iteye.com/blog/1162698)
- [Python模块学习——logging](http://www.cnblogs.com/captain_jack/archive/2011/01/21/1941453.html)
- [Python 学习入门（14）—— logging](http://blog.csdn.net/ithomer/article/details/16985379)
