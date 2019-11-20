---
date: "2017-03-01"
title: php通过定义JSON Schema来校验对应JSON的格式数据
description: php, json, json-schema, json-schema-validate
slug: json-schema-validate
categories:
- Web
tags:
- php
- json
comments: true
---

最近项目中遇到的需求，对于某JSON类型的字段，要做验证的格式校验，并且标出错误的地方。

本着不要重复造轮子的原则，在github上找到了这个项目

> 通过定义固定格式的JSON Schema来验证对应格式的数据。<br>[https://github.com/justinrainbow/json-schema](https://github.com/justinrainbow/json-schema)

具体的使用方法就不多说啦，通过composer安装，参照示例就好。

主要说一下JSON Schema文件的定义问题，👇结合一个比较复杂的例子来说一下Schema各个参数的作用

```json
{
    "title": "Product set",
    "type": "array",    //原数据格式为数组
    "minItems": 1,  //最少1个元素
    "maxItems": 10, //最多10个元素
    "uniqueItems": true,    //数据重复判断
    "items": {
        "title": "Product",
        "type": "object",   //数组元素的值为对象
        "properties": {
            "id": {
                "description": "The unique identifier for a product",   //id字段描述
                "type": "string",   //id的类型为string
                "pattern": "^(\\([0-9]{3}\\))?[0-9]{3}-[0-9]{4}$"   //验证id是否符合该正则表达式的规则
            },
            "name": {
                "type": "string",    //name的类型为string
                "maxLength": 10 //最大长度为10
            },
            "url": {
                "type": "string",   //url的类型为string
                "format": "uri" //并且符合URI规则
            },
            "price": {
                "type": "number",   //price的类型为number
                "minimum": 0,   //最小值为0
                "exclusiveMinimum": true    //该字段为true时验证price>0,false时验证price>=0
            },
            "tags": {
                "type": "string",   //tags的类型为string
                "enum": [   //并且值为enum中之一
                    "food",
                    "fruit",
                    "juice"
                ]
            }
        },
        "additionalProperties": false, //不允许除了items.properties定义以外的字段
        "required": ["id", "name", "price"] //id,name,price字段必填
    }
}
```

还有更详细的JSON Schema文档说明见以下参考链接

---

**参考**

* [https://github.com/justinrainbow/json-schema](https://github.com/justinrainbow/json-schema)
* [http://json-schema.org/](http://json-schema.org/)]
* [https://spacetelescope.github.io/understanding-json-schema/index.html](https://spacetelescope.github.io/understanding-json-schema/index.html)]