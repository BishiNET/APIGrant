API Grant System设计

# 函数解释
1.HMAC-SHA256(Key, Message)

使用HMAC摘要方式，使用SHA256作为Hash算法，生成数字摘要

2.X25519-SharedKey(PrivateKey, PublicKey)

计算X25519 SharedKey

# 常量解释：

Kspub = 服务器X25519公钥(Server Public Key)

Kspri = 服务器X25519私钥(Server Private Key)


# 错误值(int)：

NO_ERROR = 0

PERMISSION_DENIED = 1

PARAMS_ERROR = 2

# 1.请求端：

## 1.1 生成临时X25519密钥对
设临时密钥对为

**Kcspub = 临时密钥对公钥(Client Symmetric PublicKey)**

**Kcspri = 临时密钥对私钥(Client Symmetric PrivateKey)**

设请求端密钥对为

**Krpub = 请求端密钥对公钥(Request PublicKey)**

**Krpri = 请求端密钥对私钥(Request PrivateKey)**

并**定义Ksc = HMAC-SHA256(X25519-SharedKey(Krpri, Kspub), X25519-SharedKey(Kcspri, Kspub))**

**定义请求参数Sign = HMAC-SHA256(Ksc, Target UUID || Source UUID)**

## 1.2 发送请求
### 1.2.1 请求参数：
|参数名|解释|类型|
|-|-|-|
|Target|UUID(需要访问边缘节点UUID)|String|
|Source|UUID(请求端UUID)|String|
|Sign|签名|Hex String|
|Kcspub = CSPUB| |String|

### 1.2.2 请求格式：
JSON + POST

## 1.3 接受返回值：

服务端返回参数：
|参数名|接受|类型|
|-|-|-|
|ErrCode|错误返回码|int|
|AccessID| |String|
|RemoteAddr|接收端IPv4地址|String|
|Krvpub = ReceiverPublicKey| |Hex String|
|Ksspub = ServerSymmetricPublicKey| |Hex String|
|Sign|签名|Hex String|

## 1.4 与真正接收端交互(Access Request)

### 1.4.1 验证服务端Sign合法性

**Sign = HMAC-SHA256(X25519-SharedKey(Kcspri, Ksspub), Krvpub || AccessID)**

验证通过后执行 1.4.2

### 1.4.2 向接收端发送访问请求(Access Request)
设以下变量

**Ks1 = X25519-SharedKey(Kcspri, Ksspub)**

**Ks2 = X25519-SharedKey(Krpri, Krvpub)**

**Ks3 = X25519-SharedKey(Kcspri, Krvpub)**

**Token = HMAC-SHA256(Ks1 || Ks2 || Ks3, AccessID)**


发送请求(RPC)：

需要发送如下结构体

	struct Auth {
		string AccessID;
		string Token;
	};


其他遵守RPC调用方式，RPC标准不在这里叙述


# 2.API Grant服务端(简称服务端)

## 2.1 验证(Verification)
在收到访问请求(Access Request)后，服务端需要验证其Sign是否合法，验证完毕后验证其是否有权限访问该边缘节点

### 2.1.1 Sign合法性验证

**设Ksc = HMAC-SHA256(X25519-SharedKey(Kspri, Krpub), X25519-SharedKey(Kspri, Kcspub))**


**Sign1 = HMAC-SHA256(Ksc, Target UUID || Source UUID)**

验证Sign1是否等于请求端提交的Sign

请求通过后进行 2.1.2，否则返回错误代码

### 2.1.2 生成服务端临时密钥对

设以下变量

**Ksspub = 服务端临时密钥对公钥(Server Symmetric PublicKey)**

**Ksspri = 服务端临时密钥对私钥(Server Symmetric PrivateKey)**

并返还给请求端Ksspub, Krvpub, Sign
其中

**设变量Ks = X25519-SharedKey(Ksspri, Kcspub)**

**Sign = HMAC-SHA256(Ks, Krvpub || AccessID)**


并储存Ks到Redis中


储存方式为: **Redis.SET(AccessID(Key), Ks(Value), 1800(Expire))**


## 2.2 接受接收端请求验证(Request Verification)

方式: GET

API URL: request_verify/{AccessID}/{Sign}

其中:

**Sign = HMAC-SHA256(X25519-SharedKey(Kspri, Krvpub), AccessID)**

返回值:
|参数名|接受|类型|
|-|-|-|

|Ks|即2.1.2中的Ks|String|
|Sign| |Hex String|
|Krpub = RequestPublicKey| |Hex String|
|Kcspub|即1.2.1中的Kcspub|Hex String|

其中

**Sign = HMAC-SHA256(X25519-SharedKey(Kspri, Krvpub), Kcspub || Ks || RequestPublicKey)**

# 3. 接收端

设以下变量:

**Krvpub = 接收端公钥**

**Krvpri = 接收端私钥**


## 3.1 接收端接受访问请求

执行2.2步骤，其中

**Sign = HMAC-SHA256(X25519-SharedKey(Krvpri, Kspub), AccessID)**

### 3.1.2 验证服务端Sign合法性

**Sign1 = HMAC-SHA256(X25519-SharedKey(Krvpri, Kspub), Kcspub || Ks || RequestPublicKey)**

### 3.1.3 验证请求端Token
设

**Ks1 = Ks**

**Ks2 = X25519-SharedKey(Krvpri, RequestPublicKey)**

**Ks3 = X25519-SharedKey(Krvpri, Kcspub)**

**Token1 = HMAC-SHA256(Ks1 || Ks2 || Ks3, AccessID)**

当**Token == Token1**时候才能接受RPC访问请求
