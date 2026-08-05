# Provider 现状、耦合与国内适配

## 1. 当前第三方服务

| 能力 | 当前实际选择 | 仓库中预置但未启用 |
|---|---|---|
| 支付 | Stripe | Lemon Squeezy、Polar |
| 对象存储 | AWS S3 | 无其他 Provider |
| 邮件 | Dummy，只写服务端日志 | Wasp 支持 Mailgun、SendGrid、Resend、SMTP |
| AI | OpenAI SDK，`gpt-3.5-turbo` | 无其他模型 Provider |
| 认证 | Wasp Email Auth | Google、GitHub、Discord 代码已预置但注释 |
| 分析 | 服务端 Job 当前导入 Plausible | Google Analytics 文件已存在但需手动切换 |

这里的“当前”是代码默认选择，不表示实际账号、密钥、网络和回调已经配置成功。

## 2. 已有较好抽象

### Wasp Operations

Action、Query、API 和 Job 有统一的声明、类型生成、Session 和 Entity 注入机制。React 不需要手写普通业务 HTTP 客户端。

### 支付 Provider 接口

[`src/payment/paymentProcessor.ts`](../../template/app/src/payment/paymentProcessor.ts) 定义 `PaymentProcessor`，统一：

- 创建 Checkout Session；
- 获取 Customer Portal URL；
- Webhook Handler；
- Webhook Middleware；
- 汇总收入。

[`paymentProcessorPlans.ts`](../../template/app/src/payment/paymentProcessorPlans.ts) 进一步把内部套餐 ID 映射到第三方商品/价格 ID。

### 邮件

业务代码调用 `wasp/server/email` 的 `emailSender.send`。Provider 选择集中在 [`src/server/emailSender.wasp.ts`](../../template/app/src/server/emailSender.wasp.ts)，当前是 Dummy。

Wasp 0.25 内置 Dummy、Mailgun、SendGrid、Resend、SMTP：<https://wasp.sh/docs/advanced/email>。

### 参数和结构化输出

业务输入及 AI 返回值普遍使用 Zod Schema，能够把非法数据隔离在业务边界。

## 3. 主要耦合点

### 3.1 AI 直接绑定 OpenAI

[`src/demo-ai-app/operations.ts`](../../template/app/src/demo-ai-app/operations.ts) 同时负责：

- `new OpenAI(...)`；
- 读取 `OPENAI_API_KEY`；
- 固定模型名；
- 构造 Chat Completions messages；
- function tool 声明；
- 解析 OpenAI tool calls；
- 认证、额度和数据库事务。

页面错误文本也直接出现 OpenAI 名称。这使得替换国内模型时会触碰业务 Action 和 UI，而不仅是基础设施 Adapter。

### 3.2 存储接口和数据模型均带 S3 语义

[`src/file-upload/s3Utils.ts`](../../template/app/src/file-upload/s3Utils.ts) 直接使用 AWS SDK。下列名字已经扩散到服务端、前端和数据库：

- `s3Key`
- `s3UploadUrl`
- `s3UploadFields`
- `getUploadFileSignedURLFromS3`
- `AWS_S3_*`

因此接入 OSS/COS 不只是替换 SDK，还需要逐步中性化应用层契约。

### 3.3 支付抽象包含海外订阅产品假设

现有接口假设 Provider 支持：

- Hosted Checkout；
- Customer Portal；
- Subscription Webhook；
- Provider Customer ID；
- 全量收入查询。

微信支付和支付宝更常见的交互是商户订单、预支付、二维码/JSAPI/H5/App 参数、异步通知和主动查单。直接把它们塞入现有 `PaymentProcessor` 会造成大量空实现或语义错误。

此外：

- Provider 通过源码赋值选择，不是配置驱动；
- `User` 只有一个 `paymentProcessorUserId`；
- `User` 直接包含 `lemonSqueezyCustomerPortalUrl`；
- 没有 PaymentOrder、PaymentAttempt、Refund、WebhookEvent 模型。

### 3.4 环境变量集中但未按启用 Provider 裁剪

[`src/env.ts`](../../template/app/src/env.ts) 同时合并 Stripe、Lemon Squeezy、Polar、OpenAI、S3、Plausible、Google Analytics 的 Schema。Provider 未启用时，其变量仍出现在统一启动校验中。

### 3.5 分析 Provider 没有正式接口

`src/analytics/providers/` 有 Plausible 和 Google Analytics 两套实现，但 [`src/analytics/stats.ts`](../../template/app/src/analytics/stats.ts) 通过修改 import 手动切换，没有统一类型或选择器。

## 4. 国内大模型 Provider

### 推荐目录

```text
template/app/src/ai/
├─ provider.ts
├─ capabilities.ts
├─ errors.ts
├─ usage.ts
├─ env.ts
└─ providers/
   ├─ openai.ts
   ├─ openai-compatible.ts
   ├─ dashscope.ts
   └─ other-provider.ts
```

目录名表示职责建议，不代表现在已经决定具体国内厂商。

### 推荐接口边界

第一阶段不建议立即设计覆盖所有 Agent 能力的庞大抽象。应先统一当前真正使用的能力：

- 文本或消息输入；
- 按 Zod/JSON Schema 生成结构化结果；
- Provider/模型能力声明；
- 超时、限流、内容审核、不可重试错误分类；
- Token 和计费单位返回；
- Provider request ID，便于审计。

`src/demo-ai-app/operations.ts` 只依赖领域级日程生成接口，不再解析 OpenAI tool calls。

### 可能涉及的文件

