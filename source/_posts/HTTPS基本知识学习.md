---
title: HTTPS基本知识学习
date: 2023-04-24 09:39:24
tags:
- HTTPS
- 计算机网络
categories:
- 计算机网络
index_img: https://whitestorm0316.github.io/picx-images-hosting/image.4qrk3pc2oc.jpg
---
---

# HTTPS的基本认识

> HTTPS （全称：Hyper Text Transfer Protocol over SecureSocket Layer），是以安全为目标的 HTTP 通道，在HTTP的基础上通过传输加密和[身份认证]保证了传输过程的安全性

## HTTP的缺点

- 无状态
- 明文传输
- 不安全

## HTTP与HTTPS的区别

- HTTP 是超⽂本传输协议，信息是明⽂传输，存在安全⻛险的问题。HTTPS 则解决 HTTP 不安全的缺陷，在

  TCP 和 HTTP ⽹络层之间加⼊了 SSL/TLS 安全协议，使得报⽂能够加密传输
- HTTP 连接建⽴相对简单， TCP 三次握⼿之后便可进⾏ HTTP 的报⽂传输。⽽ HTTPS 在 TCP 三次握⼿之

  后，还需进⾏ SSL/TLS 的握⼿过程，才可进⼊加密报⽂传输
- HTTP 的端⼝号是 80，HTTPS 的端⼝号是 443
- HTTPS 协议需要向 CA（证书权威机构）申请数字证书，来保证服务器的身份是可信的

## HTTPS解决的问题

- 窃听风险，HTTP是明文传输，通行内容很容易被窃取
- 篡改风险，强制植⼊垃圾⼴告，视觉污染
- 冒充风险，冒充知名网站，如京东，淘宝

## HTTPS解决的方案

### 混合加密

- 在SSL/TLS握手中，使用非对称加密算法加密
- 在SSL/TLS握手后，使用对称加密算法加密

#### SSL协议基本流程

- 客户端向服务器索要并验证服务器的公钥
- 双⽅协商⽣产「会话秘钥」
- 双⽅采⽤「会话秘钥」进⾏加密通信

### 摘要算法

- **摘要算法**⽤来实现**完整性**，能够为数据⽣成独⼀⽆⼆的「指纹」，⽤于校验数据的完整性，解决了篡改的⻛险

### 数字证书

- 将**服务器公钥放在数字证书**，防止客户端在获取服务器公钥的时候被篡改

## HTTPS建立连接的过程

![TLS握手过程](https://raw.githubusercontent.com/GitWhitestorm/blog-image/master/img/image-20220424145240412.png)

### 第一次握手

- 客户端发起ClientHello
  - 服务器随机数C
  - 支持的TLS版本
  - 支持的加密套件

### 第二次握手

- 服务器回应ServerHello

  - 服务器随机数S
  - 确认使用的加密通信协议版本
  - 确认使用的加密方法
  - 服务器证书

### 第三次握手

- 客户端回应
  - 验证证书的合法性
  - 如果证书受信任，则发送经过服务器公钥加密的随机数
  - 以后的通信使用会话秘钥加密发送
  - 之前所有握手信息的摘要

### 第四次握手

- 服务端回应
  - 确认收到
  - 以后的通信使用会话秘钥加密发送
  - 之前所有握手信息的摘要
