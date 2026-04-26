# 352 Air 逆向工程调研记录

## 1. APK 反编译

- 工具: jadx
- APK: 352Life_1.7.1_APKPure.apk (165MB)
- 主包名: `com.mxchip.project352`

### 关键凭证

| 项目 | 值 |
|------|------|
| 352 APPID | `8d5018f2bc0f11ea8e6388e9fe5ac5b6` |
| 352 API Base URL | `https://app.352air.com` |
| 阿里云 Android AppKey | `27554844` |
| 阿里云 Android AppSecret | `b66d2c9767cd15a7c5a088341055d134` |
| 阿里云 iOS AppKey | `27549861`（AppSecret 未知，与 Android 不同） |
| 阿里云 API Gateway | `api.link.aliyun.com` |
| 阿里云 Open Account | `living-account.cn-shanghai.aliyuncs.com` |
| MQTT Broker | `{productKey}.iot-as-mqtt.cn-shanghai.aliyuncs.com:1883` |

## 2. 完整认证链路（6步）

### Step 1: 352 登录

```
POST https://app.352air.com/api/v1/enduser/login
签名: sign = MD5(APPID + path + timestamp)
Headers: Authorization: Token, ts, sign
Body: {"account": "phone", "password": "pwd"}
Response: access_token, refresh_token, air_token, expires_at
```

### Step 2: 获取区域

```
POST https://api.link.aliyun.com/living/account/region/get
签名: 阿里云 API Gateway (HMAC-SHA1)
Body: {"type": "THIRD_AUTHCODE", "countryCode": "CN", "authCode": "<access_token>"}
Response: oaApiGatewayEndpoint, mqttEndpoint, apiGatewayEndpoint
```

### Step 3: Open Account OAuth 登录

```
POST https://living-account.cn-shanghai.aliyuncs.com/api/prd/loginbyoauth.json
Content-Type: application/x-www-form-urlencoded; charset=utf-8
Body: loginByOauthRequest=<url-encoded-json>
```

**签名要点（与标准 API Gateway 签名的关键差异）：**
- 必须有 `date` 头参与签名
- 不能有 `content-md5` 头
- `x-ca-signature-method` 必须在签名头列表中
- form body URL-decoded 后拼到 path 作为 query string 参与签名
- 需要 `ca_version: 1` 头（不是 `x-ca-version`）

**loginByOauthRequest JSON 必须包含的字段：**
```json
{
  "country": "CN",
  "authCode": "<352_access_token>",
  "oauthPlateform": 23,
  "oauthAppKey": "27554844",
  "riskControlInfo": {
    "appVersion": "1070001",
    "USE_OA_PWD_ENCRYPT": "true",
    "utdid": "ffffffffffffffffffffffff",
    "netType": "wifi",
    "umidToken": "",
    "locale": "zh_CN",
    "appVersionName": "1.7.1",
    "deviceId": "<uuid>",
    "routerMac": "02:00:00:00:00:00",
    "platformVersion": "36",
    "appAuthToken": "",
    "appID": "com.mxchip.project352",
    "signType": "RSA",
    "sdkVersion": "3.4.2",
    "model": "HA",
    "USE_H5_NC": "true",
    "platformName": "android",
    "brand": "HomeAssistant",
    "yunOSId": ""
  }
}
```

缺少 `country`、`appID`、`signType` 等字段会返回 400 空 body。

**Response:** `sid`（用于下一步的 authCode）

### Step 4: 创建 IoT Session

```
POST https://api.link.aliyun.com/account/createSessionByAuthCode
Body: {"request": {"authCode": "<sid>", "accountType": "OA_SESSION", "appKey": "27554844"}}
Response: iotToken (20小时有效), refreshToken, identityId
```

### Step 5: 获取设备列表

```
POST https://api.link.aliyun.com/uc/listBindingByAccount
Body: {"pageNo": 1, "pageSize": 50}, iotToken in request
Response: 设备列表 [{iotId, productName, categoryKey, status, ...}]
```

### Step 6: 设备属性读写

```
POST https://api.link.aliyun.com/thing/properties/get   # 读取
POST https://api.link.aliyun.com/thing/properties/set   # 控制
Body: {"iotId": "xxx", "items": {"PowerSwitch": 1}}
```

## 3. 阿里云 API Gateway 签名算法

