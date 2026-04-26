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

### APK 关键源文件

| 文件 | 用途 |
|------|------|
| `network/Network.java` (L62-73) | 签名拦截器实现 |
| `WkUtil.java` | MD5 签名算法 |
| `network/api/MxAPI.java` | 完整 REST API 接口 |
| `network/api/ali/AliAPI.java` (L439-446) | 设备控制 setDeviceProperties |
| `model/device/air/AirPropertiesModel.java` | 空气净化器属性模型 |
| `model/device/humidity/HumidityPropertiesModel.java` | 加湿器属性模型 |
| `model/device/purifier/PurifierPropertiesModel.java` | 净水器属性模型 |
| `constants/DeviceConstant.java` | 设备常量定义（WorkMode 等） |
| `ui/device/adapter/DeviceListAdapter.java` | 风速档位数组定义 |

---

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

---

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

---

## 4. 设备属性（从 APK 反编译）

### 空气净化器 (categoryKey: AirPurifier)

**传感器属性:**

| 属性键 | 类型 | 说明 |
|--------|------|------|
| PM25 | IntegerModel | PM2.5 浓度 (μg/m³) |
| TVOC | IntegerModel | 挥发性有机化合物 (μg/m³)，开机预热时为 65535 |
| HCHO | IntegerModel | 甲醛浓度 (μg/m³)，开机预热时为 65535 |
| CO2 | IntegerModel | 二氧化碳浓度 (ppm) |
| CurrentTemperature | DoubleModel | 温度 (°C) |
| RelativeHumidity | DoubleModel | 湿度 (%) |
| FilterLifeTimePercent_1 | IntegerModel | 滤芯 1 寿命 (%) |
| FilterLifeTimePercent_2 | IntegerModel | 滤芯 2 寿命 (%) |
| FilterLifeTimePercent_3 | IntegerModel | 滤芯 3 寿命 (%) |
| WiFI_RSSI | IntegerModel | WiFi 信号强度 (dBm) |

**控制属性:**

| 属性键 | 类型 | 值域 | 说明 |
|--------|------|------|------|
| PowerSwitch | IntegerModel | 0/1 | 电源开关 |
| WorkMode | IntegerModel | 1=自动, 2=手动, 3=睡眠 | 工作模式 |
| WindSpeed / windspeed | IntegerModel | 0-6 | 风速档位（不同设备用不同属性名） |
| ChildLockSwitch | IntegerModel | 0/1 | 童锁 |
| ScreenSwitch | IntegerModel | 0/1 | 屏幕开关 |
| IonsSwitch | IntegerModel | 0/1 | 负离子 |
| SmartModeSwitch | IntegerModel | 0/1 | 智能模式 |

**风速档位对应 CADR (m³/h)，按设备型号不同:**

| 型号 | 档位 0-6 对应 CADR |
|------|-----|
| Filter Type 0 | 0, 140, 240, 360, 500, 610, 760 |
| Filter Type 1 | 0, 60, 140, 220, 330, 390, 500 |
| Filter Type 2 | 0, 130, 220, 330, 430, 500, 640 |
| X50 | 0, 150, 220, 300, 400, 510, 600 |
| X60 | 0, 120, 170, 240, 330, 430, 540 |

**WorkMode 完整定义（来自 DeviceConstant.java）:**

| 值 | 常量名 | 说明 |
|----|--------|------|
| 1 | DEVICE_WORK_MODE_AUTO | 自动模式 |
| 2 | AIR_WORK_MODE_SPEED | 手动/风速模式 |
| 3 | DEVICE_WORK_MODE_SLEEP | 睡眠模式 |
| 4 | DEVICE_WORK_MODE_HANDLE | Handle 模式 |
| 5 | DEVICE_WORK_MODE_WIND | Wind 模式 |

### 净水器 (categoryKey: WaterPurifier)

| 属性键 | 类型 | 说明 |
|--------|------|------|
| FinishedWaterTDS | IntegerModel | 出水 TDS (ppm) |
| RawWaterTDS | IntegerModel | 进水 TDS (ppm) |
| WaterTemperature | DoubleModel | 水温 (°C) |
| TotalPureWater | IntegerModel | 累计净水量 (mL) |
| FilterLifeTimePercent_1 | IntegerModel | 滤芯 1 寿命 (%) |
| FilterLifeTimePercent_2 | IntegerModel | 滤芯 2 寿命 (%) |
| ChildLockSwitch | IntegerModel | 童锁 |

### 加湿器 (categoryKey: Humidifier)

| 属性键 | 类型 | 说明 |
|--------|------|------|
| PM25 | IntegerModel | PM2.5 浓度 |
| CurrentTemperature | DoubleModel | 温度 |
| RelativeHumidity | DoubleModel | 湿度 |
| SetHumidity | IntegerModel | 设定湿度 |
| WaterShortage | IntegerModel | 缺水指示 |
| surplusWater | IntegerModel | 剩余水量 |
| FilterLifeTimePercent_1 | IntegerModel | 滤芯寿命 |
| PowerSwitch | IntegerModel | 电源 |
| ChildLockSwitch | IntegerModel | 童锁 |
| ScreenSwitch | IntegerModel | 屏幕 |
| SmartModeSwitch | IntegerModel | 智能模式 |

### 属性值格式

API 返回的属性值包裹在 Model 对象中：

```json
{
  "PM25": {"value": 45},
  "PowerSwitch": {"value": 1},
  "CurrentTemperature": {"value": 25.5}
}
```

65535 (0xFFFF) 表示传感器预热中，应视为无效值。

---

## 5. MQTT 实时推送调研

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

### MQTT 连接参数

