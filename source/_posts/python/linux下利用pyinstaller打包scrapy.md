---
title: linux下利用pyinstaller打包scrapy
date: 2019-08-31 00:01:44
tags: python, scrapy
password: 
---

要把爬虫部署到第三方云上面，自然是不能直接把代码扔上去的。若果编译成.pyc文件打包放上去，也不太保险。最后就调了python里面常用的打包工具pyinstaller，官方教程点[这里](https://pyinstaller.readthedocs.io/en/stable/)。

如何利用pyinstaller打包scrapy，其实网上有很多例子，譬如[这篇](https://blog.csdn.net/u010600274/article/details/99345367)，但是网上大多都是说如何打包成exe的，那么在linux下怎么打包呢？pyinstaller不是一个跨平台的打包工具，在windows下会打包成exe，在linux就会打包成linux下运行的elf，但是步骤都是一样的，所以可以参考上面推荐例子来打包。这里就不重复造轮子了，主要讲讲打包过程中踩过的坑。

首先，了解一下pyinstaller的打包方式，若果是打包成一个可执行文件，那么pyinstaller就会把python文件编译成.pyc文件并且嵌入到可执行文件中。pyinstaller打包时候会分析需要的模块，并且打包进去。

	To find out, PyInstaller finds all the import statements in your script. 
	It finds the imported modules and looks in them for import statements, 
	and so on recursively, until it has a complete list of modules your script may use.


譬如输入命令行`pyinstaller -F test.py`，pyinstaller首先会查找test.py里面的import语句，然后导入这些模块A，然后再从模块A中再查找import语句，导入其它依赖的模块B，如此类推，直到所有所有需要的模块的都找到。在打包的时候生成`build/`文件夹里有一个xref-xxx.html文件，列出了打包test.py所需要的所有模块。

另外，pyinstaller连python的解释器都拷贝进执行文件中，所以执行文件可以直接运行，不管计算机是否有python运行环境，但是这样的问题就是打包后的文件比较大。

从以上的描述可以知道，pyinstaller打包进去的文件都是静态加载的，对于动态加载的模块，也就是`hiddenimports`,pyinstaller是不会打包进去的，需要手动在打包配置文件`.spec`里面指定。

说完pyinstaller，再来看scrapy。按照官方的说法是，scrapy是低耦合的爬虫的框架，也就是说用户可以按照给定的API格式编写自己的组件，然后scrapy会在运行的时候动态加载，是的，动态加载。譬如爬虫组件，在scrapy源码里是这样加载进去的。

首先动态加载爬虫文件所在的包：

```py
def walk_modules(path):
    """Loads a module and all its submodules from the given module path and
    returns them. If *any* module throws an exception while importing, that
    exception is thrown back.

    For example: walk_modules('scrapy.utils')
    """

    mods = []
    mod = import_module(path)
    mods.append(mod)
    # 递归获取子模块,dfs
    if hasattr(mod, '__path__'):
        for _, subpath, ispkg in iter_modules(mod.__path__):
            fullpath = path + '.' + subpath
            if ispkg:
                mods += walk_modules(fullpath)
            else:
                submod = import_module(fullpath)
                mods.append(submod)
    return mods
```
`submod`里是爬虫类所在的modul，然后再提取这些爬虫类对象。

```py
def iter_spider_classes(module):
    """Return an iterator over all spider classes defined in the given module
    that can be instantiated (ie. which have name)
    """
    # this needs to be imported here until get rid of the spider manager
    # singleton in scrapy.spider.spiders
    from scrapy.spiders import Spider

    for obj in six.itervalues(vars(module)):
        if inspect.isclass(obj) and \
           issubclass(obj, Spider) and \
           obj.__module__ == module.__name__ and \
           getattr(obj, 'name', None):
            yield obj
```

所以爬虫的类是一个动态加载的过程。

其实不止爬虫模块，还有scarp, engine, downloadmiddleware等等scrapy的主要组件，都是动态加载的，也就是说pyinstaller都不会打包进可执行文件中。虽然打包过程没有问题，但是到了真正运行的时候就会报`ImportError: No module named '...'`的错误，其实都是这个原因导致的。

解决这个问题就是提示缺哪个模块，就在打包配置文件`.spec`的`hiddenimports`中加入这个模块。

除了这个`ImportError`的错误外，还会碰到`FileNotFound`的错误，这是因为和动态加载一样，pyinstaller也不会主动打包数据文件，同样需要在打包配置文件`.spec`的`datas`字段中，具体还是写法还是参考[例子](https://blog.csdn.net/u010600274/article/details/99345367)。

此外，还曾遇到过一种情况，就是在本地环境能够运行爬虫，但是打包后就提示`KeyError: 'Spider not found: xxx`，这里得检查两个地方。

1. 正如上文所述，pyinstaller不会主动打包动态加载的文件，所以需要检查爬虫文件是否已经打包进去。

2. `scrapy.cfg`这个scrapy的项目配置文件是否打包进去，或者是否和可执行文件在同一个目录下。没这个文件，scrapy会认为它不是在一个项目工程里面，所以找`settings.py`时候会有问题，导致不加载爬虫模块。

还有一个情况就是，执行文件可以成功运行，但是感觉就是卡住了一样不往下执行，也没有错误信息打印出来。这种情况下其实就是程序出错了，但是scrapy很迷的是异常抛出姿势有点问题，要知道错误原因，得修改一下源码。

修改一下scrapy这个第三方库里面crawler.py文件，位置是安装python的根目录下的`Lib/site-packages/scrapy/crawler.py`，找到以下代码，并且修改。

```python
 @defer.inlineCallbacks
    def crawl(self, *args, **kwargs):
        assert not self.crawling, "Crawling already taking place"
        self.crawling = True

        try:
            self.spider = self._create_spider(*args, **kwargs)
            self.engine = self._create_engine()
            # 这里调用了spider的start_requests，返回的是一个request
            start_requests = iter(self.spider.start_requests())
            yield self.engine.open_spider(self.spider, start_requests)
            yield defer.maybeDeferred(self.engine.start)
        except Exception as e:
            # In Python 2 reraising an exception after yield discards
            # the original traceback (see https://bugs.python.org/issue7563),
            # so sys.exc_info() workaround is used.
            # This workaround also works in Python 3, but it is not needed,
            # and it is slower, so in Python 3 we use native `raise`.


            # 打印错误信息
            print(e)
            if six.PY2:
                exc_info = sys.exc_info()

            self.crawling = False
            if self.engine is not None:
                yield self.engine.close()

            if six.PY2:
                six.reraise(*exc_info)
            raise
```

就是把这个`Exception`的错误信息给打印出来，就知道问题出在哪里了。

---
总结：

这个scrapy的打包问题纠结了我好几天，主要是因为scrapy的动态加载导致了很多问题。对scrapy的源码研读还不够深，`scrapy.cfg`这个文件没有打包的这个坑我就找了很久才忽然想起。网上很多的例子都是抄来抄去，很难找到说到问题上的，打铁还需自身硬啊。