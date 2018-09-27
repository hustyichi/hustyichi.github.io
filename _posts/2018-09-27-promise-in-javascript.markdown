---
layout: post
title: 'Promise 异步调用'
subtitle:   "Promise in javascript"
date:       2018-09-27 21:55:00
author:     "Bryan"
header-mask: 0.3
catalog:    true
tags:
    - frontend
    - javascript
---

## Promise 介绍
Promise是ES6提供的语法，可以将回调函数的写法变得更优雅，避免写出Callback Hell。如果不了解回调地狱可以阅读[此文档](http://callbackhell.com/)，简单来说，就是一层嵌套另一层的回调，出现多层回调时，代码就惨不忍睹了。按照之前老式的写法，为了执行回调可能会写出如下所示的代码：

```
function successCallback(result) {
  console.log("Audio file ready at URL: " + result);
}

function failureCallback(error) {
  console.log("Error generating audio file: " + error);
}

createAudioFileAsync(audioSettings, successCallback, failureCallback)
```
一般会将需要回调的函数作为参数传递进去，在需要执行的地方执行。按照Promise的写法，上面的代码就会变成如下：

```
createAudioFileAsync(audioSettings).then(successCallback, failureCallback);
```
回调函数不是作为参数进行传递，而是通过then语法进行执行。也许你会觉得好像提升很有限，但是对于看看下面的回调地狱代码：

```
doSomething(function(result) {
  doSomethingElse(result, function(newResult) {
    doThirdThing(newResult, function(finalResult) {
      console.log('Got the final result: ' + finalResult);
    }, failureCallback);
  }, failureCallback);
}, failureCallback);
```
如果采用Promise的写法，上面的代码就会变成如下所示:

```
doSomething().then(function(result) {
  return doSomethingElse(result);
})
.then(function(newResult) {
  return doThirdThing(newResult);
})
.then(function(finalResult) {
  console.log('Got the final result: ' + finalResult);
})
.catch(failureCallback);
```
瞬间就可以看到质的提升有没有。接下来就来详细了解下Promise

## Promise 机制
一般情况下，我们会有两种状态的回调，处理成功后的回调以及处理失败的回调，按照之前的写法，我们会执行业务代码，根据业务执行的情况来决定执行成功的回调还是失败的回调。

在Promise机制中，会采用状态来表示执行的情况。Promise对象最初都是`pending`状态，当执行成功时，状态会切换为`fulfilled`，一般也称为`resolved`状态，当执行失败时，状态会切换为`rejected`。当状态变为`resolved`，采用then写上回调函数后，就会执行对应对应的successCallback，当状态变为`rejected`，就会执行then中的failureCallback。


## Promise 用法

#### new Promise
创建Promise实例的用法如下所示：

```
var promise = new Promise(function(resolve, reject) {
  var status = do_action()
  
  if (is_success(status)){
    resolve(value);
  } else {
    reject(error);
  }
});
```
观察上面的代码可以看到，可以使用function构建Promise，传入两个方法参数resolve和reject，其中resolve方法用于将未完成状态变为成功状态，即`pending`到`resolved`，而reject方法用于将未完成状态变为失败状态，即`pending`到`rejected`。上面的代码执行完业务代码`do_action`后，根据状态决定需要执行成功后的回调还是失败处理的回调。


#### Promise then
前面可以看到，promise执行结束后，状态会变成`resolved`或`rejected`，如果有需要执行回调函数，可以使用then语句，指定回调函数。then方法支持两个参数，第一个参数为成功执行后的回调函数，第二个参数为执行失败后的回调函数，第二个参数可以为空。即如果promise状态变为`resolved`，会执行第一个参数执行的回调函数，如果promise状态变为`rejected`，会执行第二个参数执行的回调函数。上面的代码配合then语句如下所示：

```
promise.then(successCallback, failureCallback)
```

回调函数如果返回promise，那么可以继续使用then语句，写出链式的语法，类似：

```
promise.then(successCallback, failureCallback).then(successCallback2, failureCallback2)
```
#### Promise catch
对于某些情况下，只想对异常情况进行处理，即只处理失败的情况，此时可以采用catch语句。事实上，catch语句等同于then(null, failureCallback)，使用语句类似如下所示：

```
promise.catch(failureCallback)
```
同时要注意，在执行异步操作时，如果抛出了异常，此时promise状态会变为`rejected`，这时候就会执行failureCallback。

#### Promise finally
对于某些情况下，不管成功或失败都会执行同样的处理，此时就可以使用finally语句，使用语句类似入下所示：

```
promise.finally(callback)
```

#### Promise all
Promise.all()用于组合多个Promise，通过Promise.all()可以将多个Promise组合成一个新的Promise，使用如下所示：

```
new_promise = Promise.all([promise1, promise2, promise3])
```
可以看到上面的`new_promise`是由`promise1`, `promise2`, `promise3`组合产生的，在实际执行中，`new_promise`的状态是由`promise1`, `promise2`, `promise3`的状态决定的，决定的策略是：

- 如果`promise1`, `promise2`, `promise3`的状态都转换为`resolved`，`new_promise`的状态就会转换为`resolved`
- 如果`promise1`, `promise2`, `promise3`的状态有一个转换为`rejected`，那么`new_promise`的状态就会转换为`rejected`

#### Promise race
Promise.race()用于组合多个Promise，使用如下所示：

```
new_promise = Promise.race([promise1, promise2, promise3])
```
可以看到上面的`new_promise`是由`promise1`, `promise2`, `promise3`组合产生的，在实际执行中，只要`promise1`, `promise2`, `promise3`任意一个状态改变，那么`new_promise`状态就会跟着改变，而且状态与第一个改变的状态一致。

## 总结
利用ES6提供的Promise机制，可以将原来回调函数的嵌套结构转换为链接结构，可读性得到了较大的提高。如果希望代码更加优雅，可以采用async，await语法，如果有空，后续再写文章介绍。
