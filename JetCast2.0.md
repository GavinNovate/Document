# JetCast2.0设计方案

JetCast2.0协议用于局域网内多个设备间的通讯。

协议使用"请求-响应"模式，拆分逻辑，利于扩展。

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

协议规定了UDP和TCP数据包格式和数据包中的数据内容格式

### 数据包格式

UDP数据包格式

```
Protocol-Version(2 Bytes):协议版本
Context-Code(8 Bytes):会话编码	一对请求响应使用同一个会话编码
Package-Type(2 Bytes):包装类型	01:Request	02:Response
Encrypt-Type(2 Bytes):加密方式	00:None	01:RSA	02:AES
Content-Type(2 Bytes):数据类型	01:Byte 02:Json
Data(~ Bytes):数据内容
```

TCP数据包格式

```
Protocol-Version(2 Bytes):协议版本
Context-Code(8 Bytes):会话编码	一对请求响应使用同一个会话编码
Package-Type(2 Bytes):包装类型
Encrypt-Type(2 Bytes):加密方式
Content-Type(2 Bytes):数据类型
Content-Length(4 Bytes)：数据长度
Data(~ Bytes)：数据内容
```

说明：

1. 协议版本：描述当前协议版本，预留协议升级的能力
2. 会话编码：描述数据包的对应关系，应对并发场景
3. 包装类型：描述数据包是主动发出还是响应请求被动发出
4. 加密方式：描述数据内容的加密方式，分别为不加密，RSA加密和AES加密
5. 数据类型：描述数据内容的数据类型，分别为Byte字节流和Json字符流
6. 数据长度：TCP包中，确认数据内容的长度，用以切割数据包
7. 数据内容：具体数据

### 数据内容格式

协议规定了Byte和Json数据类型的通用格式

#### Byte类型

// Todo



#### Json类型

##### UDP包

Request

```json
{
  "Action":"POST",			// 请求类型 分为 GET/PUT/POST/DELETE
  "Resource":"Resource",	// 请求资源名字
  "ResourceId":1,			// 资源编号
  "Data":{					// 请求体
    
  },
  "Port":10000				// 回应端口 - 对方应该回应我的端口
}
```

Response

```json
{
  "Status":200,				// 响应状态码
  "Error":"Error Message",	// 响应信息
  "Data":{					// 响应体
    
  },
  "Port":10000,				// 回应端口 - 对方应该回应我的端口
}
```

##### TCP包

Request

```json
{
  "Action":"POST",			// 请求类型 分为 GET/PUT/POST/DELETE
  "Resource":"Resource",	// 请求资源类型
  "ResourceId":1,			// 资源编号
  "Data":{					// 请求体
    
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

关于四种请求方式

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