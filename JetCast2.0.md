# JetCast 2.0 协议

简介：……

## 分层

协议分为传输层和应用层，具体定义如下

### 传输层

UDP传输层 报文

```
Content-Type(2 Bytes):数据类型
Encrypt-Type(2 Bytes):加密方式
Data(~ Bytes):数据内容
```



TCP传输层 报文

```
Content-Type(2 Bytes):数据类型
Encrypt-Type(2 Bytes):加密方式
Content-Length(4 Bytes)：数据长度
Data(~ Bytes)：数据内容
```

说明：

1. 新旧协议不能互通
2. 一般情况，多个协议版本应该共存，而且支持高版本必然支持低版本
3. 一般情况，旧业务使用的协议应该继续保持，如果旧业务使用新协议，要使用新增的方式（如何判断使用旧的方式还是新的方式？）



说明：

1. 协议版本用来升级传输层协议，初始版本为 00 01，升级依次递增
2. 数据类型指定Data中数据的类型，00 01 为Byte，00 02 为 Text，00 03 为Json
3. 加密方式：指定Data中数据加密的方式 00 00 不加密，00 01 RSA加密，00 02 AES加密
4. 数据偏移指的是Data前的字节数，读取数据偏移值可以确定Data字节的起始位置
5. 在TCP传输中，数据长度用来标识Data的长度，与数据偏移值一起，可以确定Data的起始位置和长度
6. Data为具体数据内容
7. 传输层协议升级，不应该影响到数据内容，传输层升级只是对一些特性的升级，对于高低版本兼容来说，高版本可以提出一些特性，即使低版本可能不支持，也可以准确读取到Data中的数据。

### 业务层

UDP表现为无脑传输，但是要求双方知道各自是在说一件事，使用需要一个Context，语境

TCP是请求-响应方式，Request 对应 Response ，需要用相同的ID来标识



Discover		Master -> Slave

```
Resource:Discover	这个东西是什么？
Context:100	谁在和我对话，我们之前说过什么？

Service-Type:请求的服务类型
// Support-Version:支持的协议版本

Port:回应端口	我是谁？你应该回消息给谁？
Public-Key:Master公钥		你应该用什么加密消息给我？

本帧是否加密？ - 应该在外部提示
```

Discover OK	 Slave ->Master

```
Resource:DiscoverOK
Context:100 我是回答谁的？我们刚才说的说什么？

Service-Status:当前服务器的状态
// Support-Version：支持的协议版本
ASK：提问 - 可以是一个随机数

// Port:回应端口
Public-Key:Slave公钥

本帧加密
```

Binding		Master->Slave

```
Resource:Binding
Context:100

Answer:回答 - 回答Slave的提问，原样返回即可
UUID：Master的设备唯一标识号

// Port:回应端口

加密
```

BindingOK Slave -> Master

```
Resource:BindingOK
Context:100

Answer:随机数

加密
```



## 角色定义

- Master 主机
- Slave 丛机

## 传输过程

时序图

```sequence
participant Master as M
participant Slave as S
Note over M,S:UDP
Note over S:开启UDP监听
Note over M:开启UDP监听
M->S:发送Discover
Note over S:收到Discover
S->M:回应Discover OK
Note over M:收到Discover OK
M->S:发送Binding
Note over S:收到Binding
Note over M,S:TCP
Note over S:开启TCP监听
S->M:回应Binding OK
Note over M:收到 Binding OK,关闭UDP监听
Note over M:开启TCP链接
M->S:发送数据
S->M:发送数据
```

## 数据帧描述

### UDP数据帧

1. Discover

   ```
   Type:00 01
   Length:数据长度
   Protocol version：支持的协议版本

   Port:标识Discover OK接收的端口
   Key:Master的公钥
   ```

   说明：

   1. Port：Master监听Discover OK的端口，防止多个应用需要同时发送Discover，但是端口只能被一个应用监听，所以采用固定范围内随机使用，然后把端口传给Slave端
   2. Key：Master的公钥，Slave给Master发送信息使用这个秘钥加密

2. Discover OK

   ```
   Type：00 02
   Length：后面字节的长度
   IP:IP地址
   Status:00--Ready 01--Busy
   Protocol version:支持的协议版本

   Key：Slave的公钥
   Random:随机数
   ```

   说明：

   1. Key：Slave的公钥，Master给Slave发送信息时使用这个秘钥加密
   2. Random：随机数
   3. 本帧数据使用Master的公钥加密

3. Binding

   ```
   Type：帧类型
   Random:随机数
   UUID：Master设备的唯一标识
   ```

   说明：

   1. Random:Discover OK包发来的随机数，原样返回
   2. UUID：设备的唯一标识符
   3. 本帧数据使用Slave的公钥加密

4. Binding OK

   ```
   Type:帧类型
   Random:随机数
   ```

   说明：

   1. Random:随机数，防止其他设备伪装服务器
   2. 本帧数据使用Master的公钥加密



