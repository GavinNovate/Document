# JetCast Transfer Protocol

JetCast2.0协议用于局域网内多个设备间的通讯。

协议使用"请求-响应"模式，解决业务间的耦合调用，拆分逻辑，利于扩展。

协议使用RESTful设计风格，面向资源设计API，易于维护和扩展。

## 角色定义

- Server：服务端，被发现，提供服务，建立连接后可主动发起请求
- Client：客户端，主动发现，请求服务，建立连接后也需响应服务端的请求

## 协议描述

1. Client在局域网内发送特定广播寻找Server。
2. Server收到广播，响应Client，两端建立TCP连接，进行通讯。

## 使用端口

Server UDP监听端口：固定端口	如：30000

Server TCP监听端口：不同服务对应不同端口 如：30001~30099

Client UDP监听端口：固定范围内的随机未被占用的端口 如：30100~30999

## 数据格式



```
+---------+--------+-------------+~~~~~~~~+-------------+~~~~~~~~+-------------+~~~~~~~~+
| Encrytp | Action | Line Length |  Line  | Head Length |  Head  | Body Length |  Body  |
+---------+--------+-------------+~~~~~~~~+-------------+~~~~~~~~+-------------+~~~~~~~~+

固定字节数字段
+--------+
|        |
+--------+

不定字节数字段
+~~~~~~~~+
|        |
+~~~~~~~~+

Encrytp	(1 Byte):加密方式 0x00 不加密 0x01 AES 0x02 RSA
Action	(1 Byte):请求 Or 响应 0x01 请求 0x02 响应

```



### 应用层

应用层借鉴HTTP协议，将数据包分为**请求**和**响应**两部分，格式如下：

#### 请求

```java
String protocol;		// 协议版本
String method;			// 请求方法 GET/PUT/POST/DELETE
String resource;		// 请求资源

Header header;			// 请求头
byte[] body;			// 请求体
```

#### 响应

```java
String protocol;		// 协议版本
int status；				// 响应状态
String message;			// 响应信息

Header header;			// 响应头
byte[] body;			// 响应体
```



### 表示层

表示层在应用层数据包前加一个字节的数据，用于表示应用层数据包整体的加密方式

```
Encryption(1 Byte):加密方式 0x00 不加密，0x01 AES加密，0x02 RSA加密

+------------+~~~~~~~~~~~~~~+
| Encryption |  应用层数据包  |
+------------+~~~~~~~~~~~~~~+
```



### 会话层

会话层在表示层数据包前加九字节数据，用于表示数据包的会话信息

```
+---------+---------+---------+------------+~~~~~~~~~~~+
| length  | session | context | encryption |  Package  |
+---------+---------+---------+------------+~~~~~~~~~~~+


```



### UDP数据帧

UDP数据帧使用一个字节来区分数据帧的传输方向(Request/Response)。

传输方向 会话场景 加密方式 

协议规定了数据包格式和数据包中的数据内容格式

名词解释：

1. 数据包(Package)：一次UDP或TCP传输的数据，其中TCP传输时需使用Length-Package格式分割。
2. 数据内容(Content)：数据包中用于存放具体数据的字段，分为请求和响应两种数据。

### 数据包

数据包使用**Msgpack**序列化方式，将如下格式的数据包序列化

```java
String protocol;		// 协议版本	"JCTP/2.0"
String transfer;		-- // 传输方向 请求:"Request" 响应:"Response"
String context;			-- // 会话场景
String contentType;		// 数据类型	字节流:"Msgpack" 字符流:"Json"
String contentEncrypt;	-- // 加密方式	对称加密:"AES" 非对称加密:"RSA"
byte[] content;			// 数据内容
```

说明：

1. 协议版本：描述当前协议版本
2. 传输方向：描述数据包的传输方向是**主动请求**还是**被动响应**
3. 会话场景：描述数据包**请求响应**的对应关系
4. 数据类型：描述数据内容的数据类型，**字节流**使用Msgpack方式传输，**字符流**使用Json方式传输
5. 加密方式：描述数据内容的加密方式，分为**不加密**(空值/无此字段)，**对称加密**(AES)和**非对称加密**(RSA)
6. 数据内容：具体数据

