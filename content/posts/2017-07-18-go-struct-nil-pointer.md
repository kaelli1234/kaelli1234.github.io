---
date: "2017-07-18"
title: Golang中的JSON处理技巧
description: go, golang, struct, nil pointer
slug: go-struct-nil-pointer
categories:
- Web
tags:
- golang
comments: true
---

又好久没有更新博客了，最近因为项目需要学习了下Go语言，才真正明白了为什么说PHP是世界上最好的语言🤣。言归正传，首先先推荐本秘籍
[build-web-application-with-golang](https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/preface.md) Github上的高星项目，作者同时也是beego框架的开发者，本项目是作者使用Go进行Web开发的经验总结，满满的干货。

所以基础的[JSON处理](https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/07.2.md)就直接参考文档吧，主要内容大概是Go中有专门的JSON包来处理JSON数据，通过定义对应的struct，tag映射字段，很方便的可以把JSON数据转成struct，或者把struct转化成JSON字符串。

其中文档中有段描述是这样的：

> tag中如果带有"omitempty"选项，那么如果该字段值为空，就不会输出到JSON串中，这其实是一个比较常见的需求，当某个字段为空时，我们不希望该字段也输出在JSON字符串中，通过在结构体的tag中定义`omitempty`就可以达到该目的。

```go
type DataAttachment struct {
    Id     string                 `json:"id,omitempty"`
    Head   DataAttachmentHead     `json:"head,omitempty"`
    Body   DataAttachmentBody     `json:"body,omitempty"`
}

type DataAttachmentHead struct {
    Text    string `json:"text,omitempty"`
    Bgcolor string `json:"bgcolor,omitempty"`
    Tcolor  string `json:"tcolor,omitempty"`
}

type DataAttachmentBody struct {
    Title   string `json:"title,omitempty"`
    Image   string `json:"image,omitempty"`
    Content string `json:"content,omitempty"`
}
```

以上DataAttachment这个结构，通过对head，body定义omitempty选项，我们预期是希望如果当head，body为空时，可以不输出这2个字段到JSON中。但是事实上却和我们预期的有所差异，我们通过`json.Marshal`得到的JSON中还是会有空的head和body。

所以我们要怎样才能实现我们的目的呢？其实很简单，**传指针**
```go
type DataAttachment struct {
    Id     string                 `json:"id,omitempty"`
    Head   *DataAttachmentHead     `json:"head,omitempty"`
    Body   *DataAttachmentBody     `json:"body,omitempty"`
}
```

---

**参考**

* [https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/07.2.md](https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/07.2.md)
* [build-web-application-with-golang](https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/preface.md)
* [https://stackoverflow.com/questions/18088294/how-to-not-marshal-an-empty-struct-into-json-with-go](https://stackoverflow.com/questions/18088294/how-to-not-marshal-an-empty-struct-into-json-with-go)