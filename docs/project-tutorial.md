# simple_saas_cn 项目入门教程

这份教程按这个项目本身来讲 Next.js，不追求一次学完整个框架。目标是让你能看懂项目结构、理解登录和微信支付链路、知道构建部署时发生了什么，并能定位常见问题。

## 1. 这个项目是什么

`simple_saas_cn` 是一个基于 Next.js 的 SaaS 模板，主要功能包括：

- 首页和价格页
- 用户注册、登录、退出、找回密码
- Supabase 用户认证和数据库
- 微信支付 Native 扫码支付
- 支付成功 webhook 回调
- 订阅和积分系统
- Coolify / Docker 部署

它不是纯前端网站。Next.js 同时承担了页面渲染和后端接口的职责。

## 2. 先认识几个命令

项目命令在 `package.json` 里：

```bash
npm run dev
npm run build
npm run start
```

它们的作用分别是：

- `npm run dev`：本地开发模式，代码改了会自动刷新。
- `npm run build`：生产构建，把 TypeScript、React 页面、API 路由编译成生产可运行的文件。
- `npm run start`：生产运行，一般在服务器或容器里使用。

Coolify 部署时，本质上就是：

```text
拉取 Git 仓库
-> 安装依赖
-> npm run build
-> 启动 Next.js 服务
-> 用域名代理到容器内部端口 3000
```

## 3. 项目目录结构

常用目录如下：

```text
app/                 Next.js 页面、布局、后端 API
components/          页面里复用的 UI 组件
config/              订阅套餐、积分套餐配置
hooks/               浏览器端 React hooks
lib/                 第三方服务封装，比如微信支付
utils/               工具函数
utils/supabase/      Supabase 客户端、服务端、管理员权限封装
types/               TypeScript 类型定义
supabase/            数据库初始化 SQL
Dockerfile           Coolify 容器构建配置
```

最重要的是 `app/`。Next.js App Router 约定：

```text
app/page.tsx                 首页
app/layout.tsx               全局布局
app/dashboard/page.tsx       用户后台页面
app/api/xxx/route.ts         后端 API 接口
app/auth/callback/route.ts   Supabase 登录回调
```

## 4. 页面是怎么工作的

首页文件：

```text
app/page.tsx
```

用户后台：

```text
app/dashboard/page.tsx
```

全局布局：

```text
app/layout.tsx
```

`layout.tsx` 会包住所有页面。这个项目里它做了几件事：

- 设置网站标题和描述
- 读取 `BASE_URL`
- 初始化 Supabase
- 查询当前用户
- 渲染 Header、Footer、Toast

`dashboard/page.tsx` 是服务端页面。它会：

1. 创建 Supabase 服务端客户端。
2. 获取当前登录用户。
3. 如果没登录，跳转到 `/sign-in`。
4. 从 `customers` 表查询积分、订阅和积分历史。
5. 把数据传给 dashboard 组件显示。

所以访问：

```text
https://你的域名/dashboard
```

对应的核心文件就是：

```text
app/dashboard/page.tsx
```

## 5. 组件是什么

组件放在 `components/` 目录。

比如：

```text
components/header.tsx
components/footer.tsx
components/pricing-section.tsx
components/dashboard/credits-balance-card.tsx
components/dashboard/subscription-status-card.tsx
```

你可以简单理解为：

- `app/` 负责路由和页面。
- `components/` 负责页面里的局部 UI。
- `lib/` 和 `utils/` 负责业务逻辑和工具函数。

如果你想改页面长什么样，大概率去 `components/` 或 `app/.../page.tsx`。

如果你想改支付、数据库、登录逻辑，大概率去 `app/api/`、`lib/` 或 `utils/`。

## 6. 登录注册链路

登录注册的核心文件：

```text
app/actions.ts
```

这里有几个函数：

```text
signUpAction
signInAction
forgotPasswordAction
resetPasswordAction
signOutAction
```

注册流程：

```text
用户填写邮箱和密码
-> signUpAction
-> Supabase Auth 创建用户
-> 发验证邮件
-> 用户点击邮件链接
-> /auth/callback
-> 登录完成
```

登录流程：

```text
用户填写邮箱和密码
-> signInAction
-> supabase.auth.signInWithPassword
-> 成功后跳转 /dashboard
```

登出流程：

```text
signOutAction
-> supabase.auth.signOut
-> 跳转 /sign-in
```

## 7. Supabase 客户端区别

项目里有三种 Supabase 使用方式：

```text
utils/supabase/client.ts
utils/supabase/server.ts
utils/supabase/service-role.ts
```

区别很重要：

`client.ts`：浏览器端使用。只能用公开权限，不能放密钥。

`server.ts`：服务端使用。可以读取用户 cookie，知道当前登录用户是谁。

`service-role.ts`：管理员权限。使用 `SUPABASE_SERVICE_ROLE_KEY`，可以绕过普通用户权限限制。只能在服务端使用，绝对不能暴露给浏览器。

简单判断：

```text
页面里查当前用户：server.ts
浏览器交互：client.ts
webhook 更新订单/积分：service-role.ts
```

## 8. 套餐和积分配置

套餐配置在：

```text
config/subscriptions.ts
```

里面分两类：

```text
SUBSCRIPTION_TIERS   订阅套餐
CREDITS_TIERS        积分包
```

每个套餐大概包含：

```ts
{
  name: "专业版",
  id: "tier-pro",
  amountFen: 1,
  priceDisplay: "¥49",
  description: "...",
  features: [...]
}
```

注意：`amountFen` 是支付金额，单位是分。现在项目里多个套餐都是 `1`，表示测试用的 1 分钱。

## 9. 微信支付下单链路

创建订单接口：

```text
app/api/wxpay/create-order/route.ts
```