### JSON 请求（api.link.aliyun.com）

```
StringToSign = POST\nAccept\nContent-MD5\nContent-Type\n\n{sorted x-ca-* headers key:value\n}\n{path}
Signature = Base64(HMAC-SHA1(AppSecret, StringToSign))
Headers: x-ca-key, x-ca-nonce, x-ca-timestamp, x-ca-signature, x-ca-signature-headers, content-md5
```

### Form 请求（loginbyoauth）

```
StringToSign = POST\nAccept\n\nContent-Type\nDate\n{sorted x-ca-* headers}\n{path}?{form-params-url-decoded}
```

签名头列表必须包含 `x-ca-signature-method`，不包含 `content-md5`。

### aepauth 签名（MQTT 虚拟设备凭证获取）

```
sign = HMAC-SHA1(AppSecret, "appKey{AK}clientId{cid}deviceSn{sn}timestamp{ts}")
```

## 4. MQTT 实时推送调研

### 已验证可行

| 步骤 | 结果 |
|------|------|
| aepauth → 虚拟设备凭证 | ✅ 200 |
| MQTT 连接 | ✅ 成功 |
| thing/authen/sub/register | ✅ 200，返回设备 deviceSecret |
| thing/topo/add (HMAC-SHA1) | ✅ 200 |
| _LivingLink.activation.subdevice.connect | ✅ bizCode 200 |
| combine/login (网关 topic) | ✅ 200，但触发 427 |

### 已验证不可行

| 步骤 | 结果 | 原因 |
|------|------|------|
| combine/login | 427 "device connect in elsewhere" | 设备自己直连 MQTT，会被抢占 |
| /app/down/thing/properties 推送 | 无消息 | 属性推送走 ACCS 而非 MQTT |

### LivingLink 子设备连接协议

Topic: `/sys/{gwPK}/{gwDN}/_thing/service/post`

```json
{
  "id": "<uuid>",
  "version": "1.0",
  "method": "_thing.service.post",
  "params": {
    "identifier": "_LivingLink.activation.subdevice.connect",
    "serviceParams": {
      "requestId": "<uuid>",
      "version": "2.0",
      "DeviceList": [{
        "clientId": "<cid>",
        "deviceName": "<subDN>",
        "productKey": "<subPK>",
        "sign": "SHA-256(clientId{cid}deviceName{subDN}deviceSecret{gatewayDS}productKey{subPK})",
        "signMethod": "sha256",
        "subSign": "SHA-256(clientId{cid}deviceName{subDN}deviceSecret{subDS}productKey{subPK})",
        "cleanSession": "true"
      }]
    }
  }
}
```

响应在 `_thing/event/notify` topic。

### ACCS 推送通道

- 端点: `msgacs.cn-zhangjiakou.aliyuncs.com:443`, `living-accs.ap-southeast-1.aliyuncs.com:443`
- 协议: **SPDY/HTTP2**（不是 WebSocket）
- SDK: `com.taobao.accs.ACCSClient`（闭源）
- 链路: 设备上报属性(MQTT) → IoT 平台 → ACCS 推送 → App
- 结论: 无法用 Python 复刻

## 5. 局域网协议线索（待调研）

APK 中发现 ALCS/CoAP 相关代码：

- `AlcsAuthHttpRequest` — 局域网认证
- `AlcsCoAP` — CoAP 协议实现
- `LocalDeviceMgr` — 局域网设备发现
- `/thing/lan/prefix/get` — 获取局域网通信前缀

如果 352 设备支持局域网 CoAP 协议，可以：
- 在同一网络直接通信，毫秒级延迟
- 完全绕过云端
- 实现真正的实时属性推送

## 6. Surge 抓包方法

### iOS
- Surge for Mac 开启 MITM + Replica
- iPhone 设置代理到 Mac IP:6152
- MITM hostname 添加 `*.352air.com`, `api.link.aliyun.com`, `living-account.cn-shanghai.aliyuncs.com`

### Android
- 需要修改 APK 的 `network_security_config.xml` 信任用户证书（targetSdkVersion=34 默认不信任）
- apktool 解包 → 添加 `<certificates src="user" />` → 重打包签名
- adb 设置全局代理: `adb shell settings put global http_proxy <mac-ip>:6152`
- 清除代理: `adb shell settings put global http_proxy :0`