- `src/demo-ai-app/operations.ts`
- `src/demo-ai-app/schedule.ts`
- `src/demo-ai-app/DemoAppPage.tsx`
- `src/env.ts`
- `.env.server.example`
- `package.json`

## 5. 阿里云 OSS / 腾讯云 COS

### 推荐目录

```text
template/app/src/file-upload/
├─ storage/
│  ├─ provider.ts
│  ├─ selectProvider.ts
│  └─ providers/
│     ├─ s3.ts
│     ├─ oss.ts
│     └─ cos.ts
├─ operations.ts
├─ fileUploading.ts
└─ validation.ts
```

### 推荐 Provider 能力

- 创建直传凭证或预签名请求；
- 生成下载 URL；
- 检查对象是否存在；
- 删除对象；
- 返回中性的 `objectKey`、`uploadUrl`、`uploadFields`；
- 暴露 Provider 对直传方式和 Header/Form Field 的能力差异。

### 数据模型建议

逐步把 `File.s3Key` 重命名为 `objectKey`，必要时增加：

- `storageProvider`
- `bucket`
- `region`
- `etag`
- `status`

如果只允许整个部署使用一个存储 Provider，`bucket/region` 可以保留在配置而不入库；是否需要多 Provider 共存属于待验证产品决策。

### 可能涉及的文件

- `schema.prisma`
- `src/file-upload/operations.ts`
- `src/file-upload/s3Utils.ts`
- `src/file-upload/fileUploading.ts`
- `src/file-upload/FileUploadPage.tsx`
- `src/file-upload/env.ts`
- `src/file-upload/file-upload.wasp.ts`
- `src/env.ts`
- `package.json`

## 6. 微信支付 / 支付宝

### 不建议直接扩展现有接口的原因

现有 `PaymentProcessor` 同时包含“支付网关”和“订阅账单系统”职责。国内支付接入应先拆分能力，再决定哪些 Provider 支持哪些能力。

### 推荐领域边界

```text
template/app/src/payment/
├─ domain/
│  ├─ order.ts
│  ├─ paymentStatus.ts
│  ├─ entitlement.ts
│  └─ webhookEvent.ts
├─ gateways/
│  ├─ gateway.ts
│  ├─ stripe/
│  ├─ wechat/
│  └─ alipay/
├─ billing/
│  ├─ subscriptionBilling.ts
│  ├─ stripe/
│  ├─ lemonSqueezy/
│  └─ polar/
├─ operations.ts
└─ payment.wasp.ts
```

### 推荐新增数据模型

- `PaymentOrder`
- `PaymentAttempt`
- `PaymentTransaction`
- `PaymentWebhookEvent`
- `Refund`
- `EntitlementGrant` 或独立 Usage Ledger

Webhook 应先按 Provider Event ID 或商户订单号做幂等登记，再更新订单和权益。不要让支付回调直接无条件增加 `User.credits`。

### 前端返回值

当前 `CheckoutSession` 只适合跳转 URL。国内支付需要允许返回不同展现方式，例如：

- `redirectUrl`
- `qrCodeContent`
- `jsApiParams`
- `formPayload`

具体支持哪些形式取决于 Web、H5、公众号、小程序或 App 产品形态，当前待验证。

### 可能涉及的文件

- `schema.prisma`
- `src/payment/paymentProcessor.ts`
- `src/payment/operations.ts`
- `src/payment/webhook.ts`
- `src/payment/payment.wasp.ts`
- `src/payment/plans.ts`
- `src/payment/paymentProcessorPlans.ts`
- `src/payment/PricingPage.tsx`
- `src/payment/CheckoutResultPage.tsx`
- `src/payment/user.ts`
- `src/env.ts`
- `.env.server.example`
- `package.json`

## 7. AI 工作流运行与状态追踪

### 推荐目录

```text
template/app/src/ai-workflow/
├─ ai-workflow.wasp.ts
├─ operations.ts
├─ workers.ts
├─ engine/
│  ├─ runner.ts
│  ├─ stepExecutor.ts
│  └─ stateMachine.ts
├─ events/
├─ pages/
└─ validation.ts
```

### 推荐数据模型

- `WorkflowDefinition`
- `WorkflowVersion`
- `WorkflowRun`
- `WorkflowStepRun`
- `WorkflowEvent`
- `UsageReservation` 或 `UsageLedger`

至少应记录：

- `status`
- `startedAt`、`finishedAt`
- `currentStep`
- 输入、输出引用
- Provider、模型和 Provider request ID
- Token/费用
- 重试次数
- 归一化错误码和可展示错误信息
- 创建用户和租户归属

### 执行方式

MVP 可复用 Wasp PgBoss Job。Action 创建 `WorkflowRun` 后提交 Job，Worker 更新步骤状态。Wasp Jobs 支持持久化、重试、延迟和 Job ID：<https://wasp.sh/docs/advanced/jobs>。

但 PgBoss 与 Web Server 同进程，不适合高并发、CPU 密集或需要独立扩缩容的 AI Worker。是否切换到外部队列和独立 Worker，应根据工作流持续时间、并发、失败率和部署方式决定。

产品状态必须写入自己的 Workflow 表，不应依赖 PgBoss 内部表作为用户可见状态。

### 可能涉及的文件

- `main.wasp.ts`
- `schema.prisma`
- `src/env.ts`
- 新建 `src/ai-workflow/**`
- 新建 `src/ai/**`
- `src/demo-ai-app/operations.ts`
- `src/user/AccountPage.tsx`
- 管理后台中需要新增的运行监控页面