微信支付核心封装：

```text
lib/wxpay.ts
```

流程是：

```text
用户点击购买
-> 前端请求 /api/wxpay/create-order
-> create-order 检查用户是否登录
-> 根据 planId 找套餐
-> 生成 outTradeNo
-> 调用 lib/wxpay.ts 的 createNativeOrder
-> 请求微信支付接口
-> 返回 code_url 给前端
-> 前端展示二维码
```

这里的 `attach` 很关键：

```text
userId|productType|credits|planId
```

它会传给微信支付。支付成功后，微信 webhook 会把它原样带回来，项目就知道这笔钱属于哪个用户、买的是订阅还是积分。

## 10. 微信支付 webhook 链路

webhook 文件：

```text
app/api/webhooks/wxpay/route.ts
```

微信支付成功后，会请求：

```text
https://你的域名/api/webhooks/wxpay
```

流程：

```text
微信支付发送通知
-> app/api/webhooks/wxpay/route.ts
-> decryptResource 解密通知体
-> 判断 trade_state 是否 SUCCESS
-> 从 attach 还原 userId/productType/credits/planId
-> ensureCustomer 确保客户记录存在
-> 如果是积分包，增加积分
-> 如果是订阅，创建或续期订阅
```

更新数据库的核心文件：

```text
utils/supabase/subscriptions.ts
```

里面有：

```text
ensureCustomer
createOrUpdateSubscription
addCreditsToCustomer
useCredits
```

## 11. 环境变量

环境变量本地在 `.env.local`，线上在 Coolify 的 Environment Variables。

公开变量：

```bash
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
```

服务端密钥：

```bash
SUPABASE_SERVICE_ROLE_KEY=
WXPAY_MCH_ID=
WXPAY_APP_ID=
WXPAY_PRIVATE_KEY=
WXPAY_SERIAL_NO=
WXPAY_API_V3_KEY=
WXPAY_NOTIFY_URL=
BASE_URL=
```

规则：

- `NEXT_PUBLIC_` 开头的变量会进入浏览器，可以公开。
- 不带 `NEXT_PUBLIC_` 的变量只能服务端使用。
- `SUPABASE_SERVICE_ROLE_KEY`、`WXPAY_PRIVATE_KEY`、`WXPAY_API_V3_KEY` 绝对不能暴露。

`WXPAY_PRIVATE_KEY` 建议在 Coolify 里填一行，用 `\n` 表示换行：

```text
-----BEGIN PRIVATE KEY-----\nMII...\n-----END PRIVATE KEY-----
```

项目里的代码会把 `\n` 转成真正换行。

## 12. Coolify 部署

这个项目已经有：

```text
Dockerfile
.dockerignore
next.config.ts
```

Coolify 推荐配置：

```text
Build Pack: Dockerfile
Ports Exposes: 3000
Port Mappings: 留空
```

不要填：

```text
3000:3000
```

原因是你用域名访问时，Coolify 会用反向代理转发到容器内部端口。填 Port Mappings 会占用服务器宿主机端口，容易冲突。

Dockerfile 做了这些事：

```text
使用 Node 22
安装依赖
跳过 Puppeteer 浏览器下载
执行 npm run build
复制 Next standalone 输出
启动 node server.js
```

`next.config.ts` 里的这行很重要：

```ts
output: "standalone"
```

它会让 Next.js 输出一个适合 Docker 部署的精简服务端包。

## 13. 开发时怎么定位问题

页面打不开：

```text
看 Coolify Deployments Logs
看 Coolify Runtime Logs
确认 Ports Exposes 是 3000
确认 Port Mappings 留空
```

登录失败：

```text
检查 Supabase URL Configuration
检查 NEXT_PUBLIC_SUPABASE_URL
检查 NEXT_PUBLIC_SUPABASE_ANON_KEY
检查 /auth/callback 是否能访问
```

dashboard 没数据：

```text
检查 customers 表
检查 subscriptions 表
检查 credits_history 表
检查用户是否真的登录
```

微信支付失败：

```text
检查 WXPAY_MCH_ID
检查 WXPAY_APP_ID
检查 WXPAY_PRIVATE_KEY 格式
检查 WXPAY_SERIAL_NO
检查 WXPAY_NOTIFY_URL 是否是公网 HTTPS
```

webhook 没生效：

```text
检查微信商户平台回调地址
检查 /api/webhooks/wxpay 是否能访问
检查 WXPAY_API_V3_KEY
检查 Coolify runtime logs
```

## 14. 推荐阅读顺序

第一次看代码，按这个顺序：

```text
1. package.json
2. app/layout.tsx
3. app/page.tsx
4. components/pricing-section.tsx
5. config/subscriptions.ts
6. app/actions.ts
7. utils/supabase/server.ts
8. app/dashboard/page.tsx
9. app/api/wxpay/create-order/route.ts
10. lib/wxpay.ts
11. app/api/webhooks/wxpay/route.ts
12. utils/supabase/subscriptions.ts
13. Dockerfile
```

这个顺序基本对应：

```text
项目入口
-> 页面展示
-> 套餐配置
-> 登录
-> 用户后台
-> 创建支付
-> 支付回调
-> 数据库更新
-> 部署
```

## 15. 你需要掌握到什么程度

短期目标不是成为 Next.js 专家，而是做到：

- 知道每个目录大概负责什么。
- 知道页面、组件、API route 的区别。
- 知道环境变量哪些能公开、哪些不能公开。
- 知道登录和支付请求怎么流动。
- 看到 Coolify 构建日志能判断是安装、构建还是启动阶段出错。
- 能改套餐价格、文案、页面内容和简单业务逻辑。

# 等这些熟了，再深入学 React、Next.js 服务端组件、数据库权限、支付验签和测试。
