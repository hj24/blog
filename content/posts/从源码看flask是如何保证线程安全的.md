---
title: "从源码看flask是如何保证线程安全的"
date: 2020-03-02T15:37:20+08:00
draft: false
tags: ["Python"]
---
<meta name="referrer" content="no-referrer" />
之前粗略的看过flask的文档，知道了它是线程安全的，大概是通过维护了一个以线程ID (在 Greenlet 可用的情况下优先使用 Greenlet 的 ID，因此协程也是一样的)为键的字典，里面放了分配给这个线程的资源来做到线程隔离，以此实现线程安全的目的。

写这篇博客是因为之前有个同事问我他写了个全局变量，在flask里是线程安全的吗？
当时没过脑子，直接回答了是，现在想想...应该是把他坑了

为了彻底弄明白这个问题，我搜了很多知乎，stackoverflow，reddit的问答，还有一些博客，也看了一遍相关的源码，最后写下这篇博客，做个记录。

<!--more-->
## AppContext和RequestContext是什么
先说几句概念性的东西，flask的线程安全是基于`Local`、`LocalStack`和`LocalProxy`来做的，这三个对象体现在源码中的`AppContext` 和`RequestContext`中（[源码在这里](https://github.com/pallets/flask/blob/master/src/flask/ctx.py))

一开始我对flask线程安全的理解是只局限于知道它通过维护了一个以线程ID为键的字典，里面放了分配给这个线程的资源来做到线程隔离这样，AppContext和RequestContext是啥我是完全不知道的。
看了一下源码（在ctx.py内），发现是flask app和request请求的上下文管理器: 

> App Context 是代表应用上下文，可能包含各种配置信息，比如日志配置，数据库配置等
> Request Context 代表一个请求上下文，我们可以获取到当前请求中的各种信息。比如 body 携带的信息

来看一下`AppContext`的上下文管理器协议里写了些什么:
```python
def __enter__(self):
    self.push()
    return self

def __exit__(self, exc_type, exc_value, tb):
    self.pop(exc_value)

    if BROKEN_PYPY_CTXMGR_EXIT and exc_type is not None:
        reraise(exc_type, exc_value, tb)
```

在进入上下文时，会有一个栈的push操作，里面是这样定义的:
```python
def push(self):
    self._refcnt += 1
    if hasattr(sys, "exc_clear"):
        sys.exc_clear()
    _app_ctx_stack.push(self)
    appcontext_pushed.send(self.app)
```
最终它会被压到`_app_ctx_stack`这个栈中，同样的，对于Request上下文，它会被压到`_request_ctx_stack`这个栈中，不过它还多一步：
```python
def push(self):
    top = _request_ctx_stack.top
    if top is not None and top.preserved:
        top.pop(top._preserved_exc)

    # Before we push the request context we have to ensure that there
    # is an application context.
    app_ctx = _app_ctx_stack.top
    if app_ctx is None or app_ctx.app != self.app:
        app_ctx = self.app.app_context()
        app_ctx.push()
        self._implicit_app_ctx_stack.append(app_ctx)
    else:
        self._implicit_app_ctx_stack.append(None)

    if hasattr(sys, "exc_clear"):
        sys.exc_clear()

    _request_ctx_stack.push(self)

    ...
```
就是注释没被删掉的那部分，在进入request的上下文之前，需要确保已经有app上下文了，因为Flask实例化创建app时并没有进入AppContext
对于这一点，我的理解是，如果当前app没有入栈的话，存在多个app实例时，通过current_app是获取不到当前app上下文的信息的

那这两栈究竟是啥呢？
我去看了看他们被定义的文件（globals.py文件）:
```python
...

from werkzeug.local import LocalProxy
from werkzeug.local import LocalStack

...

# context locals
_request_ctx_stack = LocalStack()
_app_ctx_stack = LocalStack()
current_app = LocalProxy(_find_app)
request = LocalProxy(partial(_lookup_req_object, "request"))
session = LocalProxy(partial(_lookup_req_object, "session"))
g = LocalProxy(partial(_lookup_app_object, "g"))
```
发现使用的还是werkzeug的Local相关的那三个类，`_app_ctx_stack`和`_request_ctx_stack`其实都是`LocalStack`这个栈

到这里为止，我还是很懵逼，不懂这两个上下文管理器有什么用，为什么要用到栈？

于是我想去看看它们是在什么地方被调用的，我是先从request下手的，既然是请求，肯定是从wsgi过来的，于是去看了wsgi_app的源码 (在flask的app.py里):
```python
def wsgi_app(self, environ, start_response):
    ctx = self.request_context(environ)
    error = None
    try:
        try:
            ctx.push()
            response = self.full_dispatch_request()
        except Exception as e:
            error = e
            response = self.handle_exception(e)
        except:  # noqa: B001
            error = sys.exc_info()[1]
            raise
        return response(environ, start_response)
    finally:
        if self.should_ignore_error(error):
            error = None
        ctx.auto_pop(error)
```
可以看出来，请求到达，先进入request上下文，这时候会去检查是否有app上下文，没有就会先构建它。
到这里，我只是大概理清楚了一个flask请求周期内这两个上下文的运作流程，我找了一张流程图，可以看一下:
![](https://tva1.sinaimg.cn/large/00831rSTgy1gcfz969o4tj30k004b3yi.jpg)

到这里，我又多了个疑问，既然request上下文构建时要顺便构建app上下文，为什么要区分它们两个？

这几个问题其实我看源码琢磨的不是很透彻，不过我在搜知乎时倒是看到一个答案，有种豁然开朗的感觉:
1. 关于为什么用栈结构
其实这个问题转换一下就是，什么情况下多个flask app会共存在同一个线程内？不然单一app也没必要用栈啊
这里引用大佬的回答:

> 一个 Flask App 实例就是一个 WSGI Application，那么 WSGI Middleware 是允许使用组合模式的
> Werkzeug 内置的 Middleware 将两个 Flask App 组合成一个一个 WSGI Application。这种情况下两个 App 都同时在运行，只是根据 URL 的不同而将请求分发到不同的 App 上处理

其实一开始我理解错了，以为是gunicorn那种多个worker进程拉起多个flask app的形式，后来想想这都多线程了，肯定是隔离的啊，于是我去看了werkzeug的源码，在`werkzeug/middleware/dispatcher.py`里找到了答案：
```
This middleware creates a single WSGI application that dispatches to
multiple other WSGI applications mounted at different URL paths.
```
这段注释里写的很清楚，这个中间件是用来创建一个单独的wsgi应用，来分发不同的请求到多个其它的wsgi应用上去的。
这里我有疑问，因为从来没这么用过，这种需求不都是独立部署多个服务，走微服务那一套，或者直接用REST去请求不同应用的吗，结果继续往下看，注释里又给我解释了
```
In production, you might instead handle this at the HTTP server level,
serving files or proxying to application servers based on location. The
API and admin apps would each be deployed with a separate WSGI server,
and the static files would be served directly by the HTTP server.
```
很明显，作者也建议在生产环境上用我上面说的那些解决方案，这应该只是在开发中可能会使用的
用法也很简单，通过这样的形式在单一线程中开启多个app:
```python
app = DispatcherMiddleware(serve_frontend, {
        '/api': api_app,
        '/admin': admin_app})
```

2. 为什么要区分App和Request上下文？
继续引用大佬的回答:

> 这个做法给予在非 Web Runtime的概念 中灵活控制 Context 的可能性
> Flask 的两种 Context 分离更大的意义是为了非 web 应用的场合

说来惭愧，看完之后才意识到flask也有非web场景下的使用情况，想象一下，在非web runtime的情况下，如果这没有单独的app context，当前有多个app实例的时候怎么做到各个app的资源都是独立的呢？

来看这个例子：
```python
from data.user_model import User, db

database = Flask(__name__)
database.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:////tmp/test.db'

db.init_app(database)
db.create_all()
admin = User(username='admin', email='admin@example.com')
db.session.add(admin)
db.session.commit()
print(User.query.filter_by(username="admin").first())

database1 = Flask(__name__)
database1.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:////tmp/test1.db'


db.init_app(database1)
db.create_all()
admin = User(username='admin_test', email='admin@example.com')
db.session.add(admin)
db.session.commit()
print(User.query.filter_by(username="admin").first())
```
显然这是个没有使用app context的场景，我们的本意是给两个共存的flask app实例各自绑定一个数据库
但是它并没有做到资源隔离，db在解释执行到database1之前代表的是database的数据库，之后又和datbase1绑定在一起，没有context来隔离，怎么在不同的场景下使用不同的db？

那么，修改一下：
```python
from data.user_model import User, db

database = Flask(__name__)
database.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:////tmp/test.db'

with database.app_context():
    db.init_app(current_app)
    db.create_all()
    admin = User(username='admin', email='admin@example.com')
    db.session.add(admin)
    db.session.commit()
    print(User.query.filter_by(username="admin").first())

database1 = Flask(__name__)
database1.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:////tmp/test1.db'

with database1.app_context():
    db.init_app(current_app)
    db.create_all()
    admin = User(username='admin_test', email='admin@example.com')
    db.session.add(admin)
    db.session.commit()
    print(User.query.filter_by(username="admin").first())
```
有了app上下文就好办了，可以通过`current_app`来获取当前的app，来初始化不同的db，上面这两个db都处于不同的上下文之中，因此都是隔离的

到这里，问题就迎刃而解了，为了防止没说清楚，再给个例子:

```python
from flask import Flask, current_app
import logging

app = Flask("app1")
app2 = Flask("app2")

app.config.logger = logging.getLogger("app1.logger")
app2.config.logger = logging.getLogger("app2.logger")

app.logger.addHandler(logging.FileHandler("app_log.txt"))
app2.logger.addHandler(logging.FileHandler("app2_log.txt"))

# 没有request上下文的时候，我要使用app的logger记录一下日志，这时候就需要app上下文了
with app.app_context():
    with app2.app_context():
        try:
        	# 这里故意抛出一个异常，实际情况下可能是真实系统中的异常
            raise ValueError("app2 error")
        except Exception as e:
            current_app.config.logger.exception(e)
    try:
        raise ValueError("app1 error")
    except Exception as e:
        current_app.config.logger.exception(e)
```

这个例子也能说明为什么要用栈结构，两个app叠加使用时，不用栈，取出来的就不是对于的app了，比如这里的例子，如果使用先进先出的队列，app2压栈后取出来的是app1的上下文，那不全乱套了。

有人可能有疑问，这两app分别调用了自己的`app_context()`方法，那不是只会压到他们自己的栈里吗，不用栈不也一样的。
好问题，我之前疑问过，不过回去一看就明白了，app栈和request栈是这么定义的:
```python
# context locals
_request_ctx_stack = LocalStack()
_app_ctx_stack = LocalStack()
```
是已经实例化之后的对象，在别的模块被导入时其实是利用python的模块机制做了个单例模式，所以两个app都会被压到同一个栈中。不得不感叹flask设计的精巧啊

到这里，我算是大概理解了AppContext和RequestContext为什么要独立存在的原因了，并且我也大概了解他们的作用：通过底层的`LocalStack()`来做到当请求过来时根据线程号来取出当前关联的request和app，使用这些资源完成业务逻辑

关于后者，我还想再进一步了解他们是怎么实现的通过当前线程ID取到对应的资源，也就是具体是如何线程隔离的，于是我哼哧哼哧的去看了`Local`、`LocalStack`和`LocalProxy`的源码（在werkzeug的local.py文件内）

## Local和LocalStack
先来看Local，我在看它的源码之前是抱着这几个疑问的：
1. 它是怎么实现才能做到线程隔离的
2. 它和LocalStack有什么关系

Local的源码不长:
```python
try:
    from greenlet import getcurrent as get_ident
except ImportError:
    try:
        from thread import get_ident
    except ImportError:
        from _thread import get_ident

class Local(object):
    __slots__ = ("__storage__", "__ident_func__")

    def __init__(self):
        object.__setattr__(self, "__storage__", {})
        object.__setattr__(self, "__ident_func__", get_ident)

    def __iter__(self):
        return iter(self.__storage__.items())

    def __call__(self, proxy):
        """Create a proxy for a name."""
        return LocalProxy(self, proxy)

    def __release_local__(self):
        self.__storage__.pop(self.__ident_func__(), None)

    def __getattr__(self, name):
        try:
            return self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)

    def __setattr__(self, name, value):
        ident = self.__ident_func__()
        storage = self.__storage__
        try:
            storage[ident][name] = value
        except KeyError:
            storage[ident] = {name: value}

    def __delattr__(self, name):
        try:
            del self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)
```

开始导入了一个`get_ident`方法，主要就是用来获取当前线程ID，greenlet可用时优先导入它的`get_ident`方法，初始化local时创建了一个名为`__storage__`的字典，这就是用来根据线程ID存储它的资源的字典，从这段代码就可以看出来:
```python
def __getattr__(self, name):
    try:
        return self.__storage__[self.__ident_func__()][name]
    except KeyError:
        raise AttributeError(name)
```
1. 通过重写getattr来在`__storage__`字典里根据`self.__ident_func__()`
2. 获得的线程ID作为键值来获取当前线程对象内的name资源，这里的"__ident_func__"就是`get_ident`
3. 要是释放资源时，调用`__release_local__`来根据线程ID把它从字典里pop掉

## LocalStack
来看LocalStack的源码
```python
class LocalStack(object):
    
    def __init__(self):
        self._local = Local()

    def __release_local__(self):
        self._local.__release_local__()

    @property
    def __ident_func__(self):
        return self._local.__ident_func__

    @__ident_func__.setter
    def __ident_func__(self, value):
        object.__setattr__(self._local, "__ident_func__", value)

    def __call__(self):
        def _lookup():
            rv = self.top
            if rv is None:
                raise RuntimeError("object unbound")
            return rv

        return LocalProxy(_lookup)

    def push(self, obj):
        """Pushes a new item to the stack"""
        rv = getattr(self._local, "stack", None)
        if rv is None:
            self._local.stack = rv = []
        rv.append(obj)
        return rv

    def pop(self):
        """Removes the topmost item from the stack, will return the
        old value or `None` if the stack was already empty.
        """
        stack = getattr(self._local, "stack", None)
        if stack is None:
            return None
        elif len(stack) == 1:
            release_local(self._local)
            return stack[-1]
        else:
            return stack.pop()

    @property
    def top(self):
        """The topmost item on the stack.  If the stack is empty,
        `None` is returned.
        """
        try:
            return self._local.stack[-1]
        except (AttributeError, IndexError):
            return None
```
其实是对Local的封装，初始化的时候就是一个Local对象，给它加上了栈的结构：
```python
def push(self, obj):
    rv = getattr(self._local, "stack", None)
    if rv is None:
        self._local.stack = rv = []
    rv.append(obj)
    return rv

def pop(self):
    stack = getattr(self._local, "stack", None)
    if stack is None:
        return None
    elif len(stack) == 1:
        release_local(self._local)
        return stack[-1]
    else:
        return stack.pop()
```
具体就是在执行push、pop操作的时候先去看`self._local`有没有stack属性，有点话执行push pop，没有的话给它绑定上一个栈
根据栈后进先出的特点，获取的栈顶元素一定是目前最新的正在使用的

## LocalProxy

之前提到了Local和LocalStack，知道了它们是怎么实现的了，其实实现线程隔离，我的感觉是上面这两个就够了，为什么还要LocalProxy呢？

不过在这之前，我有个大概的猜想，从名字可以看出来这是一个代理，具体可以去看《设计模式》里的代理模式，它的作用应该就是给Local和LocalStack做代理，在它们的基础上可以做一些功能的封装之类的操作，让写法更简洁吧

为了解决这个疑问，来看LocalProxy的源码：
```python
@implements_bool
class LocalProxy(object):
    
    __slots__ = ("__local", "__dict__", "__name__", "__wrapped__")

    def __init__(self, local, name=None):
        object.__setattr__(self, "_LocalProxy__local", local)
        object.__setattr__(self, "__name__", name)
        if callable(local) and not hasattr(local, "__release_local__"):
            object.__setattr__(self, "__wrapped__", local)

    def _get_current_object(self):
        if not hasattr(self.__local, "__release_local__"):
            return self.__local()
        try:
            return getattr(self.__local, self.__name__)
        except AttributeError:
            raise RuntimeError("no object bound to %s" % self.__name__)

    @property
    def __dict__(self):
        try:
            return self._get_current_object().__dict__
        except RuntimeError:
            raise AttributeError("__dict__")

    def __repr__(self):
        try:
            obj = self._get_current_object()
        except RuntimeError:
            return "<%s unbound>" % self.__class__.__name__
        return repr(obj)

    def __bool__(self):
        try:
            return bool(self._get_current_object())
        except RuntimeError:
            return False

    def __unicode__(self):
        try:
            return unicode(self._get_current_object())  # noqa
        except RuntimeError:
            return repr(self)

    def __dir__(self):
        try:
            return dir(self._get_current_object())
        except RuntimeError:
            return []

    def __getattr__(self, name):
        if name == "__members__":
            return dir(self._get_current_object())
        return getattr(self._get_current_object(), name)

    def __setitem__(self, key, value):
        self._get_current_object()[key] = value

    def __delitem__(self, key):
        del self._get_current_object()[key]

    if PY2:
        __getslice__ = lambda x, i, j: x._get_current_object()[i:j]

        def __setslice__(self, i, j, seq):
            self._get_current_object()[i:j] = seq

        def __delslice__(self, i, j):
            del self._get_current_object()[i:j]

```

它的作用就是代理Local和LocalStack对象，举个例子，来看之前的源码:
```python
def _lookup_req_object(name):
    top = _request_ctx_stack.top
    if top is None:
        raise RuntimeError(_request_ctx_err_msg)
    return getattr(top, name)

_request_ctx_stack = LocalStack()

request = LocalProxy(partial(_lookup_req_object, "request"))
```

这是flask的request对象定义的地方
`_lookup_req_object`是为了从request栈里取出当前线程的request，把这个方法的name参数固定住存到LocalProxy里面的local属性里：
```python
object.__setattr__(self, "_LocalProxy__local", local)
```
然后就可以使用`self._loacl()`来获取当前代理的对象了，在代理里面是通过`_get_current_object`来获取的:
```python
def _get_current_object(self):
    if not hasattr(self.__local, "__release_local__"):
        return self.__local()
    try:
        return getattr(self.__local, self.__name__)
    except AttributeError:
        raise RuntimeError("no object bound to %s" % self.__name__)
```
然后，每一次获取当前线程的request对象，都会使用`self._local()`调用`_lookup_req_object`去request栈里获取当前request上下文出来

如果没有这个代理，来看这个例子 (偷懒直接拿了大佬回答里的例子，反正自己写出来的也差不多):

```python
from werkzeug.local import LocalStack
test_stack = LocalStack()
test_stack.push({'abc': '123'})
test_stack.push({'abc': '1234'})

def get_item():
    return test_stack.pop()

item = get_item()

print(item['abc'])
print(item['abc'])
```
打印结果是：
```
123
123
```

每次的输出都是一样的，都是上次栈顶弹出来的值，想要新的栈顶元素，每次都要重新执行get_item
有了代理之后，把这个例子修改一下

```python
from werkzeug.local import LocalStack, LocalProxy
test_stack = LocalStack()
test_stack.push({'abc': '123'})
test_stack.push({'abc': '1234'})

def get_item():
    return test_stack.pop()

item = LocalProxy(get_item)

print(item['abc'])
print(item['abc'])
```
打印结果是：
```
123
1234
```

通过把get_item方法传进去，使用item是每次都会把去栈里去最新数据的操作交给代理去完成，这就是Proxy的用处，试试也验证了我之前的猜想

关于源码的部分就是这么多了

## 总结

回过头来看文章开头的问题，全局变量在flask里真的是线程安全的嘛？
- 绝对不是

即使你不相信我，也请你相信davidism，他是flask的核心开发者，并且在stackoverflow的一个相关[回答](https://stackoverflow.com/questions/32815451/are-global-variables-thread-safe-in-flask-how-do-i-share-data-between-requests)里明确表达了：

> You can't use global variables to hold this sort of data. Not only is it not thread 
> safe, it's not process safe, and WSGI servers in production spawn multiple 
> processes. Not only would your counts be wrong if you were using threads to handle 
> requests, they would also vary depending on which process handled the request.
> Use a data source outside of Flask to hold global data. A database, memcached, or 
> redis are all appropriate separate storage areas, depending on your needs. If you 
> need to load and access Python data, consider multiprocessing.Manager. You could 
> also use the session for simple data that is per-user.

flask的线程安全，是基于Local，LocalStack的线程隔离机制来做的，他们通过线程ID来隔离每个线程的独自的资源，既然每个线程都是隔离的，那肯定也是线程安全的了
如果你想用全局变量，最好加锁来操作吧，或者把它写进数据库里，redis里，具体怎么做需要你看自己的需求决定了。

## 参考
1. [知乎 - flask中的context初探](https://zhuanlan.zhihu.com/p/33847569)
2. [stackoverflow - Are global variables thread safe in flask? How do I share data between requests?](https://stackoverflow.com/questions/32815451/are-global-variables-thread-safe-in-flask-how-do-i-share-data-between-requests)
3. [stackoverflow - How does Flask keep the request global threadsafe](https://stackoverflow.com/questions/23726788/how-does-flask-keep-the-request-global-threadsafe)
4. [stackoverflow - Flask global variables](https://stackoverflow.com/questions/23457658/flask-global-variables)
5. [reddit (需科学上网) - When people say flask isnt thread safe what does that mean?](https://www.reddit.com/r/flask/comments/90gjsj/when_people_say_flask_isnt_thread_safe_what_does/)
6. [reddit - Are global variables thread safe in flask?](https://www.reddit.com/r/learnpython/comments/3mnw8w/are_global_variables_thread_safe_in_flask/)
7. [Design Decisions in Flask 中的 Thread Locals 一节](https://flask.palletsprojects.com/en/1.1.x/design/)
8. [flask globals源码](https://github.com/pallets/flask/blob/master/src/flask/globals.py)
9. [flask ctx源码](https://github.com/pallets/flask/blob/master/src/flask/ctx.py)
10. [知乎 - Werkzeug(Flask)之Local、LocalStack和LocalProxy](https://zhuanlan.zhihu.com/p/33732859)
11. [werkzeug local源码](https://github.com/pallets/werkzeug/blob/master/src/werkzeug/local.py)