```
Broker: {productKey}.iot-as-mqtt.cn-shanghai.aliyuncs.com:1883
ClientId: {clientId}|securemode=2,signmethod=hmacsha1,timestamp={ts}|
Username: {deviceName}&{productKey}
Password: HMAC-SHA1(deviceSecret, "clientId{cid}deviceName{dn}productKey{pk}timestamp{ts}")
```

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
- SDK: `com.taobao.accs.ACCSClient`（淘宝闭源 ACCS 客户端）
- 需要 EMAS 设备注册的 deviceId
- 链路: 设备上报属性(MQTT) → IoT 平台 → ACCS 推送 → App
- 结论: **无法用 Python 复刻**

### 实时推送结论

352 App 的实时属性推送走的是 **ACCS（Alibaba Cloud Channel Service）**，基于 SPDY/HTTP2 的私有长连接协议。MQTT 网关模式仅用于设备拓扑管理，不负责属性推送路由。当前方案采用 **10 秒轮询**。

---

## 6. HomeKit 兼容性

### HomeKit Fan 服务特征

| 特征 | 支持 | 说明 |
|------|------|------|
| Active | ✅ | 开关 |
| RotationSpeed | ✅ | 风速百分比 |
| TargetFanState | 部分 | 仅 Auto(0) / Manual(1)，无 Sleep |
| SwingMode | - | 352 设备不支持摆头 |

HA HomeKit Bridge 将 fan 的 `preset_mode` 拆成独立 Switch 暴露给 HomeKit（因 HomeKit Fan 无原生 preset mode 概念）。

### HomeKit 传感器映射

| HA Sensor | HomeKit | 能否显示 |
|-----------|---------|---------|
| PM2.5 (device_class: pm25) | AirQualitySensor.PM2_5Density | ✅ |
| TVOC (device_class: voc) | AirQualitySensor.VOCDensity | ✅ |
| CO2 (device_class: co2) | CarbonDioxideSensor | ✅ |
| 温度 (device_class: temperature) | TemperatureSensor | ✅ |
| 湿度 (device_class: humidity) | HumiditySensor | ✅ |
| 甲醛 HCHO | 无对应 | ❌ HomeKit 无甲醛特征 |
| TDS | 无对应 | ❌ HomeKit 无水质特征 |
| 滤芯寿命 | FilterMaintenance (linked) | ⚠️ 需挂在 AirPurifier 主服务下 |

### HomeKit 不支持的设备

- **净水器**: HomeKit 无 Water Purifier 设备类型，TDS 等水质指标无对应特征
- 滤芯寿命理论上可通过 FilterMaintenance 服务暴露，但需要作为 AirPurifier/Humidifier 的 linked service

---

## 7. 局域网协议线索（待调研）

APK 中发现 ALCS/CoAP 相关代码：

- `AlcsAuthHttpRequest` — 局域网认证
- `AlcsCoAP` — CoAP 协议实现
- `LocalDeviceMgr` — 局域网设备发现
- `/thing/lan/prefix/get` — 获取局域网通信前缀

如果 352 设备支持局域网 CoAP 协议，可以：
- 在同一网络直接通信，毫秒级延迟
- 完全绕过云端
- 实现真正的实时属性推送

### 相关 APK 源文件

- `com/aliyun/alink/linksdk/cmp/manager/connect/auth/alcs/AlcsAuthHttpRequest.java`
- `com/aliyun/alink/linksdk/alcs/` — CoAP 协议实现目录
- `LocalDeviceMgr` — 局域网设备管理

---

## 8. 踩坑记录

### loginbyoauth 返回 400

缺少 `riskControlInfo` 中的必填字段（`country`、`appID`、`signType`）。通过对比 Surge 抓包的 Android 真机流量与我们的请求，补全所有缺失字段后解决。

### iOS vs Android AppKey

iOS 使用 AppKey `27549861`（AppSecret 未知），Android 使用 `27554844`。352 服务端只接受注册对应 AppKey 的 loginbyoauth 请求。最终使用 Android AppKey + AppSecret。

### 65535 传感器值

HCHO 和 TVOC 在设备刚开机预热时返回 `0xFFFF`(65535)。在 sensor 实体中过滤为 `None`。

### HA 外部依赖安装失败

`alibabacloud-iot-api-gateway` 包无法在 HA OS 上安装。解决方案：完全移除外部依赖，所有签名算法（MD5、HMAC-SHA1）仅用 Python 标准库实现。

### iotToken 失效

在外部测试登录后，HA 的 iotToken 失效导致持续报错。通过添加自动重认证逻辑解决：检测错误码（401, 2001, 2002, 2459, 26101, 26102）和关键词（identity, token, session），清除 token 后重新走完整认证链。

### 开关状态闪烁

切换开关后，状态会短暂回退到旧值，等下次轮询才更新。通过实现乐观状态更新解决：发送命令后立即更新本地 coordinator 数据并调用 `async_write_ha_state()`。

---

## 9. Surge 抓包方法

### iOS
- Surge for Mac 开启 MITM + Replica
- iPhone 设置代理到 Mac IP:6152
- MITM hostname 添加 `*.352air.com`, `api.link.aliyun.com`, `living-account.cn-shanghai.aliyuncs.com`

### Android
- 需要修改 APK 的 `network_security_config.xml` 信任用户证书（targetSdkVersion=34 默认不信任）
- apktool 解包 → 添加 `<certificates src="user" />` → 重打包签名
- adb 设置全局代理: `adb shell settings put global http_proxy <mac-ip>:6152`
- 清除代理: `adb shell settings put global http_proxy :0`

### 抓包注意事项
- apktool 重打包时可能遇到 `layout_gravity="0x0"` 报错，需改为 `"center"`
- Android 14+ 默认不信任用户证书，必须修改 `network_security_config.xml`
- Surge MITM 会干扰 Python 的 HTTPS 请求（SSL 证书验证失败）
