---
layout: post
category : 文档
title : Restify OAuth2
tagline : "安装文档"
tags : [nodejs, restful, OAuth2]
---

## Restify的OAuth2终端

本包为[Restify][]框架提供一个*非常简单*的OAuth 2.0 终端. 特别是, 它只实现了[客户端凭证][cc]和[资源所有者密码认证][ropc] 流量.

## 你会得到what?

如果你提供Restify–OAuth2,并使用合适的钩子, 它将:

* 设置一个[令牌这终端][token endpoint], 将返回[访问令牌响应][token-endpoint-success]或者[正确格式错误回应][token-endpoint-error].
* 对于所有其他资源, 当一个访问令牌是[通过`Authorization`头发送][send-token], 它会验证它:
  * 如果令牌验证失败, 它将发送[恰当的400或401错误响应][token-usage-error], 并带有[`WWW-Authenticate`][www-authenticate]头和[`Link`][web-linking] [`rel="oauth2-token"`][oauth2-token-rel]
    头指向令牌终端.
  * 否则, 它将设置要么`req.clientId`要么`req.username` (依赖下面你所用的)为通过查找访问令牌确定的值.
* 如果没有访问令牌发送, 简单设置`req.clientId`/`req.username`为`null`:
  * 您可以检查这个只要有你想要保护的资源。
  * 如果用户试图访问受保护的资源，你可以使用Restify–OAuth2的`res.sendUnauthorized()`来发送合适401错误通过有帮助的 `WWW-Authenticate`和`Link`头.

## 使用和配置

使用Restify–OAuth2, 你需要传递给你的服务插件一些选项, 包括下面提及的钩子.Restify–OAuth2同时依赖内建的`authorizationParser` 和 `bodyParser` 插件, 后者 的`mapParams`参数设为`false`. 简言之, 如下:

{% highlight js linenos %}
var restify = require("restify");
var restifyOAuth2 = require("restify-oauth2");

var server = restify.createServer({ name: "My cool server", version: "1.0.0" });
server.use(restify.authorizationParser());
server.use(restify.bodyParser({ mapParams: false }));

restifyOAuth2.cc(server, options);
// or
restifyOAuth2.ropc(server, options);
{% endhighlight %}

不幸的是, Restify–OAuth2不能成为一个简单的Restify插件. 它需要为令牌终端安装一个路由,而插件只是在每次请求时运行，不修改服务器的路由表.

## 钩子选项

传递给Restify–OAuth2的参数严重依赖你选择的两个流中的一个。一些选项在所有流中通用, 但是`options.hooks`哈希将取决于流变化. 一旦您提供相应的钩子, 你会免费得到一个OAuth 2实现.

### CC钩子(客户端认证)

非常简单的OAuth 2流下面的想法是你的API客户端通过客户端IDs和客户段谜钥认证自身，并且如果这些值认证了，你赋予他们一个访问令牌,将来在请求中可以使用。比在每次请求时简单的基于访问认证头请求的好处是现在你可一设置这些令牌过期或者在它们落在了坏人的手中撤销他们如果。

安装Restify–OAuth2的客户端证书流到你的架构, 你需要提供它带以下钩子在`options.hooks`哈希里. 你可以在演示演示应用程序里查看一些[CC钩子实例][example CC hooks].

#### `grantClientToken(clientId, clientSecret, cb)`

检测API客户端是否认证可用，并有正确的密钥。如果这样它应该为那个客户端回调一个新的令牌，或者回调'false'如果证书是错误的。如果当验证凭据的时候发生内部服务错误它也可以回调一个错误。

#### `authenticateToken(token, cb)`

检测令牌是否有效，通过上一步`grantClientToken`赋予的。如果那样它应该为那个令牌回调带客户ID，或者`false`如果令牌失效。当查看令牌的时候如果发生服务器内部错误它也可以回调错误。

### ropc钩子(资源所有者密码认证)

The idea behind this OAuth 2 flow是that your API clients will prompt the user for their username and password, and send those to your API in exchange for an access token. This has some advantages over simply sending the user's credentials to the server directly. For example, it obviates the need for the client to store the credentials, and allows expiration and revocation of tokens. However, it does imply that you trust your API clients, since they will have at least one-time access to the user's credentials.

To install Restify–OAuth2's resource owner password credentials flow into your infrastructure, you will need to provide it with the following hooks in the `options.hooks` hash. 你可子在演示应用中看到[ROPC钩子实例][example ROPC hooks].

