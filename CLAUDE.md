# CLAUDE.md

## 项目概述

352 Air 的 Home Assistant 自定义集成。通过逆向 352Life Android APK 和 Surge 抓包，完整复现了 352 → 阿里云 IoT 的认证链路。

## 认证链路（6步）

1. `POST app.352air.com/api/v1/enduser/login` → access_token（352 签名: MD5(APPID+path+ts)）
2. `POST api.link.aliyun.com/living/account/region/get` → 获取区域端点（阿里云 API Gateway 签名）
3. `POST living-account.cn-shanghai.aliyuncs.com/api/prd/loginbyoauth.json` → sid（特殊签名：form-encoded body，需 date 头，不能有 content-md5，x-ca-signature-method 参与签名）
4. `POST api.link.aliyun.com/account/createSessionByAuthCode` → iotToken（20小时有效）
5. `POST api.link.aliyun.com/uc/listBindingByAccount` → 设备列表
6. `POST api.link.aliyun.com/thing/properties/get|set` → 读取/控制设备

## 阿里云 API Gateway 签名要点

### JSON 请求（api.link.aliyun.com）
```
StringToSign = POST\nAccept\nContent-MD5\nContent-Type\n\n{sorted x-ca-* headers}\n{path}
Signature = Base64(HMAC-SHA1(AppSecret, StringToSign))
```

### Form 请求（loginbyoauth）
与 JSON 请求的关键差异：
- **必须**有 `date` 头，参与签名
- **不能**有 `content-md5` 头
- `x-ca-signature-method` **必须**在签名头列表中
- form body URL-decoded 后拼到 path 后作为 query string 参与签名
- 需要 `ca_version: 1` 头（不是 `x-ca-version`）

### 凭证
- 352 APPID: `8d5018f2bc0f11ea8e6388e9fe5ac5b6`
- 阿里云 AppKey: `27554844`
- 阿里云 AppSecret: `b66d2c9767cd15a7c5a088341055d134`

## 目录结构

```
custom_components/air352/
├── __init__.py          # 集成入口
├── api.py               # API 客户端（352 + 阿里云 IoT，零外部依赖）
├── config_flow.py       # UI 配置流
├── const.py             # 常量
├── coordinator.py       # DataUpdateCoordinator
├── manifest.json        # 集成元数据
├── sensor.py            # 传感器实体
├── switch.py            # 开关实体
├── strings.json         # 英文字符串
└── translations/
    ├── en.json           # 英文翻译
    └── zh-Hans.json      # 中文翻译
```

## 开发注意

- 零外部依赖，所有签名手动实现
- `loginbyoauth` 的 `riskControlInfo` 必须包含 `country`、`appID`、`signType` 等字段，否则返回 400
- iotToken 有效期 72000 秒（20小时），过期前 1 小时自动重新认证
- 部署到 HA: `scp -r custom_components/air352 root@<ha-host>:/root/homeassistant/custom_components/`