### 数据内容

数据内容分为两类，分别为**请求**(Request)和**响应**(Response)，可以使用Msgpack或Json序列化

#### 请求

```java
String action;			// 请求类型 GET/PUT/POST/DELETE
String resource;		// 请求资源名称
Object data;			// 请求资源体 
Integer port;			// 回应端口 - UDP专用，TCP为空(无此字段)
```

#### 响应

```java
Integer status；			// 状态码
String message;			// 提示信息
Object data;			// 响应体
```





请求报文

```
protocol method resource
protocol status message
```



## 协议接口

1. 协议发现

   - GET请求

     请求：Client/Server

     响应：Server/Client

     加密：不加密

     描述：请求对方支持的协议版本

     Request

     ```json
     {
       "Action":"GET",				// 标注为GET请求
       "Resource":"Protocol",		// 标注请求的内容
       "Port":30000
     }
     ```

     Response

     ```json
     {
       "Status":200,					// 状态码
       "Error":"Success",			// 响应信息
       "Data":{
         "AcceptVersion":[1,2,3]		// 支持的协议版本
       },
       "Port":30000
     }
     ```

   - POST请求

     请求：Client

     响应：Server

     加密：不加密

     描述：发送自己支持的协议版本

     说明：客户端使用这个接口来发现支持此协议的服务端

     Request

     ```json
     {
       "Action":"POST",
       "Resource":"Protocol",
       "Data":{
         "AcceptVersion":[1,2,3],	// 支持的协议版本
       },
       "Port":30000
     }
     ```

     Response

     ```json
     {
       "Status":200,				// 状态码
       "Error":"Success",		// 响应信息
       "Port":30000				// 回应端口
     }
     ```

     ​

2. 秘钥交换

   - GET请求

     请求：Client/Server

     响应：Server/Client

     加密：请求RSA公钥不加密，请求AES秘钥需要使用RSA/AES方式加密

     描述：请求对方的秘钥

     Request

     ```json
     {
       "Action":"GET",		// 标注为GET请求
       "Resource":"Key",		// 标注请求的内容
       "Port":30000
     }
     ```

     Response

     ```json
     {
       "Status":200,			// 状态码
       "Error":"Success",	// 响应信息
       "Data":{
         "Type":"RSA",		// 秘钥类型 RSA/AES
         "Key":"Key"			// 秘钥类型
       },
       "Port":30000
     }
     ```

   - POST请求

     请求：Client

     响应：Server

     加密：发送RSA公钥不加密，发送AES秘钥需要使用RSA/AES方式加密

     描述：发送自己的秘钥

     Request

     ```json
     {
       "Action":"POST",
       "Resource":"Key",
       "Data":{
         "Type":"RSA",		// 秘钥类型 RSA/AES
         "Key":"Key"			// 秘钥类型
       },
       "Port":30000
     }
     ```

     Response

     ```json
     {
       "Status":200,				// 状态码
       "Error":"Success",		// 响应信息
       "Port":30000				// 回应端口
     }
     ```

3. 请求验证码

   - GET请求

     请求：Client

     响应：Server

     加密：None/RSA/AES

     描述：请求验证码

     说明：客户端请求服务端产生一个验证码并显示在屏幕上，服务端产生后，把验证码Hash后返回给客户端

     Request

     ```json
     {
       "Action":"GET",			// 标注为GET请求
       "Resource":"Captcha",		// 标注请求的内容
       "Port":30000
     }
     ```

     Response

     ```json
     {
       "Status":200,			// 状态码
       "Error":"Success",	// 响应信息
       "Data":{
      	"Salt":"Salt",		// 随机盐
         "Hash":"Hash"		// Hash(Salt+数字验证码)
       },
       "Port":30000
     }
     ```