#### `validateClient(clientId, clientSecret, cb)`

Checks that the API client是authorized to use your API, and has the correct secret. It should call back with `true` or `false` depending on the result of the check. It can also call back with an error if there was some internal server error while doing the check.

#### `grantUserToken(username, password, cb)`

Checks that the API client是authenticating on behalf of a real user with correct credentials. It should call back with a new token for that user if so, or `false` if the credentials are incorrect. It can also call back with an error if there was some internal server error while validating the credentials.

#### `authenticateToken(token, cb)`

检测令牌是否有效, i.e. that it was granted in the past by `grantUserToken`. It should call back with the
username for that token if so, or `false` if the token是invalid. It can also call back with an error if there was some internal server error while looking up the token.

## 其他选项

`hooks`哈系是唯一必须选项, 但是以下调整选项可用:

* `tokenEndpoint`: 创建令牌终端的地址. 默认是`"/token"`.
* `wwwAuthenticateRealm`: 在`WWW-Authenticate`头里的"Realm"盘问值. 默认是`"Who goes there?"`.
* `tokenExpirationTime`: the value returned for the `expires_in` component of the response from the token endpoint. Note that this是*only* the value reported; you are responsible for keeping track of token expiration yourself and calling back with `false` from `authenticateToken` when the token expires. 默认无限`Infinity`.

## 看起来像什么？

好吧，让我们尝试一些更具体一点. If you check out the [示例服务器][example servers] used in the integration tests,you'll see our setup. 在这里，我们将引导您完成更复杂的资源所有者密码认证例子,但客户端认证思路的例子非常相似。.

### /

最初的资源，在人们进入服务器。

* 如果在`认证`头中提供有效令牌, `req.username`是truthy, 并且应用将响应带链接到`/public`和`/secret`.
* 如果没有提供令牌, 应用将响应带链接到`/public`和`/secret`.
* 如果提供一个无效令牌,, Restify–OAuth2在请求获得应用之前被截取，并发送一个合适的400或者401的错误.

### /token

令牌终端, 完全由Restify-OAuth2管理。 为给定的客端ID/密钥/用户名/密码，组合生成令牌.

客户端验证和令牌生成逻辑是应用提供的, 但没有必要的OAuth2一致性仪式, 错误控制, 等等是目前在应用程序代码: Restify–OAuth2接受了这一切的护理.

### /public

任何人能访问的全局资源.

* 如果在认证头中提供有效令牌, `req.username`包含用户名, 应用使用它发送个性化数据.
* 如果没有提供令牌, `req.username`是`null`. 应用仍然发送响应, 只是没有个性化数据.
* 如果提供一个无效令牌, Restify–OAuth2在请求获得应用之前被截取，并发送一个合适的400或者401的错误.

### /secret

认证用户访问的加密资源.

* 如果在认证头中提供有效令牌, `req.username`是truthy, 应用将发送加密数据.
* 如果没有提供令牌, `req.username`是`null`, 应用使用 `res.sendUnauthorized()`发送友好的带`WWW-Authenticate`和`Link`头401错误.
* 如果提供一个无效令牌, Restify–OAuth2在请求获得应用之前被截取，并发送一个合适的400或者401的错误.

[Restify]: http://mcavage.github.com/node-restify/
[cc]: http://tools.ietf.org/html/rfc6749#section-1.3.4
[ropc]: http://tools.ietf.org/html/rfc6749#section-1.3.3
[token endpoint]: http://tools.ietf.org/html/rfc6749#section-3.2
[token-endpoint-success]: http://tools.ietf.org/html/rfc6749#section-5.1
[token-endpoint-error]: http://tools.ietf.org/html/rfc6749#section-5.2
[send-token]: http://tools.ietf.org/html/rfc6750#section-2.1
[token-usage-error]: http://tools.ietf.org/html/rfc6750#section-3.1
[oauth2-token-rel]: http://tools.ietf.org/html/draft-wmills-oauth-lrdd-07#section-3.2
[web-linking]: http://tools.ietf.org/html/rfc5988
[www-authenticate]: http://tools.ietf.org/html/rfc2617#section-3.2.1
[example ROPC hooks]: https://github.com/nodrest/restify-oauth2/blob/master/examples/ropc/hooks.js
[example CC hooks]: https://github.com/nodrest/restify-oauth2/blob/master/examples/cc/hooks.js
[example servers]: https://github.com/nodrest/restify-oauth2/tree/master/examples

