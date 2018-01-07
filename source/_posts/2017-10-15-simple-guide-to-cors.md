---
title: A Simple Introduce To CORS
toc: true
date: 2017-11-02 21:14:14
tags: 
- Cross Origin
categories: 
- Web
---

When we need to combine the Front-End & Back-End testing, the question that the browsers' AJAX calls can't get the resources residing outside the current origin comes. 

That's because the security feature inside browsers which is called the **same-origin policy**. As the existence of this policy, any asynchronous calls to another server will be prohibited by browser.

So how to solve this problem? 

Here comes CORS.

<!-- more -->

<br/>

## What's CORS?

CORS(Cross-Origin Resource Sharing) is a W3C standard for enabling cross-domain requests from web browsers to servers and web APIs that opt in to handle them. 

Similar techniques, such as JSONP(JSON With Padding), can solve this delemma. But CORS has its advantages. 

Compared with JSONP, CORS supports all types of HTTP Requst (like GET, POST, PUT, etc) while JSONP only supports GET. And also CORS is safer than JSONP. JSONP also has its adventage -- it can be supported by old browsers (like IE 8 or the previous versions). But predictably, JSONP will be replaced by CORS gradually. 

<br/>

## Two types of CORS requests

Browsers divide CORS requests into two categories: simple request and complex request.

As long as the following two conditions at the same time, it is a simple request.

> (1) The request method is one of the following three methods:
>
> - HEAD
> - GET
> - POST
>
> (2) HTTP header information does not exceed the following fields:
>
> - Accept
> - Accept-Language
> - Content-Language
> - Last-Event-ID
> - Content-Type: three limited value `application/x-www-form-urlencoded`, `multipart/form-data`,`text/plain`

Those who do not meet the above two conditions at the same time, it is a complex request.

The browser's handling of these two requests is not the same.

<br/>

## Simple request

CORS needs to be supported by both browsers and servers. 

For simple request, simply what the browser end needs to do is to add a header `Origin` for the HTTP request and if this request is allowed by server end, what the server end needs to do is to add headers `Access-Control-*` to this response.

Here is an example of a CORS request and its corresponding response.

**Request:**

 ```http
 GET /cors HTTP/1.1
 Origin: http://api.mike.com
 Host: api.jack.com
 Accept-Language: en-US
 Connection: keep-alive
 User-Agent: Mozilla/5.0...
 ```

As the server receives a reqeust with the header `Origin`, the server decides whether to response it judged by the `Origin` information. 

**Response**

```http
Access-Control-Allow-Origin: http://api.mike.com
Access-Control-Allow-Credentials: true
Access-Control-Expose-Headers: FooBar
Content-Type: text/html; charset=utf-8
```

If  the `Origin` is in the accept list of the server, the server returns a few more header information fields (see above).

If the `Origin`specified source is out of range, the server will return a normal HTTP response. The browser found that the header of the response did not contain `Access-Control-Allow-Origin`fields, you know that something went wrong, throwing an error and being caught by `XMLHttpRequest`the `onerror`callback. Note that this error can not be identified by the status code because the HTTP response status code may be 200.

**(1) Access-Control-Allow-Origin**

This field is required. Its value is either the value of the `Origin`field on request or it's a request `*`that accepts any domain name.

**(2) Access-Control-Allow-Credentials**

This field is optional. Its value is a Boolean value that indicates whether cookies are allowed. Cookies are not included in CORS requests by default. Set to `true`, that means the server is explicitly licensed, cookies can be included in the request, and sent to the server. This value can only be set to `true`, if the server does not send a cookie browser, delete the field can be.

If you want to use send cookie to server, except this header, you also need to open the `withCredentials`properties in the AJAX request .

```js
var xhr = new XMLHttpRequest();
xhr.withCredentials = true;
```

Otherwise, the browser will not send even if the server agrees to send cookies. Or, the server requests to set the cookie, the browser will not handle it.

However, if you omit `withCredentials`settings, some browsers will still send cookies together. At this time, you can explicitly shut down `withCredentials`.

```js
xhr.withCredentials = false;
```

Note that, if you want to send a cookie, `Access-Control-Allow-Origin`it can not be set as an asterisk, you must specify a clear domain name consistent with the request page. At the same time, cookies still follow the same-origin policy. Only cookies set using server domain names will be uploaded. Cookies from other domains will not be uploaded, and `document.cookie`cookies under the server domain name will not be read in the original (cross-origin) web page code .

**(3) Access-Control-Expose-Headers**

This field is optional. When the request CORS, `XMLHttpRequest`an object `getResponseHeader()`method can only get six basic `Cache-Control`fields: `Content-Language`, `Content-Type`, `Expires`, `Last-Modified`, `Pragma`, . If you want to get other fields, it must be `Access-Control-Expose-Headers`specified inside. The example above specifies that the value of the field `getResponseHeader('FooBar')`can be returned `FooBar`.

<br/>

## Complex request

Complex simple requests are those that have special requirements for the server, such as the request method is `PUT`or `DELETE`, or `Content-Type`the type of the field is `application/json`.

Before complex request, there is an HTTP query request, called a "preflight," prior to formal communication.

The browser first asks the server if the domain name of the current page is on the server's permission list and which HTTP verbs and header information fields are available. Only get a positive answer, the browser will issue a formal `XMLHttpRequest`request, otherwise an error.

Here's a browser JavaScript.

```js
var url = 'http://api.jack.com/cors';
var xhr = new XMLHttpRequest();
xhr.open('PUT', url, true);
xhr.setRequestHeader('X-Custom-Header', 'value');
xhr.send();
```

