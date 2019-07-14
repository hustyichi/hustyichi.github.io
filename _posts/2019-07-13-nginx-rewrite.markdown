---
layout: post
title: 'Nginx rewrite 机制梳理'
subtitle:   "Nginx rewrite introduction"
date:       2019-07-13 17:37:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - deploy

---

## 基础介绍

最近在做前端 SPA 的部署时，采用 nginx 作为代理，在进行前端资源的访问时，使用了 nginx rewrite 机制，因为不熟悉相关机制，导致折腾了好一会儿。nginx 官网 [rewrite 机制](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#rewrite) 介绍得相当简略，对新手真心不友好，因此自己梳理一下，希望能帮到后续入坑的小伙伴们。

众所周知，nginx 在项目部署中，是作为一个代理服务器存在的。用户需要访问特定的网络资源时，访问特定的 URI，对于 部署了 nginx 服务的项目，此时请求会被发送给 nginx ，nginx 会根据请求的 URI 将请求转发给特定的服务器。大部分情况下， nginx 都是直接转发 URI 给特定的服务器，某些情况下，nginx 会需要修改 URI 地址后再访问，此时就可以使用 nginx 的 rewrite 机制来达到这个目的。比如用户发来的请求的 URI 地址为 `/user/abc` , 但是服务器实际接受的请求 URI 形式为 `/show?user=abc` ，此时就可以使用 nginx rewrite 进行转换。

## Nginx 请求处理阶段

在介绍 nginx 的 rewrite 机制之前，可以先来了解下 nginx 请求的处理阶段，对于后续理解 rewrite 的机制有很大的帮助。参考文章 [你真的了解 Nginx rewrite 么](https://cloud.tencent.com/developer/news/65131) 提供的流程图：

![request](/img/in-post/nginx-rewrite/request.jpeg)

其中与 rewrite 机制相关的主要是  `SERVER_REWRITE` , `FIND_CONFIG` , `REWRITE` 和 `POST_REWRITE` 四个阶段，四个阶段处理的事务分别为：

- `SERVER_REWRITE` 执行 server 模块中的 rewrite 操作
- `FIND_CONFIG` 根据当前的 URI 进行 location 匹配
- `REWRITE` 执行匹配到的 location 中的 rewrite 操作
- `POST_REWRITE` 根据 location 中 rewrite 操作的结果决定下一个阶段

下面以一个简单的例子来说明上面的过程：

```nginx
server {
        listen       80;
        rewrite ^/user/(.*)$  /show/$1 break;

        location /show {
            rewrite ^/show/(.*)$ /?user=$1 break;
        }
}
```

上面的例子中存在两个 rewrite 操作，其中 `rewrite ^/user/(.*)$  /show/$1 break;` 处于 server 模块，而 `rewrite ^/show/(.*)$ /?user=$1 break;` 处于 location 模块。

因此用户访问 `/user/jack` 时首先会处于 `SERVER_REWRITE` 阶段，此时会将当前 URI 修改为 `/show/jack` ， 然后进入 `FIND_CONFIG` 阶段，此时进行 location 匹配，匹配到下面的 `/show` 开头的 location，此时进入 `REERITE` 阶段，执行 location 中的 rewrite 操作，将 URI 修改为 `/?user=jack` , 然后进入 `POST_REWRITE` 阶段，判断是需要进入下一阶段  `PREACESS` ，之后进行资源访问相关的阶段。

## Rewrite 机制

上面的处理过程已经看到了 rewrite 可以将当前的 URI 修改，并根据新的 URI 进行资源的访问，现在就来具体介绍 rewrite 具体命令。命令的参数如下所示：

```nginx
rewrite regex replacement [flag];
```

其中  `regex` 为 PCRE 的正则表达式，用于匹配当前的 URI，`replacement` 是将当前 URI 替换的字符串，`flag` 用于指示后续操作。其中 `regex` 与 `replancement` 就不用过多介绍了，具体看正则表达式的语法即可，只是一个简单的匹配和替换。`flag` 的可选字段如下：

- `last` last 会停止当前模块中后续的其他命令，根据新的 URI 进行新一轮的 location 匹配，根据上面的处理阶段理解，last 字段会在 `POST_REWRITE` 之后重新进入 `FIND_CONFIG` 阶段；
- `break` break 会停止当前模块后续的其他命令，并根据新的 URI 进入 `PREACCESS` 阶段；
- `redirect` redirect 会返回请求方临时重定向，此时请求方收到的返回 code 为 302，后续的命令也都会不会执行；
- `permanent`  permanent 会返回请求方永久重定向，此时请求方收到的返回 code 为 301，后续的命令也不会执行；
- flag 不指定 如果不指定 flag , 此时会继续执行后续命令，在命令执行结束后进行新一轮的 location 匹配，因此会在 `POST_REWRITE` 之后进入 `FIND_CONFIG` 阶段

根据上面的介绍，就可以理解前面的例子中情况了，location 中 rewrite 操作  rewrite ^/show/(.*)$ /?user=$1 break;  执行完成后会根据新的 URI 进行资源的访问，如果需要进行测试，可以将 flag 修改为 last，此时就会进入 `FIND_CONFIG` 阶段重新进行 location 的匹配了。

## 注意事项

#### 请求参数

在前面的介绍中，一直没有提到请求参数的问题，比如网络请求可能为 `/show?user=jack` 那么进行 rewrite 时真正处理的是 `/show` ，参数是如何处理的呢？

默认情况下参数是会被自动默默添加到 replacement 参数后面的，比如对于请求 `/show?user=jack` 执行 `rewrite /show /show_v2` 此时请求会被修改为 `/show_v2?user=jack`

如果不想要默默地添加参数，此时可以在 replacement 的之后以 `?` 结尾，此时就不会自动添加参数了，比如同样的请求，如果执行 `rewrite /show /show_v2?` 此时请求会被修改为 `/show_v2` 

#### nginx 调试

nginx 在新手调试 rewrite 时，会时不时写错，此时是很抓狂的，因为结果不对，但是有没有办法知道发生了什么，完全靠猜。这边推荐两个很实用的功能：

1. `nginx -t` 测试配置文件

   `nginx -t` 可以测试当前的配置文件，如果配置文件中存在着明显的语法错误，此时会有提示

2. 开启 nginx 调试日志

   在 nginx 配置中的 http 模块中增加 `rewrite_log on;` 开启 nginx 日志

   同时在 server 中增加 `error_log  /var/logs/nginx/error.log notice;` 指定日志存储的文件以及日志等级。

   更新上面的 nginx 配置之后，再执行 `nginx -s reload` 更新配置就可以看到 nginx 相关的日志了。实践中发现 Mac 下如果目录中没有此文件貌似会报错，不确定是不是所有平台都是如此，手工增加文件即可

#### 循环调用

根据前面的介绍，某些情况下，rewrite 执行完后会重新进入 location 匹配阶段，此时可能又执行 rewrite，然后循环往复。这样可能会进入无限循环。为了避免这种情况，nginx 会在循环 10 次时之后，直接返回 500 异常。

因此出现 500 异常时，检查下 nginx 的日志就很有用了。

