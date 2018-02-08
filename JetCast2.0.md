# JetCast2.0设计方案

JetCast主要局域网内多个设备间的通讯。

JetCast2.0在1.0的基础上增加了安全性和扩展性。

## 角色定义

- Master(Client)：主设备
- Slave(Server)：从设备

## 使用端口

Slave UDP监听端口：固定端口	如：30000

Slave TCP监听端口：不同服务对应不同端口 如：30001~30099

Master UDP监听端口：固定范围内的随机未被占用的端口 如：30100~30999

## 协议描述

1. Master端在局域网内发送特定广播，寻找Slave端。
2. Slave端收到广播，响应Master端，两端建立TCP连接，进行通讯。

## 协议内容

协议规定了UDP和TCP数据包的格式

UDP数据包格式

```
Protocol-Version(2 Bytes):协议版本
Context-Code(8 Bytes):会话编码	同一组广播使用同一个会话编码
Encrypt-Type(2 Bytes):加密方式	00:None	01:RSA	02:AES
Content-Type(2 Bytes):数据类型	01:Byte 02:Json
Data(~ Bytes):数据内容
```

TCP数据包格式

```
Protocol-Version(2 Bytes):协议版本
Context-Code(8 Bytes):会话编码	一对请求响应使用同一个会话编码
Encrypt-Type(2 Bytes):加密方式
Content-Type(2 Bytes):数据类型
Content-Length(4 Bytes)：数据长度
Data(~ Bytes)：数据内容
```

说明：

1. 协议版本：描述当前协议版本，预留协议升级的能力
2. 会话编码：描述数据包的对应关系，应对并发场景
3. 加密方式：描述数据内容的加密方式，分别为不加密，RSA加密和AES加密
4. 数据类型：描述数据内容的数据类型，分别为Byte字节流和Json字符流
5. 数据长度：TCP包中，确认数据内容的长度，用以切割数据包
6. 数据内容：具体数据

------

协议规定了Byte和Json数据类型的通用格式

Byte类型

// Todo



Json类型

UDP包

```json
{
  "Action":"Discover",		// 动作 - 描述数据包的动机
  "Data":{					// 数据 - 需携带的数据
    
  },
  "Port":10000,				// 回应端口 - 对方应该回应我的端口
}
```

TCP包

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



## 握手过程

协议同步

1. Master发起Discover广播，查找局域网内的Slave设备。

   ```json
   {
     "Action":"Discover",
     "Data":{
       "AcceptVersion":[1,2,3],	// 支持的协议版本
     },
     "Port":10000
   }
   ```

   说明：该数据包必须使用初始的协议版本发送，格式不可改变，不加密，以保证所有版本的协议都可以正确识别。

2. Slave收到Discover广播，回应DiscoverResponse广播。

   ```json
   {
     "Action":"DiscoverResponse",
     "Data":{
       "AcceptVersion":[1,2,3]	// 支持的协议版本
     },
     "Port":10000
   }
   ```

   说明：该数据包可以选用Master支持的任何版本协议进行发送，格式尽量不做改变。

秘钥交换

1. Master向Slave发起Encrypt广播。

   ```json
   {
     "Action":"Encrypt",
     "Data":{
       "Type":"RSA"			// 加密方式
       "Key":"PublicKey"		// Master公钥
     },
     "Port":10000
   }
   ```

   说明：Master向Slave发送公钥，该数据包不需要加密

2. Slave给Master回应EncryptResponse广播。

   ```json
   {
     "Action":"EncryptResponse",
     "Data":{
       "Type":"RSA"			// 加密方式
       "Key":"PublicKey"		// Slave公钥
     }
     "Port":10000
   }
   ```

   说明：Slave向Master发送公钥，该数据包不需要加密

3. Master继续向Slave发起Encrypt广播，秘钥类型为AES，使用Slave公钥加密。

4. Slave回应Master，秘钥类型为AES，使用AES方式加密，Master做个验证。

注意：

1. 以上两个步骤分别做了协议同步和秘钥交换的工作，至此，双方可以互相秘密地进行通讯。
2. 互相可信地通讯并不代表通讯双方是真实可信的，即，可能有一方或者两方为伪造的，下文将会对验证身份进行描述。

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


## 业务内容

Discover			Master -> Slave	不加密

```json
{
  "Resource":"Discover",
  "Data":{
    "AcceptVersion":[1,2,3],	// 支持的协议版本
  },
  "Port":10000
}
```

