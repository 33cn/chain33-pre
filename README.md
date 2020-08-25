# chain33-pre

chain33联盟链代理重加密服务节点，用于重加密密钥管理和隐私数据重加密。 chain33-pre节点通过代理重加密算法对授权人的重加密密钥做转换，生成对应被授权人能解密的密钥，实现隐私数据的分享。
chain33-pre节点可单独部署，也支持部署多节点，多节点部署可使用密钥分片，降低单个节点异常或者作恶的风险。


# 运行
```shell
git clone https://github.com/33cn/chain33-pre.git

cd  chain33-pre

go build -o chain33-pre main.go

./chain33-pre -f chain33.pre.toml
```

# 使用方法
结合chain33-sdk

#### 授权人
```java
// chain33-pre节点
static RpcClient[]  preClient = new RpcClient[]{
        new RpcClient("http://127.0.0.1:11801"),
        new RpcClient("http://127.0.0.1:11802"),
        new RpcClient("http://127.0.0.1:11803"),
};

// 生成对称秘钥
EncryptKey encryptKey = sdk.GenerateEncryptKey(alicePub);

// 生成重加密密钥分片
KeyFrag[] kFrags = new KeyFrag[numSplit];
try {
    kFrags = sdk.GenerateKeyFragments(alicePriv), bobPub, numSplit, threshold);
} catch (Exception e) {
    e.printStackTrace();
}

// 密钥分片发送到代理节点
String dhProof = sdk.ECDH(ServerPub, alicePriv);
for(int i = 0; i < preClient.length; i++) {
    boolean result = preClient[i].sendKeyFragment(alicePub, bobPub, encryptKey.getPubProofR(),
            encryptKey.getPubProofU(), 100, dhProof, kFrags[i]);
    if (!result) {
        System.out.println("sendKeyFragment failed");
        return;
    }
}

// 数据加密
byte[] iv = sdk.generateIv();
byte[] cipher = sdk.encrypt(content, encryptKey.getShareKey(), iv);
```

#### 被授权人
```java
// 申请重加密
ReKeyFrag[] reKeyFrags = new ReKeyFrag[threshold];
for(int i = 0; i < threshold; i++) {
    reKeyFrags[i] = preClient[i].reencrypt(alicePub, bobPub);
}

// 解密对称密钥，需要被授权人私钥
byte[] shareKeyBob;
try {
    shareKeyBob = sdk.AssembleReencryptFragment(bobPriv, reKeyFrags);
} catch (Exception e) {
    e.printStackTrace();
    return;
}

// 解密
String text = AesUtil.decrypt(cipher, HexUtil.toHexString(shareKeyBob));
```

# 接口说明

## 1. 密钥分片分发

#### CollectFragment

> 授权人分发重加密秘钥分片


**函数原型**

```go
func CollectFragment(fragments *common.ReqSendKeyFragment, result *interface{}) error 
```

**请求参数**

|参数|类型|是否必填|说明|
|----|----|----|----|
|pubOwner|string|是|数据授权人公钥|
|pubRecipient|string|是|数据被授权人公钥|
|pubProofR|string|是|重加密随机公钥R|
|pubProofU|string|是|重加密随机公钥U|
|expire|int64|是|有效时间|
|dhProof|string|是|身份证明|
|Random|string|是|秘钥分片随机数|
|Value|string|是|秘钥分片值|
|PrecurPub|string|是|重加密预置公钥|

**返回字段**

|返回字段|字段类型|说明|
|----|----|----|
|result|bool|返回值|

## 2. 申请重加密

#### Reencrypt

> 被授权人申请重加密

**函数原型**

```go
func Reencrypt(req *common.ReqReeencryptParam, result *interface{}) error
```

**请求参数**

|参数|类型|是否必填|说明|
|----|----|----|----|
|pubOwner|string|是|数据授权人公钥|
|pubRecipient|string|是|数据被授权人公钥|

**返回字段**

|返回字段|字段类型|说明|
|----|----|----|
|reKeyR|string|重加密秘钥R|
|reKeyU|string|重加密秘钥U|
|random|string|分片随机数|
|precurPub|string|重加密预置公钥|