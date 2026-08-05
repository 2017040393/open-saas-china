# 分阶段重构路线图

## 1. 改造原则

- 保持当前 Wasp 纵向功能模块结构，不为了“分层”而把简单调用拆成大量空壳。
- 先建立可测试的接口和行为基线，再接入第二个 Provider。
- 把领域状态与第三方状态分开保存。
- 数据库事务只负责本地一致性；外部调用通过幂等、用量账本、事件表和补偿任务处理。
- 不把支付、模型、存储厂商的专有字段继续扩散到页面和核心数据模型。
- 每阶段都应同步评估模板、E2E、演示站补丁和公开文档。

## 2. 阶段 0：建立行为基线

目标：在重构前证明当前行为，避免“接口抽象”悄悄改变订阅、额度或上传语义。

建议补充的特征测试：

- AI 成功生成结构化日程；
- 未登录请求返回 401；
- 无订阅用户消耗 credit；
- credit 为 0 返回 402；
- 两个并发请求不会超扣；
- 模型成功但数据库失败时能够审计；
- 支付重复 Webhook 不重复发放权益；
- 下载 URL 只能由对象所有者获得；
- 存储删除失败能够重试或进入待清理状态。

当前 E2E 已覆盖三次免费生成、第四次 402、Stripe 付款后继续生成，但它依赖真实 OpenAI 和 Stripe，速度和成本较高。后续还需要 Provider Fake/Contract Test。

## 3. 阶段 1：第一个小型重构——提取 AI 日程生成器

这是最适合作为首个任务的模块。

### 原因

- 当前 OpenAI 耦合集中在一个文件的一段函数中，改动面小。
- 不需要立刻修改数据库或支付逻辑。
- 可以保持现有同步 HTTP、页面和额度行为不变。
- 提取后可用 Fake Provider 做快速测试。
- 直接为国内模型接入和后续 Workflow 复用建立入口。

### 建议边界

```text
template/app/src/ai/
├─ scheduleGenerator.ts
└─ providers/
   └─ openAiScheduleGenerator.ts
```

第一步只定义当前业务真正需要的 `ScheduleGenerator` 领域接口，不急于创建覆盖对话、图片、Embedding、Agent、MCP 的统一大接口。

### 保持不变

- `generateGptResponse` Wasp Action 名称和前端调用；
- 当前输入 `{ hours }`；
- `GeneratedSchedule` 返回结构；
- 订阅和 credits 语义；
- `GptResponse` 保存方式；
- 当前默认 OpenAI Provider。

### 移出 Action 的职责

- OpenAI SDK 初始化；
- 模型参数；
- messages/tool schema 适配；
- tool call 解析；
- OpenAI 特有错误映射。

### 验收标准

- 当前 E2E 行为不变；
- Action 不再 import `openai`；
- 单元测试可注入 Fake Schedule Generator；
- OpenAI API key 仍只在服务端读取；
- 不引入数据库迁移。

## 4. 阶段 2：对象存储中性化

目标：把 S3 从应用层契约降级为一个 Provider 实现。

顺序建议：

1. 定义 `StorageProvider`。
2. 用现有 S3 行为实现第一个 Adapter。
3. 把返回字段改为 `objectKey`、`uploadUrl`、`uploadFields`。
4. 为数据库字段 `s3Key` 制定兼容迁移。
5. 加入对象所有权校验和删除补偿状态。
6. 再接入 OSS、COS，并运行统一 Contract Test。

此阶段不应同时重写文件页面视觉或引入复杂媒体处理。

## 5. 阶段 3：Entitlement 与 Usage Ledger

目标：让 AI 调用、一次性 credits、订阅权益和未来 Workflow 使用统一的授权与结算规则。

建议流程：

1. 在模型调用前事务性预留额度。
2. 预留失败立即拒绝，不调用模型。
3. 模型成功后结算实际用量。
4. 可重试失败保留运行状态，最终失败释放预留。
5. 每次变更形成不可变 Usage Ledger 记录。

这一阶段是支付和 AI 工作流的共同依赖。

## 6. 阶段 4A：AI Workflow Run MVP

目标：把同步 AI 调用提升为可查询状态的后台运行。

建议最小闭环：

- `startWorkflow` Action 创建 Run；
- 使用 PgBoss 提交 Worker；
- Worker 顺序执行步骤；
- `getWorkflowRun` Query 返回状态和步骤；
- 页面轮询状态；
- 每一步记录错误、重试和用量；
- 管理员可查看失败 Run。

MVP 先不做可视化编排、任意 DAG、暂停/恢复和多队列调度。先证明状态模型和失败恢复正确。

## 7. 阶段 4B：支付领域与国内网关

这一阶段可以在 Usage Ledger 稳定后与 Workflow MVP 并行。

顺序建议：

1. 引入 PaymentOrder 和 WebhookEvent。
2. 为现有 Stripe 回调增加幂等处理，验证领域模型。
3. 拆分 Payment Gateway 与 Subscription Billing。
4. 明确国内产品形态和支付场景。
5. 接入首个国内网关。
6. 再接入第二个网关，通过统一 Contract Test 验证。

不要同时接入微信支付和支付宝后再设计订单模型；应先用一个现有 Provider 证明抽象。

## 8. 阶段 5：生产化与扩缩容

建议关注：

- Provider 超时、退避、熔断和限流；
- 回调验签、重放保护和原始事件留存；
- Outbox、补偿任务和死信处理；
- Token、人民币成本和毛利统计；
- 日志脱敏和密钥轮换；
- Workflow 卡住检测和人工重试；
- 独立 Worker 与外部队列的容量评估；
- 国内区域、域名、备案、内容审核和数据合规要求。

## 9. 涉及文件总览

| 领域 | 主要现有文件 | 建议新增目录 |
|---|---|---|
| AI Provider | `demo-ai-app/operations.ts`、`schedule.ts`、`src/env.ts` | `src/ai/` |
| 存储 | `file-upload/operations.ts`、`s3Utils.ts`、`FileUploadPage.tsx`、`schema.prisma` | `file-upload/storage/` |
| 支付 | `paymentProcessor.ts`、`operations.ts`、`webhook.ts`、`plans.ts`、`schema.prisma` | `payment/domain/`、`payment/gateways/`、`payment/billing/` |
| Workflow | `main.wasp.ts`、`schema.prisma`、`src/env.ts` | `src/ai-workflow/` |
| 权益和用量 | `demo-ai-app/operations.ts`、`payment/user.ts`、`schema.prisma` | `src/entitlement/` 或相邻领域模块 |

## 10. 实施前待验证

- 国内大模型候选、结构化输出兼容性、内容审核规则和计费方式。
- OSS/COS 是否需要和 S3 数据共存，以及历史对象迁移方式。
- 微信/支付宝采用 Web、H5、公众号、小程序还是 App 场景。
- 是否需要自动续费；这会显著影响支付领域模型和签约流程。
- AI Workflow 最长执行时间、平均并发、峰值并发和允许重试次数。
- 是否接受 PgBoss 与 Web Server 同进程，或从第一版就部署独立 Worker。
- 多租户、团队账户、发票和人民币结算是否属于近期需求。

这些问题没有从当前代码中得到答案，不能在实现阶段默认猜测。
