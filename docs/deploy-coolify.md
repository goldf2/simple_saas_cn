# simple_saas_cn Coolify 部署配置说明

这份文档用于把 `simple_saas_cn` 部署到 Coolify。项目已经包含 `Dockerfile`，推荐使用 Dockerfile 部署。

## 1. Git 仓库

Coolify 新建应用时选择：

```text
Repository: goldf2/simple_saas_cn
Branch: main
Build Pack: Dockerfile
```

## 2. 网络和端口

进入应用的 Configuration / Network，按下面配置：

```text
Ports Exposes: 3000
Port Mappings: 留空
Network Aliases: 可留空
```

不要填写：

```text
3000:3000
```

原因：使用域名访问时，Coolify 会通过反向代理把请求转发到容器内部端口 `3000`。填写 Port Mappings 会占用服务器宿主机端口，容易导致 `port is already allocated`。

## 3. 域名

在 Domains 中填写你的正式域名：

```text
https://你的域名
```

例如：

```text
https://saascn.ebm001.com
```

下面所有 URL 都要和这个域名保持一致。

## 4. 环境变量

在 Coolify 的 Environment Variables 中添加下面变量。

```bash
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=xxx
SUPABASE_SERVICE_ROLE_KEY=xxx

WXPAY_MCH_ID=你的微信支付商户号
WXPAY_APP_ID=你的微信应用APPID
WXPAY_PRIVATE_KEY=-----BEGIN PRIVATE KEY-----\nMII...\n-----END PRIVATE KEY-----
WXPAY_SERIAL_NO=你的商户API证书序列号
WXPAY_API_V3_KEY=你的32位APIv3密钥
WXPAY_NOTIFY_URL=https://你的域名/api/webhooks/wxpay

BASE_URL=https://你的域名
```

### Buildtime / Runtime 勾选建议

需要 Buildtime + Runtime：

```text
NEXT_PUBLIC_SUPABASE_URL
NEXT_PUBLIC_SUPABASE_ANON_KEY
BASE_URL
```

只需要 Runtime：

```text
SUPABASE_SERVICE_ROLE_KEY
WXPAY_MCH_ID
WXPAY_APP_ID
WXPAY_PRIVATE_KEY
WXPAY_SERIAL_NO
WXPAY_API_V3_KEY
WXPAY_NOTIFY_URL
```

如果不确定，可以先 Buildtime 和 Runtime 都勾上部署；等跑通后，再把密钥类变量的 Buildtime 关掉。

### WXPAY_PRIVATE_KEY 格式

`WXPAY_PRIVATE_KEY` 不要填成多行。推荐填成一行，用 `\n` 表示换行：

```text
-----BEGIN PRIVATE KEY-----\nMIIEv...\n-----END PRIVATE KEY-----
```

项目代码会自动把 `\n` 转成真实换行。

## 5. Supabase 配置

### 初始化数据库

在 Supabase SQL Editor 里执行：

```text
supabase/20250101000000_init_schema.sql
```

### 配置认证 URL

进入 Supabase：

```text
Authentication -> URL Configuration
```

Site URL：

```text
https://你的域名/
```

Redirect URLs 添加：

```text
https://你的域名/auth/callback
```

## 6. 微信支付配置

在微信支付商户平台配置支付通知地址：

```text
https://你的域名/api/webhooks/wxpay
```

这个地址必须是公网可访问的 HTTPS 地址。

微信支付相关环境变量来源：

```text
WXPAY_MCH_ID           微信支付商户号
WXPAY_APP_ID           公众号/小程序/开放平台 APPID
WXPAY_PRIVATE_KEY      商户 API 私钥
WXPAY_SERIAL_NO        商户 API 证书序列号
WXPAY_API_V3_KEY       API v3 密钥，32 位
WXPAY_NOTIFY_URL       支付回调通知地址
```

## 7. 部署

保存配置后，点击：

```text
Deploy
```

正常部署日志大致会包含：

```text
npm install --legacy-peer-deps
npm run build
node server.js
```

部署成功后访问你的域名。

## 8. 部署后测试

按顺序测试：

```text
1. 打开首页
2. 注册账号
3. 登录
4. 进入 dashboard
5. 点击购买套餐或积分
6. 是否生成微信支付二维码
7. 支付成功后，检查积分或订阅是否更新
```

## 9. 常见问题

### port is already allocated

原因通常是填写了 Port Mappings，例如：

```text
3000:3000
```

解决：

```text
Ports Exposes: 3000
Port Mappings: 留空
```

### 登录后跳转失败

检查 Supabase：

```text
Site URL: https://你的域名/
Redirect URL: https://你的域名/auth/callback
```

### 微信支付创建订单失败

检查：

```text
WXPAY_MCH_ID
WXPAY_APP_ID
WXPAY_PRIVATE_KEY
WXPAY_SERIAL_NO
WXPAY_NOTIFY_URL
```

尤其是 `WXPAY_PRIVATE_KEY` 是否保留了：

```text
-----BEGIN PRIVATE KEY-----
-----END PRIVATE KEY-----
```

### webhook 不更新积分或订阅

检查：

```text
微信支付商户平台回调地址
WXPAY_API_V3_KEY
Coolify Runtime Logs
Supabase customers/subscriptions/credits_history 表
```

### 改了环境变量但没有生效

修改环境变量后需要重新部署：

```text
Save -> Redeploy
```

`NEXT_PUBLIC_` 开头的变量和 `BASE_URL` 会参与构建，必须重新部署才会生效。
