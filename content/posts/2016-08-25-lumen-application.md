---
date: "2016-08-25"
title: Lumen框架在同一域名多应用环境下的部署问题
description: php, lumen, laravel, .env
slug: lumen-application
categories:
- Web
tags:
- php
- lumen
comments: true
---

### **在同一域名下部署多个应用时，使用Lumen框架遇到的问题**

官方文档的推荐一般是通过配置网站的根目录至`wwwroot/LUMEN/public/`。如果服务器上只有Lumen一个应用是没有问题的，但是如果服务器上同时部署了多个应用，想通过`wwwroot`的相对路径`http://DOMAIN/lumen/public/`来访问框架url时就会出现`NotFoundHttpException`的异常。

根据错误提示，定位到文件`LUMEN/vendor/laravel/lumen-framework/src/Concerns/RegistersExceptionHandlers.php`中370行：

{{< image src="/assets/images/lumen-application-1.jpg" alt="lumen-application-1" position="center" >}}

{{< image src="/assets/images/lumen-application-2.jpg" alt="lumen-application-2" position="center" >}}

框架中的LUMEN/public/index.php默认是`$app->run();`是不传$request的，对应调用的方法如下：

{{< image src="/assets/images/lumen-application-3.jpg" alt="lumen-application-3" position="center" >}}

可见通过此方法获取的path，如果存在多应用的情况下，$this->routes获取是错误的。

### **解决方法如下：** ###
修改`LUMEN/public/index.php`为

```php
$request = Illuminate\Http\Request::capture();
$app->run($request);
```

---

**参考**

* [https://segmentfault.com/a/1190000002724037](https://segmentfault.com/a/1190000002724037)
* [https://segmentfault.com/a/1190000004460637](https://segmentfault.com/a/1190000004460637)
* [http://stackoverflow.com/questions/29728973/notfoundhttpexception-with-lumen](http://stackoverflow.com/questions/29728973/notfoundhttpexception-with-lumen)