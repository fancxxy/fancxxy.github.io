---
title: 漫画下载器
date: 2018-04-17 20:21:17
tags: Python
---

为了解决自己喜欢看漫画却没有好的观看体验的问题，用Python写了工具comicd，配合iOS上的iComics体验有了质的提升，不用再去忍受各种web和APP的弹窗推送。

comicd完全使用Python3标准库编写，未使用第三方模块，这篇文章是回顾我开发的过程，记录当时的一些思路，涉及到的知识点有基本Python语法、多线程、HTTP协议等。

comicd可以以命令的方式运行，也可以以包的形式导入作为应用代码的接口，目前支持的站点有腾讯漫画、网易漫画、动漫之家、动漫屋，想要自定义添加新的站点，只需要继承抽象接口类并实现所有必要的方法。
<!--more-->


{% asset_img comicd.gif %}

复杂的程序都是由简单组件组成的，首先介绍程序内的几个功能类。

## 请求
下载资源需要向资源站点发送HTTP请求，使用标准库的urllib包创建一个`Request`类对象，模拟发送请求。在`Request`类对象中创建默认的请求首部，包括内容协商、支持的压缩格式以及浏览器标示。`request`方法可以指定额外的请求首部，比如Host或者Referer，函数返回值是HTTP响应的主体部分。`urllib.parse.quote`方法对URL进行编码处理，除了指定的字符之外全部编码成加utf-8格式，用于网络传输。

``` python
import urllib.request
import urllib.parse
import urllib.error
from gzip import decompress

class Request(object):
    default = {
        'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
        'Accept-Encoding': 'gzip, deflate',
        'Accept-Language': 'zh-CN,en-US;q=0.5',
        'Connection': 'keep-alive',
        'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10.12; rv:54.0) Gecko/20100101 Firefox/54.0'
    }

    def request(self, url, data=None, headers=None):
        if headers:
            headers = {**self.default, **headers}
        else:
            headers = self.default
        url = urllib.parse.quote(url, safe='%/:?=&[]')
        req = urllib.request.Request(url, data=data, headers=headers)
        try:
            with urllib.request.urlopen(req) as res:
                if res.getheader('Content-Encoding') == 'gzip':
                    res = decompress(res.read())
                else:
                    res = res.read()
                return res
        except Exception as e:
            return None
```

`request`方法已经获取到了HTTP响应内容，根据获取到的内容不同处理方式也不相同。`content`方法对响应解码返回str。`page`方法用来保存网页到指定文件，如果没有指定文件名默认保存在当前目录下，正则表达式`r'(?<=<title>).*?(?=</title>)')`获取网页的标题，正向前视断言`(?=exp)`和正向后视断言`(?<=exp)`同时使用可以获取任意节点的内容。对于图像数据，直接保存原始的二进制数据，调用`binary`方法存储图像文件。

``` python
class Request(object):
    # ...
    def content(self, url, data=None, headers=None):
        response = self.request(url, data, headers)
        if response:
            return response.decode('utf-8', 'ignore')
        else:
            return ''

    def page(self, url, file=None, data=None, headers=None):
        response = self.request(url, data, headers).decode('utf-8', 'ignore')
        if not response:
            return False

        if not file:
            pattern = compile(r'(?<=<title>).*?(?=</title>)')
            result = pattern.search(response)
            if result:
                file = result.group(0)
            else:
                file = 'comic_' + strftime('%Y%m%d_%H%M%S', localtime(time()))
        with open(file + '.html', 'wb') as f:
            f.write(response.encode('utf-8'))

        return True

    def binary(self, url, data=None, headers=None):
        response = self.request(url, data=data, headers=headers)
        if response:
            return response
        else:
            return b''
```

## 日志
日志可以用来调试程序，定位问题，甚至可以分析用户习惯。Python标准库的`logging`模块提供记录日志的功能，利用`logging`模块封装一下自己的日志类，把它设置成单例模式，先创建一个单例类对象`Singleton`，让`Log`类继承获得单例属性。Python中创建单例模式的方式有多种，这里采用的方法是用`__new__`方法控制实例对象的创建。

