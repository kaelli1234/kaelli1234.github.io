---
layout: post
title: php5.6 curl 上传文件失败
description: php, php5.6, curl, CURLFile, 上传文件
categories: [Web]
tags: [PHP]
shortinfo: 最近新项目中使用较高版本的php（php5.6），标记一下使用curl上传文件时遇到的坑
---

#### php5.6 CURLFile

新项目中要完成一个上传文件的接口，要写个自测接口时遇到的大坑。。。

之前php5.4版本时curl中直接通过@符号+文件路径，则curl自动会把文件解析上传，后端通过$_FILES变量拿到，但在写自测脚本时死活没反应。。。纠结了半天才在 [stackoverflow](http://stackoverflow.com/questions/25934128/curl-file-uploads-not-working-anymore-after-upgrade-from-php-5-5-to-5-6) 找到答案。

原来新版php中已经不支持@这样的写法，而且通过CURLFile类来完成模拟文件上传。代码如下:
{% highlight php linenos %}
if ((version_compare(PHP_VERSION, '5.5') >= 0)) {
    $aPost['file'] = new CURLFile($localFile);
    curl_setopt($ch, CURLOPT_SAFE_UPLOAD, true);
} else {
    $aPost['file'] = "@".$localFile;
}

$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, $apiurl);
curl_setopt($ch, CURLOPT_TIMEOUT, 120);
curl_setopt($ch, CURLOPT_BUFFERSIZE, 128);
curl_setopt($ch, CURLOPT_POSTFIELDS, $aPost);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
$sResponse = curl_exec ($ch);
{% endhighlight %}