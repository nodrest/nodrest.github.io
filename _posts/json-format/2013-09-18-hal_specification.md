---
layout: post
category : 文档
title : HAL规范
tagline : "超文本应用程序语言"
tags : [hal, json, api, 格式]
---

# 一个精干的超媒体类型

* __作者:__   [Mike Kelly][1] <[mike@stateless.co](mailto:mike@stateless.co)>
* __创建:__  2011-06-13
* __更新:__  2013-09-18 (Updated)

## 概要
HAL is a simple format that gives a consistent and easy way to
hyperlink between resources in your API.

Adopting HAL will make your API explorable, and its documentation easily
disocverable from within the API itself. In short, it will make your API
easier to work with and therefore more attractive to client developers.

APIs that adopt HAL can be easily served and consumed using open source
libraries available for most major programming languages. It's also
simple enough that you can just deal with it as you would any other
JSON.

## 关于作者
Mike Kelly is a software engineer from the UK. He runs an [API
consultancy][1] helping companies design and build beautiful APIs that
developers love.

## 快速链接
* [A demo API using HAL called HAL Talk][12]
* [A list of libraries for working with HAL (Obj-C, Ruby, JS, PHP, C#,
  etc.)][14]
* [A list of public hypermedia APIs using HAL][15]
* [Discussion group (questions, feedback, etc)][2]

## 概述

HAL提供了在任何JSON或XML表示的超链接一组约定。

**一个HAL文件的其余部分只是普通的老JSON或XML.**

Instead of using ad-hoc structures, or spending valuable time designing
your own format; you can adopt HAL's conventions and focus on building
and documenting the data and transitions that make up your API.

HAL is a little bit like HTML for machines, in that it is generic and
designed to drive many different types of application via hyperlinks.
The difference is that HTML has features for helping 'human actors' move
through a web application to achieve their goals, whereas HAL is
intended for helping 'automated actors' move through a web API to
achieve their goals.

说了这么多, **HAL其实很人性化了**. Its
conventions make the documentation for an API discoverable from the API
messages themselves.  This makes it possible for developers to jump
straight into a HAL-based API and explore its capabilities, without the
cognitive overhead of having to map some out-of-band documentation onto
their journey.

## 实例

下面的例子是如何使用HAL+JSON表示订单的集合。 如下:

* 被表示主资源的URI（'/orders'）用self链接表示
* 'next'连接订单下一页
* 模板链接调用'ea:find',通过id查找订单
* 多个'ea:admin'链接对象被包含在一个数组里
* 订单集合的两个属性; 'currentlyProcessing'和'shippedToday'
* 嵌入式订单资源使用用自己的链接和属性
* 命名为'ea'紧凑的URI(curie),扩展了链接的名称为它们的文档的URL

### application/hal+json
```javascript
{
    "_links": {
        "self": { "href": "/orders" },
        "curies": [{ "name": "ea", "href": "http://example.com/docs/rels/{rel}", "templated": true }],
        "next": { "href": "/orders?page=2" },
        "ea:find": {
            "href": "/orders{?id}",
            "templated": true
        },
        "ea:admin": [{
            "href": "/admins/2",
            "title": "Fred"
        }, {
            "href": "/admins/5",
            "title": "Kate"
        }]
    },
    "currentlyProcessing": 14,
    "shippedToday": 20,
    "_embedded": {
        "ea:order": [{
            "_links": {
                "self": { "href": "/orders/123" },
                "ea:basket": { "href": "/baskets/98712" },
                "ea:customer": { "href": "/customers/7809" }
            },
            "total": 30.00,
            "currency": "USD",
            "status": "shipped"
        }, {
            "_links": {
                "self": { "href": "/orders/124" },
                "ea:basket": { "href": "/baskets/97213" },
                "ea:customer": { "href": "/customers/12369" }
            },
            "total": 20.00,
            "currency": "USD",
            "status": "processing"
        }]
    }
}
```

## HAL模型

该HAL约定围绕表示两个简单的概念: _Resources_ 和 _Links_.

### 资源
资源拥有:

* 链接 (到URIs)
* 内嵌资源 (例. 包含在其中的其他资源)
* 状态 (您的沼泽是基于JSON或XML数据)

### 链接
链接游泳:

* 目标 (一个URI)
* 一个关系别名. 'rel' (链接名字)
* 其他一些可选属性，以帮助折旧，内容协商等。

下面的图像大致说明了HAL表示法的构造。

![HAL信息模型][4]

## 在API中如何使用HAL

HAL设计用于构建API，在客户端中通过以下链接浏览周围的资源。

Links are identfied by link realtions. Link realtions are the lifeblood
of a hypermedia API: they are how you tell client developers about
what resources are available and how they can be interacted with, and
they are how the code they write will select which link to traverse.