``` python
class Singleton(object):
    def __new__(cls, *args, **kwargs):
        if not hasattr(cls, '_instance'):
            cls._instance = super(Singleton, cls).__new__(cls, *args, **kwargs)
        return cls._instance

class Log(Singleton):
    def __init__(self):
        if not hasattr(self, 'logger'):
            self.logger = logging.getLogger('comicd')

            handler = logging.StreamHandler(stream=stderr)

            fmt = '%(message)s'
            formatter = logging.Formatter(fmt, datefmt=None)

            handler.setFormatter(formatter)

            self.logger.setLevel(logging.INFO)
            handler.setLevel(logging.INFO)

            self.logger.addHandler(handler)

    def info(self, message):
        self.logger.info(message)

    def warning(self, message):
        self.logger.warning(message)

    def error(self, message):
        self.logger.error(message)
```

## 配置
配置文件通过`Config`类定义，提供可修改的配置有下载主目录，线程数量，失败请求的重试次数。`@property`装饰器可以把方法变成属性，添加参数判断。

``` python
class Config(Singleton):
    def __init__(self, home='~/Comics', comic=1, chapter=1, image=5, repeat=3, mode='crawler'):
        self._home = expanduser(home)

        self.log = join(self._home, '.comicd.log')

        self.serialize = join(self._home, '.comicd.dat')

        self._threads = [comic, chapter, image]

        self.repeat = repeat

        self.mode = mode

    @property
    def home(self):
        return self._home

    @home.setter
    def home(self, value):
        self._home = expanduser(value)

    @property
    def threads(self):
        return self._threads

    @threads.setter
    def threads(self, value):
        if not isinstance(value, list) or len(value) != 3:
            value = [1, 1, 5]
        self._threads = value

    def __repr__(self):
        return '<Config object home=\'{}\' threads={{comic:{}, chapter:{}, image:{}}} repeat={} mode={}>'.format(
            self._home, self.threads[0], self.threads[1], self.threads[2], self.repeat, self.mode)

    __str__ = __repr__


config = Config()
```

`config`实例对象作为暴露给外部的接口，为了让它更像是类对象，实现`__call__`方法。

``` python
class Config(Singleton):
    # ...
    def __call__(self, home='~/Comics', comic=1, chapter=1, image=5, repeat=3, mode='crawler'):
        self.__init__(home, comic, chapter, image, repeat, mode)
        return self
```

## 接口
有了以上三个功能类之后，我们就可以来写业务了。创建`interface`包用来实现漫画站点的解析功能，每个网站的解析都用一个类对象表示并放在单独的模块中。定义`Web`基类来规定接口，所有的网站解析类都要继承`Web`类对象，以保证`Comic`和`Chapter`模型(后面会介绍)调用解析接口不会报错。

``` python
class Web(object):
    _pattern = {}

    def __init__(self):
        self._request = Request()

    def url(self, url):
        try:
            return self._pattern['comic_url'].match(url).group()
        except AttributeError:
            return ''

    def curl(self, url):
        try:
            return self._pattern['chapter_url'].match(url).group()
        except AttributeError:
            return ''
```

为了方便扩展，解析模块以文件为单位动态加载。每个网站的解析类都放在同名的文件中，例如腾讯漫画的解析类名是`Tencent`，存放在`tencent.py`文件中。实现动态加载的过程是这样的，先遍历interface包获取所有模块的文件名，对应解析类的类名就是文件名首字母大写，根据Python的反射特性获取类名以及`host`属性，动态创建解析实例对象，存放在以`host`为key字典中。

``` python
class Interface(object):
    def __init__(self):
        self._interface = {}

        path = dirname(__file__)
        for file in listdir(path):
            name, extension = splitext(file)
            if name == '__init__' or extension != '.py':
                continue
            mod = import_module('comicd.interface.' + name)
            cls = getattr(mod, name.capitalize())
            host = getattr(cls, 'host')
            self._interface[host[0]] = cls()

    def __getitem__(self, key):
        return self._interface[key]

    @property
    def lists(self):
        return {cls.name for cls in self._interface.values()}

    @lists.setter
    def lists(self, value):
        raise AttributeError('lists is not a writable attribute')
```

至于解析类需要实现的方法要根据`Comic`和`Chapter`这两个模型来确定。

## 模型
`Comic`和`Chapter`模型分别表示漫画和章节，这两个类对象也是暴露给外部代码调用的接口，模型对象通过调用内部`interface`接口来解析并获取漫画的数据内容。`Comic`和`Chapter`都继承自基类`Model`，`Model`中实现两个基本的类方法。`find`方法根据正则表达式来获取输入的的url地址的netloc，在interface字典中查找对应的解析实例对象。`support`方法列出所有支持的网站列表。