4. 校验身份

   - GET请求

     请求：Client

     响应：Server

     加密：AES

     描述：从服务端获取身份信息

     Request

     ```json
     {
       "Action":"GET",
       "Resource":"User",
       "Resource":"UUID",
       "Port":30000
     }
     ```

     Response

     ```json
     {
       "Status":200,				// 状态码
       "Error":"Success",		// 响应信息
       "Data":{
         "UUID":"UUID",
         "Salt":"Salt",
         "Hash":"Hash"			// Hash(Salt+Password)
       }
       "Port":30000				// 回应端口
     }
     ```

   - PUT请求

     请求：Client

     响应：Server

     加密：AES

     描述：向服务端提交自己的身份信息

     Request

     ```json
     {
       "Action":"PUT",
       "Resource":"Token",
       "Data":{
         "UUID":"UUID",
         "Salt":"Salt",
         "Hash":"Hash"	// Hash(Salt+Password)
       },
       "Port":30000
     }
     ```

     Response

     ```json
       "Status":200,				// 状态码
       "Error":"Success",		// 响应信息
       "Port":30000				// 回应端口
     ```

   - POST请求

     请求：Client

     响应：Server

     加密：AES

     描述：客户端向服务端注册身份

     说明：客户端显示数字输入框或者二维码取景框，获取服务端显示的验证码，把这个验证码和通过网络获取到的验证码作比较，通过后，把验证码Hash后生成注册请求发给服务端。服务端再做一遍校验，通过后，允许注册。成功后服务端随机生成一个Password，通过响应返回给客户端。服务端和客户端分别存储UUID和Password，用作以后登录的校验。

     Request

     ```json
     {
       "Action":"POST",
       "Resource":"User",
       "Data":{
         "UUID":"UUID",		// 设备/应用 识别码
         "Salt":"Salt",		// 随机盐
         "Hash":"Hash"		// Hash(Salt+数字验证码)
       },
       "Port":30000
     }
     ```

     Response

     ```json
     {
       "Status":200,				// 状态码
       "Error":"Success",		// 响应信息
       "Data":{
         "UUID":"UUID",			// 设备/应用 识别码
         "Password":"Password"	// 密码
       }
       "Port":30000				// 回应端口
     }
     ```

## 握手流程

1. 协议同步
   1. 客户端发起协议同步的POST请求广播，把自己支持的协议版本广播出去，服务端接收到广播后响应。
   2. 客户端发起协议同步的GET请求，查询服务端支持的协议版本，服务端接收到后返回自己支持的版本。
2. 秘钥交换
   - 交换RSA公钥
     1. 客户端发起交换秘钥的POST请求，把自己的RSA公钥发给服务端，服务端收到后响应。
     2. 客户端发起交换秘钥的GET请求，服务端收到后，响应自己的RSA公钥。
   - 交换AES秘钥
     1. 客户端发起交换秘钥的GET请求，请求服务端的AES秘钥，请求部分使用服务端RSA公钥加密。
     2. 服务端收到GET请求后，生成AES秘钥，响应客户端，响应部分使用客户端RSA公钥加密。
3. 身份验证
   - 没有配对过
     1. 客户端发起显示验证码的请求，服务端收到请求后，生成验证码和二维码，显示在屏幕上，并加盐Hash后，把Hash值发回给客户端。
     2. 客户端提示用户输入验证码（可以是输入或者扫码，验证码4-6位即可），待用户输入完毕，校验验证码加盐Hash值是否正确，正确后，客户端发起注册身份的请求，把自己的UUID和验证码的加盐Hash值发给服务端。（注：两次Hash的盐不能相同）
     3. 服务端收到注册请求后，校验验证码和加盐Hash值是否正确，正确后，生成一个复杂的Password回应客户端。并把UUID+Password组存在本地安全文件系统中。
     4. 客户端收到Password也存储到安全的文件系统中。后面的流程按照已经配对过的流程走。
   - 已经配对过
     1. 客户端从安全的文件系统中读取Password，结合自己的UUID，向服务端提交自己的信息。
     2. 服务端收到请求后，校验身份是否正确，然后返回。
     3. 客户端收到正确的消息后，发起Get请求，向服务端请求自己的信息，查看服务端返回自己的消息是否匹配。

