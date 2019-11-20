---
date: "2016-08-08"
title: PHP使用CURL访问HTTPS
description: php, curl, https
slug: php-curl-https
categories:
- Web
tags:
- php
comments: true
---

最近新上线的项目准备升级服务支持https访问，记录下使用curl方法访问https时遇到的问题。

直接上代码:

```php
function post($url, $param)
{
    $ssl = substr($url, 0, 8) == "https://" ? true : false;
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    if ($ssl) {
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false); // 信任任何证书
        curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, false); // 检查证书中是否设置域名

        // $cacert = getcwd() . '/cacert.pem'; //CA根证书
        // curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, true);   // 只信任CA颁布的证书
        // curl_setopt($ch, CURLOPT_CAINFO, $cacert); // CA根证书（用来验证的网站证书是否是CA颁布）
        // curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 2); // 检查证书中是否设置域名，并且是否与提供的主机名匹配
    }
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POST, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, $param);

    $return = curl_exec($ch);

    curl_close($ch);
    return $return;
}
```

简单来说访问https时，只要指定`CURLOPT_SSL_VERIFYPEER`和`CURLOPT_SSL_VERIFYHOST`为`false`就能支持访问了。当然也可以下载对应的根证书，然后在请求中带上访问也可以。

---

**参考**

* [http://php.net/manual/zh/function.curl-setopt.php](http://php.net/manual/zh/function.curl-setopt.php)
* [http://blog.csdn.net/linvo/article/details/8816079](http://blog.csdn.net/linvo/article/details/8816079)
* [http://unitstep.net/blog/2009/05/05/using-curl-in-php-to-access-https-ssltls-protected-sites](http://unitstep.net/blog/2009/05/05/using-curl-in-php-to-access-https-ssltls-protected-sites)