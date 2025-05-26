---
title: 初识protobuf
date: 2022-04-11 12:49:31
tags:
- protobuf
- 数据传输
categories:
- Golang
- protobuf
index_img: https://whitestorm0316.github.io/picx-images-hosting/image.8dx3r8w506.jpg
---



# 初识protobuf

> **Protocol buffers are Google's language-neutral, platform-neutral, extensible mechanism for serializing structured data – think XML, but smaller, faster, and simpler. You define how you want your data to be structured once, then you can use special generated source code to easily write and read your structured data to and from a variety of data streams and using a variety of languages.**

## 安装环境

* **进入**[下载地址](https://github.com/protocolbuffers/protobuf/releases)选择对应的系统版本和语言下载
* **双击解压**
* **并将其配置到系统环境变量中**
* **控制台输入protoc --version查看是否安装成功**
* **安装go语言版本的代码生成插件** `go get github.com/golang/protobuf/protoc-gen-go`

## 基本语法

### 定义消息类型

```
syntax = "proto3";
option go_package="./;hello";
message SearchRequest{
    string query = 1;
    int32 page_number = 2;
    int32 result_per_page = 3;
}
```

* `syntax = "proto3";`表示使用的是proto3语法，如果没有这一句的话默认使用的是proto2的语法
* `SearchRequest`表示该消息的名字
* **消息包含的三个字段，并非传统的键值对，例如** `string query=1`表示字段的类型是string，字段的名字是query，字段编号是1
* `option go_package`中两个参数 `./`和 `hello`，分别表示生成的go文件的存放地址和go文件所属的包名

#### 字段类型

* **示例中的三个字段都为标量字段，也可是枚举或者自定义的复合类型**

#### 字段编号

* **消息定义中的每个字段都需要一个****唯一**的编号，这些编号是在消息二进制序列化标识字段的
* **编号的范围为1~2^29-1**
* **不能使用19000~19999，这些是protobuf内置的保留字段编号**
* **同样的，也不能使用自己的保留字段编号**

#### 字段规则

* **字段默认出现为0或1次**
* `repeated`：该字段可以在消息中出现重复任意次数（包括零次）。重复值的顺序将被保留。只能标量类型字段使用
* **字段规则加在类型前面**

#### 保留字段

> **即不能使用的字段编号或字段名称，否则编译器就会报错**

```
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```

* **编译该段proto程序**

  ```
  syntax = "proto3";
  option go_package="./;hello";
  message SearchRequest{
      string query = 1;
      int32 page_number = 2;
      int32 result_per_page = 3;
      reserved 2;
  }
  ```

  **会出现错误信息**

  ```
  $ hello.proto: Field "page_number" uses reserved number 2.
  ```

### proto类型和go类型对比

| **.proto Type** | **Go Type** |
| --------------------- | ----------------- |
| **double**      | **float64** |
| **float**       | **float32** |
| **int32**       | **int32**   |
| **int64**       | **int64**   |
| **uint32**      | **uint32**  |
| **uint64**      | **uint64**  |
| **sint32**      | **int32**   |
| **sint64**      | **int64**   |
| **fixed32**     | **uint32**  |
| **fixed64**     | **uint64**  |
| **sfixed32**    | **int32**   |
| **sfixed64**    | **int64**   |
| **bool**        | **bool**    |
| **string**      | **string**  |
| **bytes**       | **[]byte**  |

### 默认值

> **解析编码时，如果字段没有被赋值，则使用默认值**

* **对于字符串，默认值为空字符串。**
* **对于字节，默认值为空字节。**
* **对于布尔值，默认值为 false。**
* **对于数字类型，默认值为零。**
* **对于enums，默认值是第一个定义的 enum value，它必须是 0。**

### 枚举

> **枚举第一个元素必须为0**

* **消息中的枚举**
  ```
  message SearchRequest {
    enum Corpus {
      UNIVERSAL = 0;
      WEB = 1;
      IMAGES = 2;
      LOCAL = 3;
      NEWS = 4;
      PRODUCTS = 5;
      VIDEO = 6;
    }
    Corpus corpus = 1;
  }
  ```
* **对于消息中的枚举（如上面的枚举），类型名称以消息名称开头：**
  ```
  type SearchRequest_Corpus int32
  ```
* **对于包级的枚举**
  ```
  enum Foo {
    DEFAULT_BAR = 0;
    BAR_BELLS = 1;
    BAR_B_CUE = 2;
  }
  ```
* **类型名称则为原始枚举名称**
  ```
  type Foo int32
  ```

### 使用其他消息类型

```
message SearchResponse {
  repeated Result results = 1;
}

message Result {
  string url = 1;
  string title = 2;
  repeated string snippets = 3;
}
```

### 嵌套类型

```
message SearchResponse {
  message Result {
    string url = 1;
    string title = 2;
    repeated string snippets = 3;
  }
  repeated Result results = 1;
}
```

### map

```
map<key_type, value_type> map_field = N;
```

* **map字段不能是** `repeated`
* **map中的key可以是除enum和float之外的任何标量**

### Service

```
service SearchService {
  rpc Search(SearchRequest) returns (SearchResponse);
}
```

* **利用grpc插件生成代码**
  ```
  $ protoc --go_out=plugins=grpc:. hello.proto
  ```

**参考：**

**[1]**[https://developers.google.cn/protocol-buffers/docs/proto3](https://developers.google.cn/protocol-buffers/docs/proto3)

**[2]**[https://developers.google.cn/protocol-buffers/docs/reference/go-generated](https://developers.google.cn/protocol-buffers/docs/reference/go-generated)
