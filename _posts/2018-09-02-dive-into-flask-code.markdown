---
layout: post
title: 'Flask 0.1 源码解析'
subtitle:   "Dive into flask 0.1"
date:       2018-09-02 22:52:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
    - flask
---

## 理解 Flask 0.1
为什么会挑选flask 0.1作为源码阅读的目标，主要原因在于简洁，可以比较清晰看到完整逻辑。还有一部分原因在于：整体的框架与思路与最新的代码都是一致的。因此，此次就以flask 0.1 源码作为介绍的目标，看看flask是如何实现优雅实现一个服务器的，具体注释的源码可以查看[Flask 0.1](https://github.com/hustyichi/flask_0.1)

理解flask的源码需要一些基础，首先是需要python的基础，flask中无处不在的magic method，需要对此有一些概念；其次是wsgi(Web Server Gateway Interface)，flask0.1 只有几百行代码，不可能实现一个完整的服务器，flask只是按照特定的wsgi协议，封装了服务器端核心的部分，让用户不需要关心服务器端具体协议，只需要关心应用代码即可。关于Wsgi的入门代码可以参考[cizixs的博客](http://cizixs.com/2014/11/08/understand-wsgi)；最后是需要实际使用flask，因为只有实际用过，才会理解flask的具体的操作，要不然就很难建立相关的基础概念。

## 完整流程

#### 启动应用
使用过Flask的小伙伴们应该知道，启动Flask时的代码如下所示：

```
app = Flask(name)
app.run()
```
看起来第一步是创建Flask对象，第二步app.run()就是将对象运行起来，我们可以追踪进去，看看run()方法中究竟是什么

```
def run(self, host='localhost', port=5000, **options):
	from werkzeug import run_simple
   return run_simple(host, port, self, **options)
```
删除非核心的代码就只是运行了werkzeug中的`run_simple()` 方法，可以看到此方法将self作为参数传递，事实上，此方法将当前启动的对象作为网络请求的处理函数，后续所有的网络请求都会调用当前对象进行处理，调用的处理的代码为： `application_iter = app(environ, start_response)`，其中的app就是`run_simple`中传递进去的self。因此会调用当前对象的`__call__()`方法进行请求的处理。

#### 请求处理

```
def __call__(self, environ, start_response):
    return self.wsgi_app(environ, start_response)
```
可以看到网络请求实际执行的是`wsgi_app()`方法进行处理，那么我们再继续看看这个方法是如何处理的呢：

```
def wsgi_app(self, environ, start_response):
    with self.request_context(environ):
        rv = self.preprocess_request()
        if rv is None:
            rv = self.dispatch_request()
        response = self.make_response(rv)
        response = self.process_response(response)
        return response(environ, start_response)
```
在实际网络请求的处理方法中，首先执行`preprocess_request()`执行网络请求处理前的回调函数，然后执行`dispatch_request()`进行请求的处理，得到处理的结果rv，然后调用`make_response()`将返回值转换为Response类型，然后调用`process_response()`方法执行处理后的回调函数以及session处理，最后返回数据。因此我们需要关注的核心就是`dispatch_request()`方法，看看是如何处理网络请求的呢

```
def dispatch_request(self):
	endpoint, values = self.match_request()
   return self.view_functions[endpoint](**values)
```
去除异常处理后，实际代码就只有如下所示的两行，调用`match_request()`看起来是匹配请求，得到end_point与values，根据实际执行`self.view_functions[endpoint](**values)`推测，应该是通过匹配找到处理的方法`self.view_functions[endpoint]`，然后调用处理请求的方法，并将请求对应的参数values传递进去，并最终返回请求处理的结果。

#### 路由规则

那么回过头来思考，服务器怎么会知道请求应该由哪个方法来处理，比如发起一个创建任务的请求，服务器怎么知道调用哪个方法去处理。这种业务相关的事情，明显是需要业务方来指定，再联想到使用flask中我们经常会使用app.route()装饰器指定网络请求的处理，比如flask官网给出的[最小的应用](http://docs.jinkan.org/docs/flask/quickstart.html):

```
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello World!'

if __name__ == '__main__':
    app.run()
```
运行此任务后，访问127.0.0.1/ 会得到返回值'Hello world'，从这个例子中可以看到当服务器收到网络请求时，事实上是通过我们指定的处理方法进行处理。看起来的确是通过@app.route()绑定请求处理的方法，那么我们来看看是具体的代码：

```
def route(self, rule, **options):
    def decorator(f):
        self.add_url_rule(rule, f.__name__, **options)
        self.view_functions[f.__name__] = f
        return f
    return decorator
    
def add_url_rule(self, rule, endpoint, **options):
    options['endpoint'] = endpoint
    options.setdefault('methods', ('GET',))
    self.url_map.add(Rule(rule, **options))
```
代码是标准的装饰器写法，用来装饰特定的方法f，可以看到`self.view_functions[f.__name__] = f`了解到`self.view_functions`为dict类型，存储的是方法名与方法的映射关系，而根据上面的最小应用实例知道，rule就是对应的请求的url，从而推测出`self.add_url_rule(rule, f.__name__, **options)`此方法用于存储url到方法名的映射关系，查看`add_url_rule`实现的代码可以看到，flask中利用rule和endpoint构建成werkzeug中的Rule类型，然后存储为至url_map中，方便后续查找。

从而理顺思路，当请求到达时，我们可以根据url从url_map中获取到方法名（endpoint）与参数，然后根据方法名从`self.view_functions`中获取处理请求的方法，执行此方法用于处理网络请求。

按照这个思路我们去查看`dispatch_request`可以看到先调用`match_request()`方法得到处理的endpoint和values，而此endpoint就是上面说的方法名，values就是请求的参数，然后执行`self.view_functions[endpoint](**values)`获取处理函数并执行，从而验证了我们最初的推测。

url与endpoint的映射关系一般情况下的思路是直接采用普通dict存储，查找时就可以使用url作为key直接找到了。但是这种方式灵活性不足，而且合法的等价url可能无法找到对应的endpoint，可以看到flask中是将url与endpoint用于构建Rule类型并存储至`url_map`中，那么具体查找时是如何做的呢，具体的代码应该是在`match_request()`方法中

```
def match_request(self):
   rv = _request_ctx_stack.top.url_adapter.match()
   request.endpoint, request.view_args = rv
   return rv
```
通过上面的代码可以看到，最终是调用了请求的match()方法来获取到endpoint和参数，而调用者`url_adapter = url_map.bind_to_environ(environ)`，也就是说实际获取endpoint与参数是通过调用`url_map.bind_to_environ(environ).match()`来获取的。通过前面的介绍我们已经知道，url_map中存储的是url与endpoint之间的映射关系，这种映射关系是通过@app.route()进行指定的。而environ为单次请求信息，内部包含请求的url。可以理解为存储信息的对象`url_map`绑定特定的请求信息environ，然后进行匹配match()，即可得到请求对应的endpoint和参数value。

#### 请求信息
前面的介绍可以看到获取当前请求的信息是从` _request_ctx_stack.top`中获取出来的，也就是说请求会被加入请求栈中，栈顶的就是当前请求。可以看一下这个请求栈`_request_ctx_stack`的定义：

```
_request_ctx_stack = LocalStack()
```
通过定义可以看到，确实是一个请求栈，而且是一个多线程隔离的请求中。关于LocalStack的具体信息可以参考[之前的博客](https://hustyichi.github.io/2018/08/22/LocalProxy-in-flask/)，在这边我们简单理解LocalStack就是一个多线程安全的栈，提供push,pop,top的方法。而栈中元素必然就是单个请求了，元素类型为`_RequestContext`：

```
class _RequestContext(object):
    def __init__(self, app, environ):
        self.app = app
        self.url_adapter = app.url_map.bind_to_environ(environ)
        self.request = app.request_class(environ)
        self.session = app.open_session(self.request)
        self.g = _RequestGlobals()
        self.flashes = None

    def __enter__(self):
        _request_ctx_stack.push(self)

    def __exit__(self, exc_type, exc_value, tb):
        if tb is None or not self.app.debug:
            _request_ctx_stack.pop()
```
看到单个请求使用app和environ进行初始化，其中app就是Flask创建的实例，environ为单次请求具体信息。其中就包含`url_adapter`属性，前面已经介绍过，就是通过`url_adapter.match()`进行匹配后获取到endpoint和value的，从而获取到请求处理的方法的，从而与前面的解释相互印证。那么现在还剩下一个问题，flask是什么时候将`_RequestContext`加入到`_request_ctx_stack`中的呢？让我们回头看一下`wsgi_app()`方法，使用with进行调用：

```
def wsgi_app(self, environ, start_response):
    with self.request_context(environ):
        rv = self.preprocess_request()
        if rv is None:
            rv = self.dispatch_request()
        response = self.make_response(rv)
        response = self.process_response(response)

def request_context(self, environ):
    return _RequestContext(self, environ)
```
可以看到调用了`request_context()`方法，此方法创建了一个`_RequestContext`对象，然后使用with的调用方式，会执行`_RequestContext`的`__enter__()`魔术方法，即会发现`_request_ctx_stack.push(self)`，将创建的`_RequestContext `加入请求栈`_request_ctx_stack`中，然后在执行处理结束的时候，执行`__exit__()`方法，将请求从请求栈中移除。至此，一切豁然开朗。

## 总结
在flask启动中，需要创建一个Flask对象，然后调用此对象的`run()`方法启动应用，在`run()`方法中，会将当前的Flask对象注册为网络请求处理对象，后续的网络请求都会调用当前对象的`__call__()`方法进行处理，而`__call__()`实际调用的是`wsgi_app()`方法进行处理。在处理请求时，会利用请求的信息environ创建单个请求对象`_RequestContext`，此对象会存储单个请求相关的所有信息，然后将请求对象加入到`_request_ctx_stack`请求栈中，在处理过程中，可以通过`_request_ctx_stack.top`获取本次请求对象。在请求对象`_RequestContext `中存在`url_adapter`属性，此属性是存储了url到endpoint映射关系的url_map绑定请求信息environ后创建的对象，在处理请求中，可以调用`url_adapter.match()`方法得到当前请求对象的endpoint，默认情况下endpoint是处理请求的方法名，利用endpoint可以通过`view_functions[endpoint]`得到处理请求的具体方法, 此方法是提前通过@app.route()进行注册指定的。 执行处理方法即可得到请求处理的结果，将请求处理的结果转换为标准的Response对象，然后返回给客户端，就完成单次网络请求的处理。
