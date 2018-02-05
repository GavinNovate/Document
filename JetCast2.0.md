# JetCast 2.0 方案

###### 简介

JetCast主要是用于公司仪表端和中控端互相通讯的协议。2.0是在1.0的基础上重新设计的版本。

本文描述了JetCast2.0简洁化的方案。主要剔除了繁杂的版本校验，使用简便的方式做版本兼容。

## 角色定义

- Master(Client)：中控端，Android设备，IOS设备
- Slave(Server)：仪表端

## 过程描述

中控与仪表建立起局域网，可以是中控仪表一体设备的本地环回网络，也可以是中控仪表之间建立的无线连接。

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
流程说明：

1. Slave开启UDP监听，固定监听一个端口。
2. Master开启UDP监听，监听固定范围内一个未被使用的端口。
3. Master向局域网发送Discover广播，并且携带自己监听的端口号和随机生成的RSA加密公钥。
4. Slave收到Discover广播，读取到Master监听的端口号和Master的公钥。
5. Slave随机生成一个AES的加密秘钥，使用Master的公钥加密后向Master监听的端口发送Discover OK广播。
6. Master收到Discover OK广播，使用自己的私钥解密，拿到AES秘钥。
7. Master使用AES秘钥加密自己的设备唯一编号和AES秘钥本身给Slave发送Binding广播。
8. Slave收到Binding广播，使用AES秘钥解密，验证AES秘钥是否正确（其实能解密就可以验证了），然后保存Master的设备唯一编号。
9. Slave开启TCP监听，Slave把监听端口放到Binding OK发送给Master。
10. Master收到Binding OK包，关闭UDP监听，开启TCP链接。然后Master和Slave之间就可以互发数据了。

说明：

1. 关于伪装客户端：服务端接受到Binding包，读取设备唯一标识提示给用户，用户确认后才回应Binding OK包
2. 关于伪装服务端：客户端收到Discover OK包时，提示给用户选择。

## 使用端口



## 加密传输



## 协议内容

协议分为三层，分别为表示层，应用层和业务层。

表示层主要负责描述数据类型，数据加密方式和多个数据包的对应。

应用层

### 表示层

UDP传输层 报文

```
Context-Code(16 Bytes):会话UUID 确认一组UDP传输
Content-Type(2 Bytes):数据类型	01:Byte 02:Json
Encrypt-Type(2 Bytes):加密方式	00:None	01:RSA	02:AES
Data(~ Bytes):数据内容
```



TCP传输层 报文

```
Context-Code(16 Bytes):会话UUID 用于对应请求和响应关系
Content-Type(2 Bytes):数据类型
Encrypt-Type(2 Bytes):加密方式
Content-Length(4 Bytes)：数据长度
Data(~ Bytes)：数据内容
```

说明：

1. 传输层无升级概念
2. 数据类型为Byte时候，每条字段采取 index-length-content 三段式存储，或者是 length-content 两段式存储
3. 加密方式表明Data内容的加密方式



### 应用层

UDP包装

```json
{
  "Resource":"Discover",	// 资源 - 这个包里面装的东西是什么
  "Data":{					// 数据 - 该资源包含的具体数据
    
  },
  "Port":10000				// 回应端口 - 表示我会监听哪个端口，等待对方的回应
}
```



TCP包装

Request

```json
{
  "Action":"POST",			// 请求类型 分为 GET/POST/DELETE
  "Resource":"Theme",		// 请求资源类型
  "Data":{					// 请求体 - 可空
    
  }
}
```

Response

```json
{
  "Status":200,				// 响应状态码
  "Error":"Error Message",	// 响应信息
  "Data":{					// 响应体
    
  }
}
```

说明：

兼容方案：重载

1. 当协议升级时候，不要修改以前Data内的字段，而应该添加新的字段
2. 添加的字段应该是可空字段，新协议要对空值进行兼容
3. 当老协议给新的协议发数据时，新协议应忽略空值/补充默认值，然后按新协议处理
4. 当新协议给老协议发送数据时，老协议不会读到新协议的字段，然后自动按照老协议处理
5. 对于不能处理的Resource类型，给予提示



使用该方案的原因：

如果使用版本兼容的方式

即使我们在握手期间做好了版本对接，两边SDK拿到了版本号，但如何去使用这个版本号依然是一个头疼的问题。

举个例子：Master端支持1、2、3版本，Slave支持1、2版本，取版本交集最大值2作为双方约定的版本号。假如有个通信协议是设置主题，版本3的设置主题比版本2多了一个属性。这时Master端调用了一个发送主题的函数，然后SDK要根据版本号对这个函数的处理方式做修正。随着版本增多，这里的处理方式会令人非常头疼。而对于不同版本中的相同处理，比如某个API升级后完全没变，也显得很累赘。



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


