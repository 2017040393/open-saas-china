# Open SaaS 架构与国内生态适配说明

本目录记录当前仓库的真实架构、主要调用链、厂商耦合以及后续国内生态适配建议。结论以当前检出的代码为准；官方文档只用于确认 Wasp 和 Open SaaS 的职责边界。

## 文档导航

1. [仓库结构与运行时架构](01-repository-and-runtime-architecture.md)
   - 仓库根目录及 `template/app` 的职责
   - Wasp、React、Node.js、Prisma 的关系
   - 当前模块化单体架构图
2. [AI 请求、认证、额度与事务](02-ai-request-auth-quota-transactions.md)
   - 一次 AI 请求的完整时序
   - 用户认证、订阅和额度处理位置
   - 显式数据库事务及一致性边界
3. [Provider 现状、耦合与国内适配](03-provider-coupling-and-china-adaptation.md)
   - 当前支付、存储、邮件、AI 和分析服务
   - 已有抽象和强耦合模块
   - 国内大模型、OSS/COS、微信支付/支付宝、AI 工作流的推荐扩展点
4. [分阶段重构路线图](04-refactoring-roadmap.md)
   - 推荐实施顺序
   - 第一个小型重构任务
   - 待验证事项和决策门槛

## 核心结论

- 当前仓库是 Open SaaS 模板的开发仓库，真正的 SaaS 应用模板位于 [`template/app`](../../template/app)。
- 运行时是一个按业务功能纵向拆分的 Wasp 模块化单体，不是微服务架构。
- Wasp 是 React 客户端、Node.js 服务端和 Prisma/PostgreSQL 数据层之间的编译与编排层。
- 支付已经具备部分 Provider 抽象，邮件由 Wasp 提供统一抽象。
- AI 直接绑定 OpenAI，对象存储直接绑定 AWS S3，是当前最明显的厂商耦合点。
- AI 工作流扩展前，应先建立独立的 AI Provider、Entitlement/Usage Ledger 和 Workflow Run 状态模型。

## 阅读与维护约定

- “当前事实”表示能从本仓库代码直接确认。
- “建议”表示后续设计方向，不代表仓库已经实现。
- “待验证”表示缺少运行环境、实际配置、商户信息或容量数据，不能仅凭当前代码确认。
- 若修改模板核心能力，应同时评估 `template/e2e-tests`、`opensaas-sh/app_diff` 和公开文档是否需要同步。

## 分析基线

- Wasp 声明版本：`^0.25.0`
- 数据库 Provider：PostgreSQL
- 当前代码默认支付 Provider：Stripe
- 当前代码默认邮件 Provider：Dummy
- 当前 AI SDK：OpenAI
- 当前对象存储 SDK：AWS S3

本次分析没有运行 Wasp 编译、服务启动、数据库迁移或第三方服务连接测试。