Link relations are not just an identifying string in HAL, though. They
are actually URLs, which developers can follow in order to read the
documentation for a given link. This is what is known as
"可发现". The idea is that a developer can enter into your API,
read through documentation for the available links, and then
跟踪它们的鼻子 through the API.

HAL鼓励使用链接关系来:

*   在表示中识别链接和嵌入的资源
*   推断目标资源的预期结构和含义
*   发信号通知哪些请求及声明，可以提交到目标资源

## 如何服务HAL

HAL里包含着JSON和XML的变种媒体类型，whos的名称分别是`application/hal+json`和`application/hal+xml`。

当在HTTP上供应HAL时，响应的`Content-Type`应该包含相关媒体类型名称。

## HAL文档架构

### 最小有效文档
一个HAL文件必须至少包含一个空的资源。

一个空的JSON对象：

```javascript
{}
```

### 资源
在大多数情况下，资源应该有一个自己的URI

通过'self'链接表示：

```javascript
{
    "_links": {
        "self": { "href": "/example_resource" }
    }
}
```

### 链接

链接必须直接包含在一个资源：

链接被表示为包含在`_links`哈希里JSON对象，它必须是一个资源对象的直接属性的：

```javascript
{
    "_links": {
        "next": { "href": "/page=2" }
    }
}
```

#### 链接关系
链接有一个关系 (别名. 'rel'). 这声明语义 - 意义 - 一个特定的链接。

链接rels是区分一种资源的的链接主要途径。

它基本商是'_links'哈希的一个键值，关联链接的意义('rel') 连接对象包含如实际的'href'值的数据：

```javascript
{
    "_links": {
        "next": { "href": "/page=2" }
    }
}
```

#### API可发现性

链接rels的应该是它揭示给定链接的文档，使他们“发现”的网址。网址一般都相当长，有点讨厌用作键。为了解决这个问题, HAL提供了"CURIEs" which are basically named tokens that you can define in the document and use to express link relation URIs in a friendlier, more compact fashion i.e.  `ex:widget` instead of `http://example.com/rels/widget`. 详情可在再往下CURIEs一节找到。

### 使用相同的关系的表示多个链接
一个资源可以有多个链接，共享相同的链接关系。

对于可能有多个链接的链接关系中，我们使用的链接数组。

```javascript
{
    "_links": {
      "items": [{
          "href": "/first_item"
      },{
          "href": "/second_item"
      }]
    }
}
```

**注意:** 如果你不确定该链接是否应为单数，假设它会多个。 If you pick singular and find you need to change it,you will need to create a new link relation or face breaking existing clients.

### CURIEs

"CURIE"s 帮助提供链接到资源文件。

HAL为您提供了一个保留的链接关系“居里”，你可以用它来暗示资源文件的位置。

```javascript
"_links": {
  "curies": [
    {
      "name": "doc",
      "href": "http://haltalk.herokuapp.com/docs/{rel}",
      "templated": true
    }
  ],

  "doc:latest-posts": {
    "href": "/posts/latest"
  }
}
```

There can be multiple links in the 'curies' section. They come with a 'name' and a templated 'href' which must contain the `{rel}` placeholder.

Links in turn can then prefix their 'rel' with a CURIE name. Associating the `latest-posts` link with the `doc` documentation CURIE results in a link 'rel' set to `doc:latest-posts`.

To retrieve documentation about the `latest-posts` resource, the client will expand the associated CURIE link with the actual link's 'rel'. This would result in a URL `http://haltalk.herokuapp.com/docs/latest-posts` which is expected to return documentation about this resource.


## 待续...
This relatively informal specification of HAL is incomplete and still in progress. For now, if you would like to have a full understanding please read the [formal specification][13].

## RFC
The JSON variant of HAL (application/hal+json) has now been published as an internet draft: [draft-kelly-json-hal][13].

## 致谢

* Darrel Miller
* Mike Amundsen
* Mark Derricutt
* Herman Radtke
* Will Hartung
* Steve Klabnik
* everyone on hal-discuss

Thanks for the help :)

## 备注/ TODO

 [1]: http://stateless.co/
 [2]: http://groups.google.com/group/hal-discuss
 [3]: http://blog.stateless.co/post/13296666138/json-linking-with-hal
 [4]: http://stateless.co/info-model.png
 [5]: http://tools.ietf.org/html/rfc2119
 [6]: http://tools.ietf.org/html/rfc5988#section-5.1
 [7]: http://tools.ietf.org/html/rfc5988
 [8]: http://tools.ietf.org/html/rfc5988#section-5.3
 [9]: http://tools.ietf.org/html/rfc5988#section-5.4
 [10]: http://www.w3.org/TR/curie/
 [11]: http://tools.ietf.org/html/rfc6570
 [12]: http://haltalk.herokuapp.com/
 [13]: http://tools.ietf.org/html/draft-kelly-json-hal
 [14]: https://github.com/mikekelly/hal_specification/wiki/Libraries
 [15]: https://github.com/mikekelly/hal_specification/wiki/APIs
