---
title: Redis缓存一致性问题
date: 2024-10-07 21:02:39
tags:
- redis
- 缓存
- 一致性问题
categories:
- Redis
index_img: https://whitestorm0316.github.io/picx-images-hosting/image.9gwt24u8be.jpg
---
# redis缓存一致性问题

> **在实际业务中，经常会涉及到缓存，那么保持缓存数据和数据库数据之间的一致性同样是一个非常重要的举措**

## 三种常见的缓存更新方案

### 方案一| 先更新数据库，再更新缓存

![方案一流程](https://raw.githubusercontent.com/GitWhitestorm/blog-image/master/img/image-20220407153127364.png)

**问题：**

* **高并发时，如果线程A更新完数据库，线程B再次更新数据库，线程B比线程A先一步更新缓存，那么缓存还是线程A的数据，数据库是线程B的数据**

### 方案二 | 先删除缓存，再更新数据库

![方案二流程](https://raw.githubusercontent.com/GitWhitestorm/blog-image/master/img/image-20220407155035593.png)

**问题：**

* **线程A更新数据，先删除缓存，此时，线程B读取数据，因为此时线程A还未上锁，所以线程B可以从数据库读取数据，并且再次存入缓存中**

#### 改进| 延时双删

* **即根据业务需要，在延迟多少时间后再把缓存删除**

**问题：**

* **延迟删除，会影响接口吞吐量，影响接口性能**
* **第二次删除可能存在问题**

### 方案三| 先更新数据库，再删除缓存

![方案三流程](https://raw.githubusercontent.com/GitWhitestorm/blog-image/master/img/image-20220407201533461.png)

**问题：**

* **线程A读取数据，恰好此时没有缓存时，从数据库读到旧值，线程B更新新值到数据库后删除缓存，删除缓存，线程A更新缓存**
* **概率较低，数据库操作中极少出现查询比更新慢的情况**

## 基于消息的队列| 异步更新缓存

> **基于订阅binlog的同步机制**

* **模仿mysql的主从复制交互协议，将自己伪装成mysql slave 向mysql master 发送dump协议**
* **mysql master收到dump请求，推送bin log给的slave**
* **解析bin log对象**
* **利用消息队列进行分发消费**

![binlog异步订阅流程](https://raw.githubusercontent.com/GitWhitestorm/blog-image/master/img/image-20220407210105040.png)

## 最终一致性

* **给缓存添加过期时间可以保证缓存的最终一致性**
