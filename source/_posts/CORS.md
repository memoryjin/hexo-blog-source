---
title: 深入理解CORS
---

##### 简介

出于浏览器的同源策略限制，浏览器会限制跨域请求。这里要注意的是，所谓的限制跨域请求并不是说该请求无法传递给服务器，而是说服务器返回的正常响应**被浏览器拦截**了！随着IE10浏览器基本上已经退出历史的舞台，CORS已成为最常用的跨域解决方案。跟JSONP相比，CORS的好处主要有两个：一是“前端开发人员无感知”，前端几乎不需要任何额外的操作就能达到跨域请求服务器；二是不仅支持GET方法，还支持POST、PUT、DELETE等一系列方法。

##### 两种请求

浏览器将CORS请求分为两类：*简单请求*和*非简单请求*。只有满足以下两大条件，才属于简单请求。

1. 请求方法是以下三种方法之一：HEAD、GET、POST
2. HTTP的头信息不超过以下几种字段：Accept、Accept-Language、Content-Language、Last-Event-ID、Content-Type（只限于三个值——application/x-www-form-urlencoded、multipart/form-data、text/plain）

除此之外的都是非简单请求，浏览器对两类请求的处理方式是不一样的。

##### 简单请求

客户端发起一个跨域请求时，浏览器会在请求header信息中增加一个Origin字段：

```http
GET /cors HTTP/1.1
Connection: keep-alive
Origin: http://www.api.com
Host: www.api.com
Accept-Language: en-US,
User-Agent: Mozila/5.0...
```

上面的头信息中，Origin用来说明该请求来自哪个源(协议+域名+端口号)。

对于服务器返回的响应，浏览器会去检查头信息中是否包含Access-Control-Allow-Origin字段，如果**没有包含该字段或者Origin指定的域名未在许可范围内**，此时浏览器会拦截该响应并抛出一个错误被XMLHttpRequest的onerror回调函数捕获。如果响应头信息中有Access-Control-Allow-Origin字段且Origin指定的域名在许可范围内，浏览器将不拦截该响应，此时我们可以看到多出了几个头信息字段。

```http
Access-Control-Allow-Origin: http://www.api.com
Access-Control-Allow-Credentials: true
Access-Control-Expose-Headers: header
```

1. Access-Control-Allow-Origin

   该字段是必须的，表示接受的可允许跨域请求的域名。为*时表示接受任意域名的请求

2. Access-Control-Expose-Headers

   该字段可选。CORS请求时，XMLHttpRequest对象的getResponseHeader()方法只能拿到6个基本字段：Cache-Control、Content-Language、Content-Type、Expires、Last-Modified、Pragma。如果想拿到其他字段，就必须在Access-Control-Expose-Headers指定

3. Access-Control-Allow-Credentials

   该字段可选，具体信息见后文。

##### 非简单请求

客户端发起一个跨域请求时，若浏览器判定该请求不是简单请求，则浏览器会自动发起一个方法为OPTIONS预检请求:

```http
OPTIONS /cors HTTP/1.1
Origin: http://www.api.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: X-Custom-Header
...
```

除了Origin字段，预检OPTIONS请求会包含两个特殊字段：

1. Access-Control-Request-Method

   该字段必传，表示正式请求时的方法。

2. Access-Control-Request-Headers

   该字段非必传，表示正式请求时将增加的额外请求头信息。

若预检请求通过，在服务器返回的响应中将新增几个字段：

```http
HTTP/1.1 200 OK
Date: Mon, 01 Dec 2008 01:15:39 GMT
Server: Apache/2.0.61 (Unix)
Access-Control-Allow-Origin: http://www.api.com
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: X-Custom-Header
Access-Control-Max-Age: 1728000
...
```

1. Access-Control-Allow-Methods

   该字段必需，表示服务器接受的跨域请求方法

2. Access-Control-Allow-Headers

   该字段非必需，表示可接受的请求头字段

3. Access-Control-Max-Age

   该字段非必需，表示预检请求的有效期，单位为秒。即在该有效期范围内，相同域名对服务器发起的跨域请求可以跳过预检请求这一阶段，直接进行数据交互。

在预检请求通过后，浏览器会自动发起正式请求，该过程跟简单请求类似，在此不再赘述。

##### 跨域Cookie传递

默认情况下浏览器对跨域请求不会携带Cookie，但鉴于 Cookie 在身份验证等方面的重要性， CORS 推荐使用额外的响应头字段来允许跨域发送 Cookie

1. 客户端代码

在`open` [XMLHttpRequest](https://developer.mozilla.org/zh-CN/docs/Web/API/XMLHttpRequest)后，设置`withCredentials`为`true`即可让该跨域请求携带 Cookie（当然，前提是预检请求通过）。 注意携带的是目标页面所在域的 Cookie。

```javascript
const xhr = new XMLHttpRequest()
xhr.withCredentials = true
```

需要注意的是，如果省略`withCredentials`的设置，部分浏览器跨域请求时还是会发送cookie，所以如果要禁止跨域请求带上cookie，请记得将`withCredentials`设置为false。

2. 服务端代码

要想跨域发送cookie，服务端的`Access-Control-Allow-Origin`就不能设为*，必须制定与请求一致的域名。