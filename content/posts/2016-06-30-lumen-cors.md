---
date: "2016-06-30"
title: Lumen CORS
description: php, lumen, lumen cors
slug: lumen-cors
categories:
- Web
tags:
- php
- lumen
comments: true
---

### **使用Lumen框架如何解决跨域问题**
跨域问题想必是每个从事web相关开发的同学经常遇到的问题，特别是现在越来越多的应用采用的前后端分离的开发模式，所以解决跨域问题成了大家必不可少的一项技能。最近在使用Lumen框架开发时也遇到了这个问题。

进入主题之前我们先来个铺垫，首先我们来了解一下什么是跨域，以及一些解决方案 [这是铺垫](http://www.cnblogs.com/davidwang456/p/3977627.html)

有兴趣的同学可以仔细看看，其实W3C标准的解决方案是只要在服务器response的头部带上`Access-Control-Allow-Origin：DOMAIN`(允许跨域访问的域名，`*`代表所有)，不过需要一提的是该方案只支持IE8及以上，以及一些现代浏览器(Fire Fox,Chrome,etc.)。解决跨域比较常见的还有使用jsonp，但这不是本文重点，这里就不具体展开了。

好了，上面啰嗦了那么多，下面进入主题，那么在Lumen框架中，我们怎么解决跨域问题呢?
这边主要参考了[这里](https://gist.github.com/danharper/06d2386f0b826b669552)该大神的方案。
通过运用框架的Middleware和ServiceProvider来解决跨域问题。

新增中间件，`LUMEN/app/Http/Middleware/CorsMiddleware.php`

```php
namespace App\Http\Middleware;

use Closure;

class CorsMiddleware
{
    /**
     * Handle an incoming request.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @return mixed
     */
    public function handle($request, Closure $next)
    {
        $response = $next($request);

        $response->header('Access-Control-Allow-Methods', 'HEAD, GET, POST, PUT, PATCH, DELETE');
        $response->header('Access-Control-Allow-Headers', $request->header('Access-Control-Request-Headers'));
        $response->header('Access-Control-Allow-Origin', '*');

        return $response;
    }
}
```

定义ServiceProvider，`LUMEN/app/Providers/CatchAllOptionsRequestsProvider.php`

```php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;

/**
 * If the incoming request is an OPTIONS request
 * we will register a handler for the requested route
 */
class CatchAllOptionsRequestsProvider extends ServiceProvider
{
	public function register()
	{
	    $request = app('request');

	    if ($request->isMethod('OPTIONS'))
	    {
	    	app()->options($request->path(), function() {
	    		return response('', 200)->header('Access-Control-Allow-Origin', '*');
	    	});
	    }
  	}
}
```

在`LUMEN/bootstrap/app.php`中加载对应的中间件，注册ServiceProvider。

```php
$app->routeMiddleware([
    'cors' => App\Http\Middleware\CorsMiddleware::class,
]);
...
$app->register(App\Providers\CatchAllOptionsRequestsProvider::class);
```

看到这里，问题是已经解决了。但有一个问题我开始一直想不明白，CatchAllOptionsRequestsProvider存在的意义是什么，为什么需要单独处理OPTIONS的请求？不是一个中间件就可以全部处理了吗？
github上也有人跟我有同样的疑问
{{< image src="/assets/images/lumen-cors-1.jpg" alt="lumen-bootstrap-app" position="center" >}}

直到前段时间再去看时才看到下面这位大神的评论
{{< image src="/assets/images/lumen-cors-2.jpg" alt="lumen-bootstrap-app" position="center" >}}

在结合上面的铺垫，原来

> CORS (Cross-Origin Resource Sharing)，W3C制定的跨站资源分享标准。post前会产生一次options嗅探（称之为preflight，但简单请求不会出现）来确认有否跨域请求的权限。

由于在Lumen的路由里并未定义对应的options请求，所以框架直接返回405了。请求并不会被分发，所以不会被中间件处理。因此才需要用`CatchAllOptionsRequestsProvider.php`来获取请求，并对options的请求单独处理。

---

**参考**

* [http://www.cnblogs.com/davidwang456/p/3977627.html](http://www.cnblogs.com/davidwang456/p/3977627.html)
* [https://gist.github.com/danharper/06d2386f0b826b669552](https://gist.github.com/danharper/06d2386f0b826b669552)