``` python
from comicd.interface import interface
from comicd.config import config

class Model(object):
    def __init__(self):
        self.instance = None
        self.repeat = 1

    @classmethod
    def find(cls, url):
        try:
            pattern = compile(r'https?://([^/]+?)/')
            host = pattern.search(url).group(1)
            return interface[host]
        except (AttributeError, KeyError):
            raise ComicdError('comicd does not support this website')

    @classmethod
    def support(cls):
        return interface.lists
```

`Comic`对象包含漫画的所有内容，可以使用多线程来加快内容的获取。虽然CPython有GIL锁的限制，但是网络请求属于IO密集型的任务，所以使用多线程不受GIL锁影响。为了不让主线程在创建实例的时候等待响应而阻塞，初始化的时候只绑定不需要发送请求就能获取到的属性，而在之后需要访问其他属性时再发送请求并解析响应。

所以我利用描述器的特性创建一个`LazyProperty`装饰器来延迟初始化，原理就是用装饰器把类方法装饰成描述器，在第一次访问某个方法时，在漫画实例的`__dict__`查找，没有找到再到漫画类的`__dict__`中查找，找到之后调用方法并把返回的结果放到实例对象的`__dict__`中，之后再访问该描述器时就能直接从实例对象的`__dict__`获取结果。

``` python
class LazyProperty(object):
    def __init__(self, func):
        self.func = func

    def __get__(self, instance, type=None):
        val = self.func(instance)
        setattr(instance, self.func.__name__, val)
        return val
```

初始化时只绑定了两个实例属性，`self.instance`是解析漫画内容的实例对象，`self.url`是漫画的主页地址，在工作线程中可以调用`init`方法来强制获取所有属性的值。`data`属性存放HTTP的响应主体，`title`、`chapters`和`summary`分别是漫画标题，章节列表和简介，它们的获取方式都是通过把`data`属性传给`interface`接口，由`interface`接口解析返回结果。

``` python
from comicd.utils import LazyProperty as Lazy

class Comic(Model):
    def __init__(self, url):
        super().__init__()

        self.instance = Model.find(url)
        self.url = self._check(url)

    def init(self):
        if self.title == '' or self.summary == '' or self.chapters == []:
            return False
        else:
            return True

    @Lazy
    def data(self):
        return self.instance.comic(self.url)

    @Lazy
    def title(self):
        title = ''
        if self.data:
            title = self.instance.title(self.data)
        return title

    @Lazy
    def chapters(self):
        chapters = []
        if self.data:
            chapters = self.instance.chapters(self.data)
        return chapters

    @Lazy
    def summary(self):
        summary = ''
        if self.data:
            summary = self.instance.summary(self.data)
        return summary.strip(' ')
        
    def download(self):
        from comicd.crawler import Crawler
        crawler = Crawler()
        crawler.put('Comic', self)
        crawler.crawl()

    def _check(self, raw_url):
        url = self.instance.url(raw_url)
        if not url:
            raise ComicdError('comic url is incorrect')
        return url

    def __repr__(self):
        return '<Comic object interface=\'{}\' title=\'{}\'>'.format(self.instance.name, self.title)

    __str__ = __repr__
```

`Chapter`类的处理方式和`Comic`类似，用来获取章节的信息，两个类都提供了`download`方法来下载资源，`Chapter`类解析可以获得图片资源的真实地址，另外创建内部使用的模型`Image`来作为图片对象，`Image`类对象不对外公开，每个实例表示一张图片资源，`Chapter`实例对象解析出图片地址后批量初始化`Image`实例放到资源队列中，由其他工作线程下载对应图片。

``` python
class Image(Model):
    def __init__(self, url, instance, filename, referer, title=None, ctitle=None, page=None):
        super().__init__()
        self.url = url
        self.instance = instance
        self.filename = filename
        self.referer = referer

        self.title = title
        self.ctitle = ctitle
        self.page = page

    def download(self):
        complete = False
        binary = self.instance.image(self.url, self.referer)
        if binary:
            with open(self.filename, 'wb') as image:
                image.write(binary)
            complete = True
        return complete

    def __repr__(self):
        return '<Image object interface=\'{}\' title=\'{}\', ' \
               'ctitle=\'{}\', page=\'{}\'>'.format(self.instance.name, self.title, self.ctitle, self.page)

    __str__ = __repr__
```

