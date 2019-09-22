---
title: scrapy 日志输出踩过的坑
date: 2018-10-24 00:01:44
tags: python, scrapy
password: 
---

最近在写爬虫，记录一下在日志输出的时候踩过的坑

scrapy使用的日志输出库是python的标准输出模块logging，使用方法网上一[大堆](https://www.cnblogs.com/yyds/p/6901864.html)。配置方式参考[这里](https://www.cnblogs.com/yyds/p/6885182.html)

在开发中一开始是用[logging.config.fileConfig](https://docs.python.org/3.7/library/logging.config.html#logging.config.fileConfig),但是这个配置方式不能使用filter组件，只好换了一个功能更强大的[logging.config.dictConfig](https://docs.python.org/3.7/library/logging.config.html#logging.config.dictConfig)。

类似的代码如[这里](https://blog.csdn.net/caoxinjian423/article/details/84196609)。

运行代码后，发现的问题就是scrapy本身自带的日志输出功能全部都没有了，无论怎么样都不输出。对代码进行debug，发现scrapy的logger都被disabled了。继续调试，发现在自定义的输出日志语句之前，scrapy的logger都是disabled=False的，但是执行完自定义的logging输出日志语句后，scrapy的logger就变成了disabled=True了。这里是真的百思不得其解。经过大量的面向谷歌后，终于发现在[官方文档](https://docs.python.org/3.7/library/logging.config.html#logging.config.fileConfig)里有那么一句话。

	Note If you want to send configurations to the listener which don’t disable existing loggers,
	you will need to use a JSON format for the configuration, which will use dictConfig() for configuration.
	This method allows you to specify disable_existing_loggers as False in the configuration you send.


disable_existing_loggers默认为true，这个会把scrapy的logger给disabled了

在json的配置文件里，加上那么一句`disable_existing_loggers:false`。scrapy的log就都输出来了，问题解决。

---

还有一个关于在多进程下利用ConcurrentLogHandler输出日志的问题，在linux下正常工作，但是在windows下，写文件的时候会被锁住挂起。解决方法是用[concurrent-log-handler](https://github.com/Preston-Landers/concurrent-log-handler)进行替代，参考[这里](https://blog.csdn.net/wkb342814892/article/details/80281182)。

---
总结:因为都是边做边学，所以都没有好好研究过，特别是官方文档，浪费的时间有点多。还是那句老话，工欲善其事必先利其器，还是得先静下心来学习。