In the code above, the HTTP request method is `PUT`, and sends a custom header `X-Custom-Header`.

The browser found that this is a complex request, it automatically issues a "preflight" request, asking the server to confirm that such a request can be made. Below is the HTTP header information for this "Preflight" request.

**Preflight Request**

```http
OPTIONS /cors HTTP/1.1
Origin: http://api.mike.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: X-Custom-Header
Host: api.jack.com
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
```

<br/>

The request method for the "Preflight" request is `OPTIONS`to indicate that the request is for questioning. Inside the header, the key field `Origin`indicates which source the request came from.

In addition to the `Origin`fields, the header information for the Preflight request includes two special fields.

**(1) Access-Control-Request-Method**

This field is required to list which HTTP methods will be used by the browser's CORS request, in the above example `PUT`.

**(2) Access-Control-Request-Headers**

This field is a comma delimited string that specifies the header information field that the browser CORS request will send additionally, in the above example `X-Custom-Header`.

<br/>

After the server receives the "Preflight" request, after checking `Origin`, `Access-Control-Request-Method`and `Access-Control-Request-Headers`after confirming that cross-origin requests are allowed, it can respond.

**Preflight Response**

```http
HTTP/1.1 200 OK
Date: Mon, 01 Dec 2008 01:15:39 GMT
Server: Apache/2.0.61 (Unix)
Access-Control-Allow-Origin: http://api.bob.com
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: X-Custom-Header
Content-Type: text/html; charset=utf-8
Content-Encoding: gzip
Content-Length: 0
Keep-Alive: timeout=2, max=100
Connection: Keep-Alive
Content-Type: text/plain
```

In the above HTTP response, the key is the `Access-Control-Allow-Origin`field, which means that the `http://api.bob.com`data can be requested. This field can also be set to an asterisk, indicating that you agree to any cross-origin request.

```http
Access-Control-Allow-Origin: *
```

If the browser negates the "Preflight" request, it returns a normal HTTP response, but without any CORS-related header fields. At this point, the browser will assume that the server does not agree with the preflight request, thus triggering an error that is caught by the `XMLHttpRequest`object's `onerror`callback. The console prints the following error message.

```http
XMLHttpRequest cannot load http://api.alice.com.
Origin http://api.bob.com is not allowed by Access-Control-Allow-Origin.
```

The other CORS related fields the server responds to are as follows.

```http
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: X-Custom-Header
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 1728000
```

**(1) Access-Control-Allow-Methods**

This field is required, and its value is a comma-delimited string that indicates the method of all cross-domain requests supported by the server. Note that all supported methods are returned, not just the one requested by the browser. This is to avoid multiple "preflight" requests.

**(2) Access-Control-Allow-Headers**

Fields are required if the browser request includes `Access-Control-Request-Headers`fields `Access-Control-Allow-Headers`. It is also a comma-delimited string indicating that all header fields supported by the server are not limited to the fields requested by the browser in Preflight.

**(3) Access-Control-Allow-Credentials**

This field has the same meaning as a simple request.

**(4) Access-Control-Max-Age**

This field is optional and is used to specify the validity period of this preflight request in seconds. In the above result, the validity period is 20 days (1728000 seconds), that is, the cache is allowed to respond to 1728000 seconds (that is, 20 days), during which time it is not necessary to issue another preflight request.

<br/>

Once the server has passed the "preflight" request, every subsequent normal CORS request from the browser will have the same `Origin`header information field as the simple request . Server response, there will be a `Access-Control-Allow-Origin`header information field.

Below is the browser's normal CORS request after the "Preflight" request.

**Normal Request**

```
PUT /cors HTTP/1.1
Origin: http://api.bob.com
Host: api.alice.com
X-Custom-Header: value
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
```

The header information above `Origin`is automatically added by the browser.

The following is a normal server response.

**Normal Response**

```
Access-Control-Allow-Origin: http://api.bob.com
Content-Type: text/html; charset=utf-8
```

The above header information, the `Access-Control-Allow-Origin`field is always included in each response.

<br/>

Here is the whole process of CORS Preflighting:

![CORS Preflighting](/images/CORS/cors.png)

<br/>

## Add CORS support to your server

For simple CORS requests, the server only needs to add the following header to its response:

``` http
Access-Control-Allow-Origin: *
```

If you'd like to learn more about implementing CORS for a specific platform, follow one of the links below:

- [Apache](https://enable-cors.org/server_apache.html)
- [App Engine](https://enable-cors.org/server_appengine.html)
- [ASP.NET](https://enable-cors.org/server_aspnet.html)
- [AWS API Gateway](https://enable-cors.org/server_awsapigateway.html)
- [Caddy](https://enable-cors.org/server_caddy.html)
- [CGI Scripts](https://enable-cors.org/server_cgi.html)
- [ExpressJS](https://enable-cors.org/server_expressjs.html)
- [IIS6](https://enable-cors.org/server_iis6.html)
- [IIS7](https://enable-cors.org/server_iis7.html)
- [Spring Boot Applications in Kotlin](https://enable-cors.org/server_spring-boot_kotlin.html)
- [Meteor](https://enable-cors.org/server_meteor.html)
- [nginx](https://enable-cors.org/server_nginx.html)
- [Perl](https://enable-cors.org/server_perl.html)
- [PHP](https://enable-cors.org/server_php.html)
- [ColdFusion](https://enable-cors.org/server_coldfusion.html)
- [Tomcat](https://enable-cors.org/server_tomcat.html)
- [Virtuoso](https://enable-cors.org/server_virtuoso.html)
- [WCF](https://enable-cors.org/server_wcf.html)

<br/>
