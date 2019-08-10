---
layout: post
title: 'Requests 源码阅读'
subtitle:   "Requests source codes reading"
date:       2019-08-10 13:16:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
---

## 基础阶段

之前一直都有在实践中使用 Requests 库，基本上算是 Python 领域网络请求必备的，之前也一直有听过 Http
for human 的说法，Requests 作为一个封装良好的代码库，一直被认为是值得一读的。

在实际的项目中，对 Requests 库有过简单的使用，主要是用于发起 `get()` 和 `post()` 请求，因此在阅读源码之前，先完整过了一遍 [Requests 官方文档](https://2.python-requests.org/en/master/) 。Requests 的使用基本没有特别复杂的地方，由于 Requests 的良好封装，使用者可以直接使用 Requests 提供的几个封装良好的方法即可。

本文选择的版本为目前的 v2.22.0，一般情况下，如果希望先从最基础的版本开始的，可以先从更早版本的代码进行阅读。

## 基础流程

一般情况下，我们使用 Requests 只是为了发起单个网络请求，我们按照最基础的发起单个 `get()` 请求流程去查看内部处理的顺序，大致的流程如下：

1. 创建单个 `Session` 对象；
2. 调用 `Session.request()` 方法发起实际的网络请求，得到返回对象 `Response` 对象；
   - 利用请求的参数构建 `Request` 对象；
   - 利用 `Request` 对象构建出 `PrepareRequest` 对象用于发起网络请求；
   - 将 `PrepareRequest` 对象作为参数调用 `HTTPAdapter` 发起实际网络请求，得到 `Response` 对象的响应；
3. 返回 `Response` 对象作为网络请求返回值；

按照基础的流程来看，Requests 中的几个基础模块包括如下所示：

1. 网络请求参数类，包括 `Request` 和 `PrepareRequest` 
2. 网络请求返回类， `Response` 
3. 会话管理类，`Session` 
4. 实际网络请求处理类，包括 `BaseAdapter` 和 `HTTPAdapter`

事实上，阅读文档的话，还能发现 Requests 中还包含一些其他内容，比如 `Cookie` 管理， `Auth`  身份验证。在这篇文章暂时不详细介绍，后续再单独介绍相关细节。下面分别对现有的主要功能模块进行介绍：

## 网络请求参数类

网络请求参数类主要用于保存用户发起网络请求相关的参数，后续可以将保存的参数的对象 `PrepareRequest` 直接发送给 `HTTPAdapter` 对象进行实际的网络请求。一个令人费解的地方就在于，为什么会需要两个类 `Request` 和 `PrepareRequest` ，而不是直接使用 `PrepareRequest` 呢？

在前面的基础流程中可以看到，构建用于保存网络请求参数的 `PrepareRequest` 类主要包括两步：

```python
p = PreparedRequest()
p.prepare(...)
```

那么看起来直接使用 `PrepareRequest` 就可以进行实际的网络的网络请求了。那么就看下 `Request` 也提供的 `prepare()` 方法做了什么呢？具体的实现如下所示：  

```python
# class Request
def prepare(self):
    p = PreparedRequest()
    p.prepare(
        method=self.method,
        url=self.url,
        headers=self.headers,
        files=self.files,
        data=self.data,
        json=self.json,
        params=self.params,
        auth=self.auth,
        cookies=self.cookies,
        hooks=self.hooks,
    )
    return p
```

可以看到 `Request.prepare()` 只是调用了前面的 `PrepareRequest` 的数据构造的基础流程，和手动构造 `PrepareRequest` 数据是完全等价了，完全可以不使用 `Request` 类构建网络请求参数。

那么接下来我们看下构建请求参数对象的 `PrepareRequest.prepare()` 方法做了什么必要的准备呢？代码如下所示：

```python
# class PrepareRequest
def prepare(self,
            method=None, url=None, headers=None, files=None, data=None,
            params=None, auth=None, cookies=None, hooks=None, json=None):

    self.prepare_method(method)
    self.prepare_url(url, params)
    self.prepare_headers(headers)
    self.prepare_cookies(cookies)
    self.prepare_body(data, files, json)
    self.prepare_auth(auth, url)
    self.prepare_hooks(hooks)
```

看起来是进行不同类型的基础数据的准备，选择其中一个方法查看相关实现：

```python
# class PrepareRequest
def prepare_method(self, method):
    self.method = method
    if self.method is not None:
        self.method = to_native_string(self.method.upper()
```

可以看到 `prepare_method()` 就是将数据进行必要的编解码，将数据转换为与版本无关的数据。其他的 `prepare_url()` , `prepare_headers()` 等方法都做了类似的工作，以及一些其他的数据准备，就不深入准备数据的细节了。

总结下来，网络请求参数类用于保存用户发起网络请求相关的数据，包括method, url, headers 等信息，在数据准备阶段，将数据进行必要的编解码，保存至 `PrepareRequest` 对象中，后续网络请求模块可以直接使用此对象进行实际的网络请求。

## Session 会话管理

理论上我们可以直接通过 `PrepareRequest` 发起网络请求，为什么需要构造一个 Session 对象，在 Session 对象的上下文中发起网络请求呢？

关于这个问题，官方文档中已经有了[相关的介绍](https://2.python-requests.org/en/master/user/advanced/#session-objects) ，使用 Session 对象可以让你跨请求保持参数，在同一个 Session 实例中发出的网络请求可以保持 cookie，同一主机的 TCP 请求会被重用，从而带来性能提升。

在 Requests 中发出网络请求存在着两种方式，分别为如下所示：

1. 使用 `requests.get()` 方式发起网络请求，这种方式可以最终实际调用的代码为：

   ```python
   with sessions.Session() as session:
       return session.request(method=method, url=url, **kwargs)
   ```

   可以看到实际上也是创建了一个新的 `Session` 对象用于发起网络请求

2. 使用 `Session.get()` 方式发起网络请求

   这种方式最终调用 `Session.request()` 发起网络请求

可以看到两种方式发起网络请求最终是殊途同归，但是可以看到明显的不同之处，第一种方式每次都是创建一个新的 Session 对象，必然无法使用 Session 对象提供的跨请求保持参数的特性，第二种方式可以保持使用同一个 Session 发起网络请求，因此是可以进行跨请求保持参数的。

以 [官方文档的 Session 介绍](https://2.python-requests.org/en/master/user/advanced/#session-objects) 提供的网站进行实践可以看到第一种方式确实实现了跨请求保持 cookie。那么接下来的问题就是：Session 是如何跨请求保持 cookie 的呢？

由于请求相关的信息都会保存至 `PrepareRequest` 对象中，那么我们可以跟踪请求参数构建的过程就应该可以看到了：

```python
# class Session
def prepare_request(self, request):
    # 从 Request 对象中获取 cookies
    cookies = request.cookies or {}

    # 在 cookies 类型不是 CookieJar 时进行类型转换
    if not isinstance(cookies, cookielib.CookieJar):
        cookies = cookiejar_from_dict(cookies)

    # 合并本次请求 cookies 和 Session 中的 cookies
    merged_cookies = merge_cookies(
        merge_cookies(RequestsCookieJar(), self.cookies), cookies)

    # 构造实际的网络请求参数
    p = PreparedRequest()
    p.prepare(
        method=request.method.upper(),
        url=request.url,
        files=request.files,
        data=request.data,
        json=request.json,
        headers=merge_setting(request.headers, self.headers, dict_class=CaseInsensitiveDict),
        params=merge_setting(request.params, self.params),
        auth=merge_setting(auth, self.auth),
        cookies=merged_cookies,
        hooks=merge_hooks(request.hooks, self.hooks),
    )
    return p
```

通过上面的实现可以看到，在构建网络请求参数时，调用了 `merge_cookies()` 方法将 Session 中的 cookies 与 本次请求的 cookies 合并了，因此使用同一个 Session 对象发起网络请求时才能实现跨请求保持 cookie。至于其他的参数，可以看到调用了 `merge_setting()` 方法进行了合并。

那么看到了保持参数的秘密在于将本次请求参数与 Session 中的请求参数进行合并，那么接下来的问题就来了，类似 cookie 这样的请求参数是何时写入 Session 中的呢？

初步猜测应该是在请求返回的时候写入的，如果不确定，也可以比较容易确认，盯着 `Session.cookies` 属性就好了，必然会存在写入的情况。追踪到的实现如下所示： 

```python
# class Session
def send(self, request, **kwargs):
    # 发起实际的网络请求
    adapter = self.get_adapter(url=request.url)
    r = adapter.send(request, **kwargs)

    # 返回值的历史中存在 cookie，保持至 Session.cookies 中
    if r.history:
        for resp in r.history:
            extract_cookies_to_jar(self.cookies, resp.request, resp.raw)

    # 当前请求的 cookies 保持至 Session.cookies 中
    extract_cookies_to_jar(self.cookies, request, r.raw)

    return r
```

总结下来，利用 Session 对象，确实可以实现跨请求的参数保持，实现的方式是通过在发起请求时，合并 Session 中的请求参数与本次请求的参数。而在网络请求返回时，会将请求的必要信息，比如 cookie 保存在 Session 中。

## 实际的网络请求

实际的网络请求类包括 `BaseAdapter` 和 `HTTPAdapter` 。其中 `BaseAdapter` 只定义了基础的接口，而  `HTTPAdapter` 是对 `BaseAdapter` 的实现。

从名字上可以大致猜测实现的是适配器模式，即将原有接口进行封装后提供出新的接口。之前了解到 Requests 是依赖 urllib3 进行实际的网络请求的，因此可以合理估计是对 urllib3 接口的封装，提供一个便于处理现有的 `PrepareRequest` 类型请求，返回 `Response` 结构的接口。

根据 `BaseAdapter` 可以了解到，需要提供的接口为 `send()` 和 `close()` ，其中 `send()` 是用于发起网络请求，`close()` 用于执行清理。下面可以看下 `HTTPAdapter` 实现的网络请求处理：

```python
# class HTTPAdapter
def send(self, request, stream=False, timeout=None, verify=True, cert=None, proxies=None):
  	# 获取网络连接
    conn = self.get_connection(request.url, proxies)

    # 判断是否是块网络请求
    chunked = not (request.body is None or 'Content-Length' in request.headers)

    if not chunked:
        # 非块请求直接调用 url_open() 进行网络请求
        resp = conn.urlopen(
            method=request.method,
            url=url,
            body=request.body,
            headers=request.headers,
            redirect=False,
            assert_same_host=False,
            preload_content=False,
            decode_content=False,
            retries=self.max_retries,
            timeout=timeout
        )
    else:
      	# 块请求需要构造数据，按照字节发送数据
        low_conn = conn._get_conn(timeout=DEFAULT_POOL_TIMEOUT)
        low_conn.putrequest(request.method, url, skip_accept_encoding=True)

        for header, value in request.headers.items():
            low_conn.putheader(header, value)
        low_conn.endheaders()

        for i in request.body:
            low_conn.send(hex(len(i))[2:].encode('utf-8'))
            low_conn.send(b'\r\n')
            low_conn.send(i)
            low_conn.send(b'\r\n')
        low_conn.send(b'0\r\n\r\n')

        r = low_conn.getresponse(buffering=True)
        resp = HTTPResponse.from_httplib(
            r,
            pool=conn,
            connection=low_conn,
            preload_content=False,
            decode_content=False
        )

    # 根据网络请求 PrepareRequest 对象和网络请求返回值构造标准返回对象 Response
    return self.build_response(request, resp)
```

对于一般情况下的非块网络请求，可以看到只是调用了 urllib3 提供的 `urlopen()` 发起网络请求，并利用返回值构造出 Requests 的标准输出类型 `Response` 。

## 网络请求返回值

网络请求的返回值用于提供标准的返回类型。是对网络请求返回值的封装。

通过前面的介绍可以知道，实际的网络请求是通过调用 `HTTPAdapter.send()` 发起的，最后在获取到返回值后调用 `HTTPAdapter.build_response()` 将返回值构造成 `Response` 对象，因此对于 `Response` 对象的了解主要关注 `HTTPAdapter.build_response()` 的实现即可。下面查看相关实现：

```python
# class HTTPAdapter
# 根据请求对象 PrepareRequest 和 urllib3 的返回值构造出 Response
def build_response(self, req, resp):
    response = Response()

    # 从 urllib3 的返回值中获取网络状态码
    response.status_code = getattr(resp, 'status', None)
    # 从 urllib3 中获取 headers 信息
    response.headers = CaseInsensitiveDict(getattr(resp, 'headers', {}))
    response.encoding = get_encoding_from_headers(response.headers)
    
    # 存储 urllib3 的原始返回值 resp
    response.raw = resp
    response.reason = response.raw.reason

    # 存储网络请求的 url
    if isinstance(req.url, bytes):
        response.url = req.url.decode('utf-8')
    else:
        response.url = req.url

    # 存储 cookie 至 Response 中
    extract_cookies_to_jar(response.cookies, req, resp)

    # 将之前的请求信息存储至 Response 中
    response.request = req
    response.connection = self

    return response
```

通过上面的实现，通过将网络请求参数和 urllib3 的返回值存储至 `Response` 对象中，后续可以通过标准的接口将数据提供给用户。具体提供了哪些便于使用的接口，可以参考 [官方文档的 Response 部分](https://2.python-requests.org/en/master/user/quickstart/#response-content) 。至于如何实现这些便于使用的接口，感兴趣的可以自行去查看相关接口的实现。

## 总结

通过上面的介绍，可以看到对 Requests 的基础流程有了足够的了解。至于各个部分的一些实现细节，这边没有过多涉及。如果在实践中有相关的需求，可以自行去查看各个模块的具体实现即可。