DiscoverOK		Slave -> Master	不加密

```json
{
  "Resource":"DiscoverOK",
  "Data":{
    "AcceptVersion":[1,2,3]	// 支持的协议版本
  },
  "Port":10000
}
```

Encrypt			Master -> Slave	不加密

```json
{
  "Resource":"Encrypt",
  "Data":{
    "PublicKey":"PublicKey"		// Master公钥
  }
  "Port":10000
}
```

EncryptOK		Slave -> Master	用Master公钥加密

```json
{
  "Resource":"EncryptOK",
  "Data":{
    "SecretKey":"SecretKey"		// AES秘钥
  }
  "Port":10000
}
```



Verify	校验	Master->Slave 发起校验请求

```json
{
  "Action":"Verify",
  "Port":10000
}
```

VerifyOK

```json
{
  "Action":"VerifyOK",
  "Data":{
    "Salt":"Salt",		// 随机盐
    "Hash":"Hash"		// Hash(Salt+数字密码)
  }
  "Port":10000
}
```

Register

```json
{
  "Resource":"Register",
  "Data":{
    "UUID":"UUID",
    "Salt":"Salt",
    "Hash":"Hash"
  },
  "Port":10000
}
```

RegisterOK

```json
{
  "Resource":"RegisterOK",
  "Data":{
    "UUID":"UUID",
    "Password":"Password"
  }
  "Port":10000
}
```

Login

```json
{
  "Resource":"Login",
  "Data":{
    "UUID":"UUID",
    "Salt":"Salt",
    "Hash":"Hash"	// Hash(Salt+Password)
  },
  "Port":10000
}
```

LoginOK

```json
{
  "Resource":"LoginOK",
  "Data":{
    "UUID":"UUID",
    "Salt":"Salt",
    "Hash":"Hash",	// Hash(Salt+Password)
    "Port":30001
  },
  "Port":10000
}
```



## 加密验证

1. Master发送Discover包，内容如下

   ```json
   {
     "Resource":"Discover",
     "Data":{
       "Service":"Music",		// 请求服务类型
       "Key":"1234",			// Master公钥
       "UUID":"uuid",			// 可空 Master设备的UUID
       "Salt":"SALT"			// 可空 盐
       "Hash":"HASH"			// 可空 散列 Hash(Slat+数字密码)
     },
     "Port":10000
   }
   ```

   说明：

   1. Service和Key容易理解。
   2. UUID：Master的设备唯一标识。
   3. Hash：Hash(UUID+数字密码)，使用UUID+设备存储的数字密码然后用MD5散列。

2. Slave收到Discover包

   判断UUID和Hash是否为空。

   1. 如果任一为空，就表明Master没有匹配过任何Slave，Slave端显示四位数字密码和它的二维码。
   2. 如果Salt和Hash都不为空，Slave去存储块查找对应的UUID和数字密码，如果查询到，把UUID和数字密码散列进行比对，通过就表示认证通过，说明之前已经获得了用户授权。没通过就显示四位数字密码和它的二维码。

3. Slave回应DiscoverOK包

   ​

## 升级兼容

方案一：重载（开闭原则）

1. 当协议升级时候，不要修改以前Data内的字段，而应该添加新的字段
2. 添加的字段应该是可空字段，新协议要对空值进行兼容
3. 当老协议给新的协议发数据时，新协议应忽略空值/补充默认值，然后按新协议处理
4. 当新协议给老协议发送数据时，老协议不会读到新协议的字段，然后自动按照老协议处理
5. 对于不能处理的Resource类型，给予提示

使用该方案的原因：

如果使用版本兼容的方式

即使我们在握手期间做好了版本对接，两边SDK拿到了版本号，但如何去使用这个版本号依然是一个头疼的问题。

举个例子：Master端支持1、2、3版本，Slave支持1、2版本，取版本交集最大值2作为双方约定的版本号。假如有个通信协议是设置主题，版本3的设置主题比版本2多了一个属性。这时Master端调用了一个发送主题的函数，然后SDK要根据版本号对这个函数的处理方式做修正。随着版本增多，这里的处理方式会令人非常头疼。而对于不同版本中的相同处理，比如某个API升级后完全没变，也显得很累赘。



方案二：在方案一的基础上，提供协议版本约定，但是可能对兼容性产生影响