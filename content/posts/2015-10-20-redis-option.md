---
date: "2015-10-20"
title: Redis::OPT_SERIALIZER
description: redis, redis-php, php-redis, redis setOption, OPT_SERIALIZER, SERIALIZER_IGBINARY
slug: redis-option
categories:
- Web
tags:
- redis
comments: true
---

### **标记一个最近使用redis遇到的坑**


#### **背景**：
php-redis版本: `2.2.1`
新老项目使用的封装PHP Redis类不一样(主要是以下选项)：

```php
$redis->setOption(Redis::OPT_SERIALIZER, Redis::SERIALIZER_IGBINARY);
```

导致使用设置过该option的封装的Redis类获取由未设置该选项的Redis类生成的key时程序会down掉，反之获取的key值为乱码。

---

#### **解决方案**:
暂时没想到更好的方案，目前的解决方式是统一redis中的键值，将老项目中用的redis key全delete掉 统一使用新的Redis类生成