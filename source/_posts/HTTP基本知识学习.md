---
title: HTTP基本知识学习
date: 2023-04-23 10:06:36
tags:
- HTTP
- 计算机网络
categories:
- 计算机网络
index_img: https://whitestorm0316.github.io/picx-images-hosting/image.54xzul0haf.jpg
---
# HTTP基础

## 定义

**HTTP** **是⼀个在计算机世界⾥专⻔在「两点」之间「传输」⽂字、图⽚、⾳频、视频等「超⽂本」数据的「约定和**

**规范」**

## 状态码

### 1XX

- 类状态码属于**提示信息**，是协议处理中的⼀种中间状态，实际⽤到的⽐较少

### 2XX

> 2xx 类状态码表示服务器**成功**处理了客户端的请求，也是我们最愿意看到的状态。

#### 200 OK

- 是最常⻅的成功状态码，表示⼀切正常。如果是⾮ HEAD 请求，服务器返回的响应头都会有 body数据。

#### 204 No Content

- 也是常⻅的成功状态码，与 200 OK 基本相同，但响应头没有 body 数据

#### 206 Partial Content

- 是应⽤于 HTTP 分块下载或断点续传，表示响应返回的 body 数据并不是资源的全部，⽽是其中的⼀部分，也是服务器处理成功的状态。

### 3XX

> 3xx 类状态码表示客户端请求的资源发送了变动，需要客户端⽤新的 URL ᯿新发送请求获取资源，也就是**重定向**。

#### 301 **Moved Permanently**

- 表示永久重定向，说明请求的资源已经不存在了，需改⽤新的 URL 再次访问

#### **302 Found**

- 表示临时重定向，说明请求的资源还在，但暂时需要⽤另⼀个 URL 来访问。

  `301 和 302 都会在响应头⾥使⽤字段 Location ，指明后续要跳转的 URL，浏览器会⾃动重定向新的 URL。`

#### **304 Not Modified**

- 不具有跳转的含义，表示资源未修改，᯿定向已存在的缓冲⽂件，也称缓存᯿定向，⽤于缓·存控制。

### 4XX

> 4xx 类状态码表示客户端发送的**报⽂有误**，服务器⽆法处理，也就是错误码的含义。

#### 400 Bad Request

- 表示客户端请求的报文有错误，但只是个笼统的错误

#### 403Forbdden

- 表示服务器禁止访问资源，但不是客户端的请求出错

#### 404 Not Found

- 表示请求的资源在服务器不存在或未找到

### 4XX

### 5XX

> 5xx 类状态码表示客户端请求报⽂正确，但是**服务器处理时内部发⽣了错误**，属于服务器端的错误码。

#### 500 Internal Server Error

- 表示服务器内部出错，跟400一样也是个笼统的错误

#### 501 Not Implemented

- 表示客户端请求的功能不支持，有”即将开业，敬请期待“的意思

#### 502 Bad Getaway

- 通常是服务器作为网关火代理时返回的错误码，表示服务器自身工作正常， 访问后端服务器发生了错误

#### 503 Service Unavailable

- 表示服务器当前很忙，暂时无法响应服务器

## 头部常见字段

### Host

- 客户端发送请求时，⽤来指定服务器的域名。
- 作用：可以将请求发往「同⼀台」服务器上的不同⽹站。

### Content-Length

- 服务器在返回数据时，会有 Content-Length 字段，表明本次回应的数据⻓度,后⾯的字节就属于下⼀个回应了。

### Connection

- 字段最常⽤于客户端要求服务器使⽤ TCP 持久连接，以便其他请求复⽤
- HTTP/1.1 版本的默认连接都是持久连接，但为了兼容⽼版本的 HTTP，需要指定 Connection ⾸部字段的值为

  `Keep-Alive `

### Accept

- 字段用于客户端请求时，表明⾃⼰可以接受哪些数据格式

### Content-Type

- 字段⽤于服务器回应时，告诉客户端，本次数据是什么格式。

### Accept-Encoding

- 字段用于客户端请求时，表明自己可以接受哪些压缩方法

### Content-Encoding

- 字段说明数据的压缩⽅法。表示服务器返回的数据使⽤了什么压缩格式

## 请求方法

### HTTP1.0

#### GET

- GET方法用于使用给定的URI从给定服务器中检索信息，即从指定资源中请求数据。使用GET方法的请求应该只是检索数据，并且不应对数据产生其他影响，因此具有**幂等性**

#### HEAD

- 和GET方法的行为很类似，但服务器在响应中只返回头部信息

#### POST

- 新增或提交数据。会修改服务器上的资源，所以是**不安全**的，且多次提交数据就会创建多个资源，所以**不是幂等**的。

### HTTP1.1

#### PUT

- 用于将数据发送到服务器以创建或更新资源，它可以用上传的内容替换目标资源中的所有当前内容，是**幂等**的

#### DELETE

- 用来删除指定的资源，它会删除URI给出的目标资源的所有当前内容

#### TRACE、OPTIONS、CONNECT

- 实际开发中基本不使用

## 优缺点

### 优点

- 简单
- 灵活易拓展
- 跨平台

### 缺点

- 无状态
- 明文传输
- 不安全
  - 明文传输可能被监听
  - 不知道通行放的身份，可能遭遇伪装
  - 无法证明报文的完整性
