---
layout: post
title: Lumen框架在同一域名多应用环境下的部署问题
description: php, lumen, index.php
categories: [Web]
tags: [PHP, Lumen]
shortinfo: Lumen框架在同一域名多应用环境下的部署问题
---

### **在同一域名下部署多个应用时，使用Lumen框架遇到的问题**

官方文档的推荐一般是通过配置网站的根目录至```wwwroot/LUMEN/public```。如果服务器上只有Lumen一个应用是没有问题的，但是如果服务器上同时部署了多个应用，想通过```wwwroot```的相对路径```http://DOMAIN/lumen/public/```来访问框架url时就会出现```NotFoundHttpException```的异常。

根据错误提示，定位到文件
```LUMEN/trunk/vendor/laravel/lumen-framework/src/Concerns/RegistersExceptionHandlers.php中370行：```

```
/**
 * Dispatch the incoming request.
 *
 * @param  SymfonyRequest|null  $request
 * @return Response
 */
public function dispatch($request = null)
{
    list($method, $pathInfo) = $this->parseIncomingRequest($request);

    try {
        return $this->sendThroughPipeline($this->middleware, function () use ($method, $pathInfo) {
            if (isset($this->routes[$method.$pathInfo])) {
                return $this->handleFoundRoute([true, $this->routes[$method.$pathInfo]['action'], []]);
            }

            return $this->handleDispatcherResponse(
                $this->createDispatcher()->dispatch($method, $pathInfo)
            );
        });
    } catch (Exception $e) {
        return $this->sendExceptionToHandler($e);
    } catch (Throwable $e) {
        return $this->sendExceptionToHandler($e);
    }
}

/**
 * Parse the incoming request and return the method and path info.
 *
 * @param  \Illuminate\Http\Request|null  $request
 * @return array
 */
protected function parseIncomingRequest($request)
{
    if ($request) {
        $this->instance(Request::class, $this->prepareRequest($request));
        $this->ranServiceBinders['registerRequestBindings'] = true;

        return [$request->getMethod(), $request->getPathInfo()];
    } else {
        return [$this->getMethod(), $this->getPathInfo()];
    }
}
```

框架中的LUMEN/public/index.php默认是```$app->run();```是不传$request的，对应调用的方法如下：

```
/**
 * Get the current HTTP request method.
 *
 * @return string
 */
protected function getMethod()
{
    if (isset($_POST['_method'])) {
        return strtoupper($_POST['_method']);
    } else {
        return $_SERVER['REQUEST_METHOD'];
    }
}

/**
 * Get the current HTTP path info.
 *
 * @return string
 */
protected function getPathInfo()
{
    $query = isset($_SERVER['QUERY_STRING']) ? $_SERVER['QUERY_STRING'] : '';

    return '/'.trim(str_replace('?'.$query, '', $_SERVER['REQUEST_URI']), '/');
}
```

可见通过此方法获取的path，如果存在多应用的情况下，$this->routes获取是错误的。

### **解决方法如下：** ###
修改```LUMEN/public/index.php```为

```
$request = Illuminate\Http\Request::capture();
$app->run($request);
```

---

**参考**

* [https://segmentfault.com/a/1190000002724037](https://segmentfault.com/a/1190000002724037){:target="_blank"}
* [https://segmentfault.com/a/1190000004460637](https://segmentfault.com/a/1190000004460637){:target="_blank"}
* [http://stackoverflow.com/questions/29728973/notfoundhttpexception-with-lumen](http://stackoverflow.com/questions/29728973/notfoundhttpexception-with-lumen){:target="_blank"}


