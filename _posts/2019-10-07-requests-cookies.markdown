---
layout: post
title: 'Requests cookie 源码分析'
subtitle:   "Requests cookie source codes reading"
date:       2019-10-07 11:10:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - python
---

## 背景介绍

之前介绍过 [Requests 源码阅读](https://hustyichi.github.io/2019/08/10/requests-codes/) ，对 Requests 库中单次网络请求的完整流程进行过介绍，对于某些细节则直接掉过了。这次补上 cookie 相关的源码分析，丰富相关细节。

cookie 是一串数据，在网络请求中会带上相关数据，用于请求者身份的标识。如果对 cookie 基础原理还不了解，可以自行搜索。

Requests 中对 cookie 的管理依赖 Python 内置的 cookie 管理库 http.cookielib ，对内置库中具体的代码实现，不深入细节，只是大致介绍相关的功能。

## cookie 使用流程

从 Requests 的 [官方文档](https://requests.kennethreitz.org//zh_CN/latest/index.html) 的功能特性可以看到，与 cookie 相关的功能特性包括如下：

- 带持久 cookie 的会话
- 优雅的 key/value cookie

对于持久化 Cookie 的实现，在前面的源码阅读中已经分析过了，是利用 `Session` 进行 cookie 的保存与复用。而优雅的 key/value Cookie 指的是在发起请求时，可以通过 dict 类型指定 cookie 数据，Requests 库将 cookie 的指定接口设计得更加优雅了，仅此而已。下面我们从 cookie 的使用流程来具体分析下 Requests 中 cookie 的使用。

在完整的网络请求中，对 cookie 的使用主要包括如下三个方面：

- 网络请求准备阶段，需要构造 cookie，存储至 `PrepareRequest` 对象中，方便进行网络请求时使用；
- 网络请求中，需要从 `PrepareRequest` 中获取 cookie，进行实际的网络请求；
- 网络请求返回时，需要从构造 cookie 存储至 `Response` 中，并从 `Response` 中存储至 `Session` 中，方便进行 cookie 的持久化；

#### 网络请求准备阶段

在网络请求阶段，需要构造所需的 cookie ，而 cookie 的来源包括：

1. 手工指定 cookie；
2. Session 自动保存 cookie；

因此在准备阶段，需要将多种类型的 cookie 进行合并，从而在请求阶段可以直接使用，可以查看具体的实现：

```python
# class Session

def prepare_request(self, request):
    # 获取本次请求 Requests 中的 cookie
    
    cookies = request.cookies or {}
		
    # 对 Requests 中 cookie 数据进行处理，生成 CookieJar 类型数据
    
    if not isinstance(cookies, cookielib.CookieJar):
        cookies = cookiejar_from_dict(cookies)

    # 将 Requests 中 cookie 与 Session 中 cookie 合并
    
    merged_cookies = merge_cookies(
        cookiejar_from_dict(RequestsCookieJar(), self.cookies), cookies)

```

可以看到准备阶段比较简单，只是进行 CookieJar 数据的构造与合并。最终生成的数据为 `RequestsCookieJar` 类型，此数据中会保存当前会话中所有的 cookie，后续再具体介绍。

在此阶段中，依赖 cookie 模块提供的接口为 `cookiejar_from_dict()`  和 `merge_cookies()` ，其中 `cookiejar_from_dict` 是用于根据 dict 类型的数据生成 `RequestsCookieJar` ，前面提到的优雅的 key/value cookie 的秘诀也就在于此。而 `merge_cookies` 则用于将不同来源的 cookie 进行合并，方便后续使用。后续对接口的实现进行具体介绍。

#### 网络请求阶段

在实际的网络请求阶段，对于非块的网络请求，可以看到代码如下所示：

```python
# class HTTPAdapter

def send(self, request, stream=False, timeout=None, verify=True, cert=None, proxies=None):
    conn = self.get_connection(request.url, proxies)

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
```

可以看到并没有显示使用 cookie，那么 cookie 是如何在网络请求中使用的呢？如果不在请求时显式传递，那么必然是被存储到其他部分里面进行传递了。稍微想一下，就应该能想到，cookie 应该是被放在 `request.headers` 里面了。那么必然是在准备阶段就放进去了，查看下相关实现：

```python
# class PreparedRequest

def prepare_cookies(self, cookies):
    # 在类型不是 CookieJar 类型时，构造出 CookieJar 类型的数据
    
    if isinstance(cookies, cookielib.CookieJar):
        self._cookies = cookies
    else:
        self._cookies = cookiejar_from_dict(cookies)

    # 根据 CookieJar 类型的数据生成实际网络请求中发送过去的 cookie header 字符串数据
    
    cookie_header = get_cookie_header(self._cookies, self)
    
    # 将生成的 cookie header 数据放入 request.headers 里面
    
    if cookie_header is not None:
        self.headers['Cookie'] = cookie_header
```

可以看到此阶段主要的工作是将准备阶段生成的 `RequestsCookieJar` 数据通过 `get_cookie_header()` 接口转换为 cookie header 数据。后续对此接口进行具体介绍。

#### 网络请求返回阶段

在网络请求返回阶段，需要将 cookie 从返回值中存储至 `Response` 中，可以看到具体的实现如下所示：

```python
# class HTTPAdapter

def build_response(self, req, resp):
    response = Response()
    
    # 从返回值中提取 cookie 存储至 Response 中
    
    extract_cookies_to_jar(response.cookies, req, resp)

    return response
```

可以看到此方法中利用 cookie 模块中的 `extract_cookies_to_jar()` 方法将网络请求的返回值中提取 cookie 存储至 `Response` 中，构造标准返回值。

在本次会话中，会从本次的返回值中获取 cookie 存储至会话中，方便后续复用，可以看到具体的代码实现如下所示：

```python
# class Session

def send(self, request, **kwargs):
    # 发起网络请求
    
    r = adapter.send(request, **kwargs)
    
    # 从返回值中提取 cookie 存储至当前会话中
    
    extract_cookies_to_jar(self.cookies, request, r.raw)
    return r
```

可以看到调用同样的方法 `extract_cookies_to_jar()` 从 Response 中提取 cookie 存储至当前会话中，后续对实际的提取方法进行具体介绍。

## 基础数据类型

在完整流程的介绍中，持续出现 CookieJar 类型，此数据类型是 Python 系统库中 cookie 管理中的内置类型。

而实际在 Requests 中使用的 `RequestsCookieJar` 类型，此类型继承自 Python 系统库中的 CookieJar 类型，可以用于保存所有的 cookie 数据。`RequestsCookieJar` 的功能与 Python 系统库中的 CookieJar 类型基本一致，只是增加了一系列类似 dict 的接口，提高了易用性。具体实现如下所示： 

```python
class RequestsCookieJar(cookielib.CookieJar, MutableMapping):
    def iterkeys(self):
        for cookie in iter(self):
            yield cookie.name

    def keys(self):
        return list(self.iterkeys())

    def itervalues(self):
        for cookie in iter(self):
            yield cookie.value

    def values(self):
        return list(self.itervalues())

    def iteritems(self):
        for cookie in iter(self):
            yield cookie.name, cookie.value

    def items(self):
        return list(self.iteritems())
    
    def __getitem__(self, name):
        return self._find_no_duplicates(name)

    def __setitem__(self, name, value):
        self.set(name, value)

    def __delitem__(self, name):
        remove_cookie_by_name(self, name)
```

通过上面的实现可以看到 `RequestsCookieJar` 继承自 CookieJar ，因此具备 CookieJar 相关的功能，同时 `RequestsCookieJar` 继承自 MutableMapping 类，此类是可变 dict 需要实现的接口，在具体的实现中，可以看到 `RequestsCookieJar` 实现了 dict 相关的接口，提高了易用性。

## cookie 接口实现

在上面的流程中，介绍了 cookie 基础模块需要提供的接口，下面分别对这些接口的具体实现进行介绍：

#### `cookiejar_from_dict`

此接口是用于根据字典数据生成 `RequestsCookieJar` 数据，下面可以查看具体的实现：

```python
def cookiejar_from_dict(cookie_dict, cookiejar=None, overwrite=True):
  
    # 构建待返回的 `RequestsCookieJar` 数据
    
    if cookiejar is None:
        cookiejar = RequestsCookieJar()

    # 保存 cookie_dict 中的 cookie 数据
    
    if cookie_dict is not None:
        names_from_jar = [cookie.name for cookie in cookiejar]
        for name in cookie_dict:
            if overwrite or (name not in names_from_jar):
                cookiejar.set_cookie(create_cookie(name, cookie_dict[name]))

    return cookiejar
```

上面的实现中，构建了 `RequestsCookieJar` 数据，并将 cookie_dict 中的数据通过 `create_cookie()` 生成 `Cookielib.Cookie` 类型的数据，并调用 `CookieJar.set_cookie()` 方法将生成的 Cookie 数据保存至 `RequestsCookieJar` 中，即可得到包含 cookie 数据的 `RequestsCookieJar` 类型。

#### `merge_cookies`

此接口用于合并 `RequestsCookieJar` 类型的数据，最终得到一个包含了两者的 cookie 数据的 `RequestsCookieJar` 数据。具体的实现如下所示：

```python
def merge_cookies(cookiejar, cookies):
    if not isinstance(cookiejar, cookielib.CookieJar):
        raise ValueError('You can only merge into CookieJar')

    # 如果是 dict 类型的数据，可以调用 `cookiejar_from_dict` 生成包含两者数据的 `RequestsCookieJar`
    
    if isinstance(cookies, dict):
        cookiejar = cookiejar_from_dict(
            cookies, cookiejar=cookiejar, overwrite=False)
    # 如果是 `CookieJar` 类型，调用`update()` 方法或 `set_cookie()` 合并数据
    
    elif isinstance(cookies, cookielib.CookieJar):
        try:
            cookiejar.update(cookies)
        except AttributeError:
            for cookie_in_jar in cookies:
                cookiejar.set_cookie(cookie_in_jar)

    return cookiejar
```

在上面的实现中，可以支持合并 cookie 数据，被合并的 cookie 数据支持 CookieJar 类型，也支持 dict 类型。

#### `get_cookie_header`

此接口用于根据 `RequestsCookieJar` 以及实际的网络请求 `Request`  对象生成实际网络请求发送的 cookie header 的数据，实际实现如下：

```python
def get_cookie_header(jar, request):
    # 根据 Requests 库中的网络请求 request 构建系统库需要的 `urllib2.Request`
    
    r = MockRequest(request)
    # 调用系统库中的 `add_cookie_header()` 方法利用请求信息中保存的 cookie 生成 cookie header 字符串，并保存至 `MockRequest` 数据中
    
    jar.add_cookie_header(r)
    # 获取 `MockRequest` 中的 cookie header 字符串数据
    
    return r.get_new_headers().get('Cookie')
```

传入的参数是 `RequestsCookieJar` 对象与 `Request` 对象，其中 `RequestsCookieJar` 对象中保存的是当前会话中所有的 cookie，`Request` 中保存的是本次请求相关的信息，包括请求的域名， 路径等信息，根据 `Request` 对象中的信息即可匹配对应的 cookie，并调用系统库中的 `CookieJar.add_cookie_header()` 方法，生成 cookie header 字符串，用于后续发起网络请求时标识身份信息。

#### `extract_cookies_to_jar` 

此接口用于从网络请求返回值中提取 cookie 信息并保存至 `RequestsCookieJar` 数据中。具体的实现如下所示：

```python
def extract_cookies_to_jar(jar, request, response):
    if not (hasattr(response, '_original_response') and
            response._original_response):
        return
    # 用于根据 Requests 库中的请求数据 request 构建系统库所需的 `urllib2.Request`
    
    req = MockRequest(request)
    # 用于根据 Requests 库中的返回数据 Response 构建系统库所需的 `urllib.addinfourl`
    
    res = MockResponse(response._original_response.msg)
    # 根据返回值与请求值信息，将 cookie 提取至 `RequestsCookieJar` 中
    
    jar.extract_cookies(res, req)
```

可以看到上面的实现主要是调用了系统库中的 `extract_cookies()` 接口用于从网络请求返回值中提取 cookie 信息并保存至 `RequestsCookieJar` 中，同时由于系统库中需要的请求数据类型与返回数据类型与 Requests 库中的类型不一致，因此需要分别构建 `MockRequest` 和 `MockResponse` 进行转换。

## 总结

通过上面的介绍，我们可以从整体流程上了解 Requests 库中 cookie 的管理流程，其中的基础实现都依赖 Python 系统库的实现，这边就没有深入其中进行解析。后续有空进行研究了再额外介绍。