## 安全说明

### 信息加密

1. 使用RSA加密方式，传输AES秘钥。
2. 使用AES加密方式，加密大量数据，可以提升加密解密速度。

### 身份校验

1. 身份校验是双向的，客户端需要校验服务端是否真实，服务端也要校验客户端是否真实。
2. 首次配对，需要用户参与，让用户鉴别客户端和服务端的真实性。
3. 服务端生成验证码时，Hash后再传输给客户端，可以防止验证码丢失，确又可以让客户端验证了服务端的真实性。
4. 客户端收到用户输入的验证码时，可以校验服务端的真实，然后再Hash后发给服务端，服务端收到后也可以校验客户端的真实性。
5. 校验后，即可保证双方的信息是安全而真实的。
6. 服务端的数据存在安全的文件系统中，需要防止别盗取。
7. 客户端的数据使用Android的秘钥库加密后存在文件系统中，Android的秘钥库可以保证只有当前应用可以访问，且不会被盗取。

## 升级说明

### 协议升级

为协议标注版本号，在协议同步阶段约定双方可以使用的协议版本。

强调：协议升级不能影响Data中的业务数据，要完全分层，达到协议变更不影响业务的能力。

举例：如果新增了一种加密方式，要在交换秘钥时，做好解密方案，最终提供给业务层的数据是解密后的数据，这样才可以兼容一起的业务。

### 业务升级

业务升级的方案为重载，遵循开闭原则

1. 当协议升级时候，不要修改以前Data内的字段，而应该添加新的字段
2. 添加的字段应该是可空字段，新协议要对空值进行兼容
3. 当老协议给新的协议发数据时，新协议应忽略空值/补充默认值，然后按新协议处理
4. 当新协议给老协议发送数据时，老协议不会读到新协议的字段，然后自动按照老协议处理
5. 对于不能处理的Resource类型，给予提示

## 其他说明

### 请求方式

| 请求方式   | 安全（不会改变服务端资源）？ | 幂等（多次请求结果相同）？ |
| ------ | -------------- | ------------- |
| GET    | 是              | 是             |
| POST   | 否              | 否             |
| PUT    | 否              | 是             |
| DELETE | 否              | 是             |

说明：

1. GET：查，需指定ID，从服务端获取数据，不会改变服务端数据，多次请求结果相同
2. POST：增，向服务端新建一个数据，会改变服务端数据，多次请求创建多次
3. PUT：改，需指定ID，向服务端修改一个数据，会改变服务端数据，多次请求数据相同
4. DELETE：删，需指定ID，从服务端删除一个数据，会改变服务端数据，多次请求不会删除多余数据




### 序列化方式

Data字段序列化方式

1. [MessagePack](https://msgpack.org)

   MessagePack是一种高效的二进制序列化格式。它类似JSON，但更快并且更小。

2. [JSON](http://json.org)

   JSON是一种轻量级的数据交换格式，采用完全独立于语言的文本格式，是理想的数据交换语言。

对比：

1. MessagePack和JSON序列化体积对比：
   1. MessagePack基本总是比JSON小一些，具体幅度需要看序列化的对象的内容。
2. MessagePack相对JSON解析速度对比：
   1. MessagePack对二进制数据特别友好，二进制数据会原封不动的放进MessagePack中，当需要序列化的对象二进制数据远多于其他数据时，MessagePack的序列化和反序列化速度大幅度高于JSON(2~5倍)。
   2. MessagePack对字符流数据不太友好，特别是经过编码的字符，对此，JSON的效率明显更高。当需要序列化的对象中字符流数据远多于其他数据时，JSON的速度通常是MessagePack的1~2倍。（JSON中二进制通过Base64转化）
   3. MessagePack对大量数据比较友好，当数据量比较大时，通常超过1M，MessagePack的解析速度要比JSON稍快。
   4. JSON对小数据量比较友好，当数据量比较小时，JSON的解析速度稍快。