## 线程
comicd本质就是一个爬虫，从输入漫画的主页地址开始，解析出漫画的所有章节地址，每获得一个章节地址就去解析该章节包含的所有图片的链接，通过链接下载资源。

为了完成上述功能，需要创建一个`Crawler`类，类三个队列分别用来存放`Comic`、`Chapter`以及`Image`实例对象，另外还需要初始化多个线程来处理上述队列中的实例对象。`initialize`方法接收一个URL列表，私有方法`_parse`根据URL的格式来返回字符串model，字符串的可选值是`Comic`和`Chapter`，`'{}(url)'.format(model)`生成字符串表达式，再通过`eval()`来构造实例对象，`self._queue[model].put()`把实例对象放到队列中。`crawl`方法创建并启动线程，`CrawlerThread`是一个线程类，重写`run`方法定义线程任务，`name`属性用来区分线程的工作类型。

``` python
class CrawlerThread(Thread):
    def __init__(self, handler, crawler, name):
        super().__init__(name=name)
        self.crawler = crawler
        self.handler = handler

    def run(self):
        while not interrupt:
            try:
                task = self.crawler.get(self.name)
                if self.handler(task):
                    self.crawler.put(self.name, task)
                else:
                    self.crawler.exception(self.name, task)
                self.crawler.finish(self.name)
            except Empty:
                if self.crawler.complete(self.name):
                    break

class Crawler(object):
    def __init__(self):
        self._queue = {'Comic': Queue(), 'Chapter': Queue(), 'Image': Queue()}

        self._lock = Lock()

        self._config = config

        self._log = Log()
        
        if config.mode == 'crawler':
            signal(SIGINT, Crawler.sig_handler)

    def initialize(self, urls):
        for u in urls:
            model, url = self._parse(u)
            if model:
                self._queue[model].put(eval('{}(url)'.format(model)))

        self.crawl()

    def crawl(self, comic_thread=None, chapter_thread=None, image_thread=None):

        comic_threads = [CrawlerThread(handler=Comic.init, crawler=self, name='Comic')
                         for _ in range(comic_thread if comic_thread else self._config.threads[0])]

        chapter_threads = [CrawlerThread(handler=Chapter.init, crawler=self, name='Chapter')
                           for _ in range(chapter_thread if chapter_thread else self._config.threads[1])]

        image_threads = [CrawlerThread(handler=Image.download, crawler=self, name='Image')
                         for _ in range(image_thread if image_thread else self._config.threads[2])]

        [t.start() for t in comic_threads]
        if config.mode == 'crawler':
            sleep(1)
        [t.start() for t in chapter_threads]
        if config.mode == 'crawler':
            sleep(1)
        [t.start() for t in image_threads]

        [t.join() for t in (*comic_threads, *chapter_threads, *image_threads)]
        
        if interrupt:
            self.serialize()
        
    def _parse(self, raw_url):
        try:
            instance = Model.find(raw_url)
            url = instance.curl(raw_url)
            if url:
                return 'Chapter', url
            url = instance.url(raw_url)
            if url:
                return 'Comic', url
        except ComicdError:
            self._log.warning('cannot handle url {}'.format(raw_url))
            return None, raw_url
            
    @staticmethod
    def sig_handler(signum, frame):
        global interrupt
        interrupt = True
```

`Crawler`实例在初始化时设定了捕捉`SIGINT`信号，如果检测到Ctrl+C信号，则调用`sig_handler`方法把`interrupt`设置成True，在`crawl`方法中如果发现线程不是自动结束而是被`interrupt`标志给终止时，调用`serialize`方法把三个队列中未处理的实例序列化到本地，对应的实现了一个反序列化的方法`deserialize`可以快速恢复到上一次终止的状态，这样就实现了断点续传的功能了。

``` python
class Crawler(object):
    # ...
    def serialize(self):
        with open(self._config.serialize, 'wb') as f:
            dump((self._queue['Comic'].queue, self._queue['Chapter'].queue, self._queue['Image'].queue), f)

    def deserialize(self):
        if not exists(self._config.serialize):
            self._log.error('no serialize file found')
            return

        with open(self._config.serialize, 'rb') as f:
            q1, q2, q3 = load(f)
            [self._queue['Comic'].put(item) for item in list(q1)]
            [self._queue['Chapter'].put(item) for item in list(q2)]
            [self._queue['Image'].put(item) for item in list(q3)]

        self.crawl()
```

线程类的`run`方法定义了线程功能，主要逻辑是从队列获取元素，处理完成之后放到下一级队列中，如果队列为空需要判断自身是否满足退出条件，满足则退出线程否则进行下一轮循环。不同功能的线程处理方式略有不同，通过创建线程时指定的线程名作为判断线程功能的标记。每个实例对象都有一个`repeat`属性用来标记实例被处理的次数，如果实例处理失败了，在未超过预设定的重复次数之前会被重新放到队列中等待下次处理，一定程度上排除网络原因带来的干扰。

``` python
class Crawler(object):
    # ...
    def put(self, name, task):
        if name == 'Comic':
            for chapter in task.chapters:
                try:
                    self._queue['Chapter'].put(Chapter(chapter[1]))
                except ComicdError:
                    self._log.warning('cannot handle url {}'.format(chapter[1]))
        elif name == 'Chapter':
            f = Folder(self._config.home, task.title, task.ctitle)
            self._queue['Image'].put(f)
            for image in task.images:
                try:
                    self._queue['Image'].put(Image(image[1], task.instance,
                                                   join(f.path, image[0]),
                                                   task.url, task.title,
                                                   task.ctitle, image[0]))
                except ComicdError:
                    self._log.warning('cannot handle url {}'.format(image[1]))

    def get(self, name):
        if name == 'Comic' or name == 'Chapter':
            task = self._queue[name].get(timeout=1)
        else:  # name == 'Image'
            with self._lock:
                task = self._queue[name].get(timeout=1)
                while isinstance(task, Folder):
                    task.create()
                    self._log.info(' <' + task.title + '> ' + task.ctitle)
                    task = self._queue[name].get(timeout=1)
        return task

    def finish(self, name):
        self._queue[name].task_done()

    def complete(self, name):
        if name == 'Comic':
            return True
        elif name == 'Chapter':
            return self._queue['Comic'].empty()
        elif name == 'Image':
            return self._queue['Comic'].empty() and self._queue['Chapter'].empty()

    def exception(self, name, task):
        if task.repeat >= self._config.repeat:
            self._log.warning('handle task {} failed'.format(task))
        else:
            task.repeat += 1
            self._queue[name].put(task)
```

## 脚本
最后一步用上述功能类来组装一个脚本程序完成漫画下载的功能。标准库的`argparse`包可以很方便的给脚本添加参数，我这里设置了三个参数，`—url`参数是一个URL的列表，包含了所有需要下载的某个漫画或某个章节的地址列表，`—resume`参数恢复上一次终止的任务，`—file`参数指定文件名，从文件中读取地址列表。

``` python
def comicd():
    if not exists(config.home):
        makedirs(config.home)

    parser = ArgumentParser(description='Comic Download Tool')
    parser.add_argument('-u', '--url', nargs='+', dest='urls', help='urls of comics')
    parser.add_argument('-r', '--resume', action='store_true', default=False, dest='resume',
                        help='resume last crawl task')
    parser.add_argument('-f', '--file', action='store', dest='file', help='crawl from file')
    args = parser.parse_args()

    urls = []

    from comicd.crawler import Crawler
    crawler = Crawler()

    if args.resume:
        crawler.deserialize()
        exit(0)

    if args.urls:
        urls = args.urls
    elif args.file:
        with open(args.file, 'r') as f:
            urls = [u for u in f.readlines()]

    crawler.initialize(urls)


if __name__ == '__main__':
    comicd()
```

## 打包
comicd提供了`Comic`, `Chapter`, `Config`和`ComicdError`作为外部使用的编程接口，全部写入到__init__.py的`__all__`变量。

``` python
from comicd.model import Comic, Chapter
from comicd.config import config as Config
from comicd.error import ComicdError

__all__ = [
    'Comic',
    'Chapter',
    'Config',
    'ComicdError'
]
```

setup.py填写基本的配置信息，指定脚本名为`comic`，配置好依赖，当然在这个项目里requirements.txt内容是空的。

``` python
from setuptools import setup, find_packages

requirements = open('requirements.txt').readlines()

setup(
    name='comicd',
    version='0.0.1',
    author='fancxxy',
    author_email='fancxxy@gmail.com',
    url='https://github.com/fancxxy/comicd',
    description='comic download tool',
    scripts=['comic'],
    license='MIT',
    classifiers=[],
    packages=find_packages(exclude=('tests',)),
    install_requires=requirements
)
```

