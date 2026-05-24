# AI 工具聚合平台 — 项目规划

> 项目代号：if-gen-ai
> 主体：企业（已具备营业执照、可申请微信/支付宝商户号、可 ICP 备案）
> MVP 范围：对话 + 图片生成（视频生成放 v2）
> 模型接入：通过自建中转站，统一 OpenAI 兼容协议（如 one-api / new-api）

---

## 1. 产品定位

一个聚合多家大模型能力的订阅制 SaaS：
- 用户注册 → 购买套餐/充值积分 → 在站内调用各家模型（对话 / 图像 / 视频）
- 平台通过自建中转站统一调度上游 API，按积分计费
- 管理员后台负责：用户、模型/渠道、积分、订单、统计、风控、内容审核

商业模式：套餐订阅 + 按量积分（参考 ai1600 的"积分/次"模式）。

---

## 2. 技术栈最终决定

### Frontend
| 选型 | 用途 | 备注 |
|---|---|---|
| **Next.js 15 (App Router)** | 全栈框架 | RSC + Server Actions，SEO 友好 |
| **TypeScript** | 类型 | 必选 |
| **Tailwind CSS v4** | 样式 | |
| **shadcn/ui** | 组件库 | 拷贝式，可控性高 |
| **next-intl** | i18n | ✅ 取代你提的 i18next；App Router 原生支持 |
| **TanStack Query** | 服务端状态 | 配合 Server Actions 用于客户端缓存 |
| **Zustand** | 客户端状态 | 轻量，UI 状态够用 |
| **react-hook-form + zod** | 表单/校验 | |
| **Auth.js v5 (NextAuth)** | 鉴权 | Credentials Provider 接国内短信 |
| **Sonner** | Toast | shadcn 推荐 |

### Backend
| 选型 | 用途 | 备注 |
|---|---|---|
| **Next.js Route Handlers / Server Actions** | API 层 | 同仓库，简化部署 |
| **独立 Node Worker (tsx + BullMQ)** | 异步任务 | ⚠️ 关键：不要在 Web 进程里跑长任务 |
| **Prisma** | ORM | |
| **PostgreSQL 16** | 主库 | |
| **Redis 7** | 缓存 + BullMQ 队列 + 限流 + Session | |
| **阿里云 OSS** | 生成产物存储 | 配 CDN |
| **阿里云短信 / 腾讯云 SMS** | 手机验证码 | |
| **阿里云内容安全 (Green)** | 文本/图片审核 | 国内合规必须 |
| **wechatpay-node-v3** | 微信支付 | JSAPI / Native / H5 |
| **alipay-sdk** | 支付宝 | 当面付 / 电脑网站 / 手机网站 |
| **Pino** | 日志 | 结构化 |
| **OpenTelemetry** | 链路追踪 | 后期接 Jaeger / 阿里云 ARMS |

### DevOps
| 选型 | 用途 |
|---|---|
| **Docker + docker-compose** | 本地与生产编排 |
| **阿里云 ECS** | 应用服务器 |
| **阿里云 RDS PostgreSQL** | 数据库（生产） |
| **阿里云 Tair (Redis)** | 缓存 |
| **阿里云 OSS + CDN** | 静态/产物 |
| **阿里云 SLB** | 负载均衡 |
| **GitHub Actions** | CI/CD（镜像推送 ACR） |
| **Sentry** | 错误监控 |

### 你原方案的调整说明
| 你的想法 | 调整 | 原因 |
|---|---|---|
| i18next | → **next-intl** | App Router 原生集成，少配置 |
| 未提队列 | **+ BullMQ + Redis** | 视频/图片生成必须异步 |
| 未提对象存储 | **+ 阿里云 OSS + CDN** | 生成物不能进 DB |
| 未提内容审核 | **+ 阿里云 Green** | 国内必备 |
| Auth 方式未定 | **Auth.js v5 + Credentials** | NextAuth 不直接支持国内短信，需自己写 Provider |
| 部署只提 Docker | **+ ICP 备案、RDS、SLB、ACR** | 阿里云生产实践 |

---

## 3. 合规与上线前置（不要等做完才发现）

⚠️ 这些项 **现在就启动**，否则会卡上线 2–4 周：

1. **ICP 备案**：阿里云控制台提交，企业资质 + 域名 + 服务器实例（必须先买 ECS）。约 7–20 工作日。
2. **公安备案**：ICP 通过后 30 天内做。
3. **微信支付商户号**：mp.weixin.qq.com，需对公账户验证。
4. **支付宝商户号**：open.alipay.com，企业自研模式。
5. **阿里云短信签名 + 模板**：需提交营业执照审核，1–3 天。
6. **生成式 AI 服务备案**（依据《生成式人工智能服务管理暂行办法》）：如果对外提供生成服务，主体需向网信办备案。**这一项目前是行业灰色地带，许多中小平台先上线后跟进；建议至少把内容审核接入做扎实。**
7. **隐私协议 / 用户协议 / 退款政策**：上线必备。

---

## 4. 架构总览

```
                 ┌──────────────┐
                 │   用户浏览器  │
                 └──────┬───────┘
                        │ HTTPS
                  ┌─────▼──────┐
                  │   阿里云 SLB │
                  └─────┬──────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
   ┌────▼─────┐   ┌─────▼────┐    ┌────▼─────┐
   │ Next.js  │   │ Next.js  │    │  Worker  │  (BullMQ consumer)
   │  (Web)   │   │  (Web)   │    │  process │
   └────┬─────┘   └────┬─────┘    └────┬─────┘
        │              │               │
        └──────┬───────┴───────────────┘
               │
   ┌───────────┼───────────┬───────────┬───────────┐
   │           │           │           │           │
┌──▼──┐    ┌──▼──┐     ┌──▼──┐    ┌──▼──┐    ┌──▼──────────┐
│ RDS │    │Redis│     │ OSS │    │ SMS │    │ 中转站 (上游)│
│ PG  │    │Tair │     │+CDN │    │Green│    │ OpenAI 兼容  │
└─────┘    └─────┘     └─────┘    └─────┘    └─────────────┘
```

**关键点**：
- Web 进程**只做** API 路由、SSR、Server Actions、SSE 连接
- Worker 进程**只做** 重活：调上游模型、轮询任务、写 OSS、扣积分、内容审核
- Web → Worker 通信走 Redis 队列；Worker → Web → 用户的进度推送走 Redis Pub/Sub + SSE

---

## 5. 数据模型（Prisma 草案）

> 这是骨架，字段会在实现时细化。**积分必须走账本**，不要在 `User` 上直接 `credits` 字段做加减。

```prisma
// === 用户 & 鉴权 ===
model User {
  id            String   @id @default(cuid())
  phone         String?  @unique
  email         String?  @unique
  passwordHash  String?
  nickname      String?
  avatarUrl     String?
  role          Role     @default(USER)   // USER | ADMIN | FINANCE
  status        UserStatus @default(ACTIVE)
  createdAt     DateTime @default(now())
  // 关系
  subscriptions Subscription[]
  orders        Order[]
  tasks         GenerationTask[]
  ledgerEntries CreditLedger[]
  apiKeys       ApiKey[]  // 用户自己的 API key（可选功能）
  @@index([createdAt])
}

enum Role { USER ADMIN FINANCE SUPPORT }
enum UserStatus { ACTIVE BANNED PENDING }

// === 模型 & 渠道 ===
// Provider = 模型厂商分组（OpenAI, Anthropic, ...）
// Model    = 具体模型（gpt-5, claude-opus-4-7, ...）
// Channel  = 上游中转渠道（一个模型可能有多个渠道，按价格/速度/成功率路由）
model Provider {
  id     String  @id @default(cuid())
  name   String  @unique  // "OpenAI"
  slug   String  @unique  // "openai"
  iconUrl String?
  models Model[]
}

model Model {
  id           String   @id @default(cuid())
  providerId   String
  provider     Provider @relation(fields: [providerId], references: [id])
  slug         String   @unique          // "gpt-5"
  displayName  String                    // "GPT-5"
  modality     Modality                  // CHAT | IMAGE | VIDEO
  inputPricePer1M  Decimal?  @db.Decimal(18, 6)  // 文本积分/百万token
  outputPricePer1M Decimal?  @db.Decimal(18, 6)
  perCallPrice     Decimal?  @db.Decimal(18, 6)  // 按次（图/视频）
  enabled      Boolean  @default(true)
  channels     Channel[]
}

enum Modality { CHAT IMAGE VIDEO AUDIO }

model Channel {
  id          String   @id @default(cuid())
  modelId     String
  model       Model    @relation(fields: [modelId], references: [id])
  name        String                    // "XL 官转OpenAI"
  baseUrl     String                    // 中转站 URL
  apiKeyEnc   String                    // ⚠️ AES 加密存储
  costPer1M   Decimal  @db.Decimal(18,6) // 我方成本（用于利润核算）
  weight      Int      @default(100)
  successRate Float    @default(1.0)
  enabled     Boolean  @default(true)
  priority    Int      @default(0)      // 价格优先/速度优先时使用
}

// === 积分账本（不可变）===
model CreditLedger {
  id          String   @id @default(cuid())
  userId      String
  user        User     @relation(fields: [userId], references: [id])
  type        LedgerType        // RECHARGE | CONSUME | REFUND | GRANT | EXPIRE
  amount      Decimal  @db.Decimal(18, 6)   // 正负
  balanceAfter Decimal @db.Decimal(18, 6)
  refType     String?           // "order" | "task" | "admin"
  refId       String?
  idempotencyKey String? @unique
  meta        Json?
  createdAt   DateTime @default(now())
  @@index([userId, createdAt])
}

enum LedgerType { RECHARGE CONSUME REFUND GRANT EXPIRE ADJUST }

// === 套餐 & 订单 & 支付 ===
model Plan {
  id          String   @id @default(cuid())
  slug        String   @unique
  name        String
  priceCents  Int                // 人民币分
  creditsGranted Decimal @db.Decimal(18,6)
  validDays   Int?               // null = 永久
  enabled     Boolean  @default(true)
}

model Order {
  id           String      @id @default(cuid())
  outTradeNo   String      @unique   // 我方流水号
  userId       String
  user         User        @relation(fields: [userId], references: [id])
  planId       String?
  amountCents  Int
  channel      PayChannel             // WECHAT | ALIPAY
  status       OrderStatus            // PENDING | PAID | FAILED | REFUNDED
  paidAt       DateTime?
  transactionId String?               // 三方流水号
  rawCallback  Json?
  createdAt    DateTime @default(now())
}

enum PayChannel { WECHAT_NATIVE WECHAT_H5 WECHAT_JSAPI ALIPAY_PC ALIPAY_WAP }
enum OrderStatus { PENDING PAID FAILED REFUNDED CLOSED }

model Subscription {
  id        String   @id @default(cuid())
  userId    String
  planId    String
  startAt   DateTime
  expireAt  DateTime
  status    String   // ACTIVE | EXPIRED | CANCELLED
}

// === 生成任务 ===
model GenerationTask {
  id          String   @id @default(cuid())
  userId      String
  user        User     @relation(fields: [userId], references: [id])
  modality    Modality
  modelId     String
  channelId   String?  // 实际命中渠道
  status      TaskStatus
  prompt      String?  @db.Text
  params      Json
  inputAssetUrls String[]
  outputAssetUrls String[]
  errorCode   String?
  errorMsg    String?
  costCredits Decimal? @db.Decimal(18,6)
  upstreamCostCents Int?       // 我方成本（财务用）
  createdAt   DateTime @default(now())
  startedAt   DateTime?
  finishedAt  DateTime?
  @@index([userId, createdAt])
  @@index([status])
}

enum TaskStatus { QUEUED RUNNING SUCCEEDED FAILED CANCELLED }

// === 对话会话（CHAT 用，单独建表更灵活）===
model Conversation {
  id        String   @id @default(cuid())
  userId    String
  title     String?
  modelId   String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  messages  Message[]
}

model Message {
  id             String   @id @default(cuid())
  conversationId String
  conversation   Conversation @relation(fields: [conversationId], references: [id], onDelete: Cascade)
  role           String   // user | assistant | system
  content        String   @db.Text
  tokensIn       Int?
  tokensOut      Int?
  costCredits    Decimal? @db.Decimal(18,6)
  createdAt      DateTime @default(now())
}

// === 审计 ===
model AuditLog {
  id        String   @id @default(cuid())
  actorId   String?
  action    String
  target    String?
  meta      Json?
  ip        String?
  createdAt DateTime @default(now())
}
```

---

## 6. 关键子系统设计

### 6.1 鉴权（Auth.js v5）
- **Credentials Provider** × 2：
  - `phone-otp`：手机号 + 短信验证码（验证码 Redis 存 5 分钟，60 秒重发）
  - `email-password`：邮箱 + 密码（bcrypt）
- Session：JWT（默认）+ Edge 中间件鉴权
- RBAC：中间件读取 `session.user.role`，`/admin/*` 限 `ADMIN`

### 6.2 积分扣费（账本模式）
```
function consumeCredits(userId, amount, ref) tx {
  lock user row (SELECT FOR UPDATE)
  current = sum(ledger.amount where userId)   // 或维护一个 balance cache
  if current < amount: throw InsufficientCredits
  insert ledger { CONSUME, -amount, balanceAfter, idempotencyKey: ref }
}
```
- 所有扣费 / 充值 / 退款 都过这个函数
- `idempotencyKey` 唯一索引保证重复回调不重复扣
- 余额缓存到 Redis（TTL 60s），写时失效

### 6.3 异步任务（BullMQ）
- 队列：`chat`（短，可同步流式）/ `image`（30s–2min）/ `video`（2–10min）
- Web 接到请求 → 预扣积分 → 入队 → 返回 taskId
- Worker 消费：调用上游 → 轮询/流式 → 写 OSS → 更新 task → 推 SSE → 多退少补
- 失败 → 退还预扣 + 写错误日志
- **渠道路由**：按"价格优先/速度优先/成功率优先"策略选 Channel，失败自动切换重试

### 6.4 进度推送
- **对话**：Server-Sent Events（SSE），Server Action 直接 stream
- **图片/视频**：客户端轮询 `GET /api/tasks/:id` 或订阅 SSE `GET /api/tasks/:id/stream`
- Worker → Redis Pub/Sub → Web SSE 转发

### 6.5 支付
- 用户点支付 → 创建 Order（PENDING）→ 调支付 SDK 返回二维码/链接
- 用户支付 → 三方异步回调 `/api/pay/callback/wechat` `/api/pay/callback/alipay`
- 回调：验签 → 幂等检查 → 更新 Order → 写 ledger（RECHARGE）→ 返回 success
- 前端轮询订单状态显示成功

### 6.6 内容审核
- 文本：Prompt 进队前过阿里云 Green Text
- 图片：上游返回后过 Green Image，违规则标记 + 不返给用户 + 扣费策略可配
- 高危用户自动封禁

### 6.7 i18n
- next-intl，`messages/{zh-CN,en,...}.json`
- 路由：`/[locale]/...`，默认 zh-CN
- 服务端组件直接 `getTranslations`

### 6.8 管理后台
- 路径：`/admin/*`，中间件鉴权
- 模块：
  - Dashboard：今日/本月 收入、用户、调用量、成本毛利
  - 用户管理：搜索、详情、调整积分、封禁
  - 模型与渠道：增删改、启停、密钥管理（加密展示）、测试调用
  - 套餐管理
  - 订单与流水
  - 积分账本（财务对账）
  - 任务列表（含失败排查）
  - 内容审核
  - 系统设置 / 公告 / 协议

---

## 7. 目录结构

```
if-gen-ai/
├── apps/
│   ├── web/                    # Next.js
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── [locale]/
│   │   │   │   │   ├── (marketing)/       # 首页、定价
│   │   │   │   │   ├── (auth)/            # 登录、注册
│   │   │   │   │   ├── (app)/             # 工具页（对话、图片、视频）
│   │   │   │   │   └── admin/             # 后台
│   │   │   │   └── api/                   # Route Handlers
│   │   │   ├── components/
│   │   │   ├── lib/
│   │   │   │   ├── auth/
│   │   │   │   ├── db/                    # prisma client
│   │   │   │   ├── redis/
│   │   │   │   ├── credits/               # ledger
│   │   │   │   ├── queue/                 # bullmq producers
│   │   │   │   ├── upstream/              # OpenAI 兼容客户端
│   │   │   │   ├── payment/
│   │   │   │   ├── oss/
│   │   │   │   ├── moderation/
│   │   │   │   └── sms/
│   │   │   ├── messages/                  # i18n
│   │   │   └── middleware.ts
│   │   └── package.json
│   └── worker/                            # 独立进程
│       ├── src/
│       │   ├── workers/
│       │   │   ├── chat.worker.ts
│       │   │   ├── image.worker.ts
│       │   │   └── video.worker.ts
│       │   ├── router/                    # 渠道路由策略
│       │   └── index.ts
│       └── package.json
├── packages/
│   ├── db/                                # Prisma schema + migrations
│   ├── shared/                            # 共享类型、zod schemas
│   └── config/                            # eslint, tsconfig
├── docker/
│   ├── Dockerfile.web
│   ├── Dockerfile.worker
│   └── docker-compose.yml
├── .github/workflows/
│   ├── ci.yml
│   └── deploy.yml
├── PLAN.md
└── README.md
```

> 选 Turborepo 做 monorepo（web + worker 共享 packages）。

---

## 8. 迭代路线图

### Phase 0 — 项目初始化（1–2 天）
- [ ] Turborepo + pnpm 初始化
- [ ] Next.js 15 + Tailwind + shadcn 接入
- [ ] Prisma + Postgres + Redis 本地 docker-compose
- [ ] CI（lint / typecheck / test）
- [ ] 提交首个 PR，跑通 `pnpm dev`

### Phase 1 — Auth & 基础壳（3–5 天）
- [ ] Auth.js v5 + 手机/邮箱登录
- [ ] 阿里云短信对接（沙盒）
- [ ] next-intl 中英双语
- [ ] 首页 / 登录页 / 用户中心骨架
- [ ] 用户表 + Session

### Phase 2 — 对话功能（4–6 天）
- [ ] Conversation / Message 模型
- [ ] OpenAI 兼容客户端封装（中转站调用）
- [ ] 流式对话（Server Action + SSE）
- [ ] 模型选择器、价格优先/速度优先选择
- [ ] 历史对话侧边栏

### Phase 3 — 积分 & 支付（5–7 天）
- [ ] CreditLedger 实现 + 余额缓存
- [ ] 套餐展示页 / 充值页
- [ ] 微信支付 Native（PC 扫码）
- [ ] 支付宝当面付
- [ ] 回调幂等、订单状态机
- [ ] 对话扣费接入

### Phase 4 — 图片生成 & 任务队列（5–7 天）
- [ ] BullMQ 接入，独立 worker 进程
- [ ] 图片生成页（上传、提示词、宽高比、质量）
- [ ] 阿里云 OSS 上传/下载（前端走签名直传）
- [ ] 任务历史、SSE 进度
- [ ] 失败重试 + 多渠道路由

### Phase 5 — 内容审核 & 风控（3 天）
- [ ] 阿里云 Green 文本/图片审核
- [ ] 高危拦截 + 用户标记
- [ ] 限流（IP / 用户 / 模型维度）

### Phase 6 — 管理后台 v1（5–7 天）
- [ ] RBAC 中间件
- [ ] Dashboard 数据看板
- [ ] 用户 / 模型 / 渠道 / 套餐 / 订单 CRUD
- [ ] 积分调整 + 审计日志
- [ ] 数据筛选与导出（CSV）

### Phase 7 — 部署上线（3–5 天）
- [ ] Dockerfile 多阶段构建
- [ ] GitHub Actions → 阿里云 ACR
- [ ] ECS 部署 + Nginx + Certbot
- [ ] RDS / Tair / OSS / CDN 接通
- [ ] Sentry + 日志收集
- [ ] 灰度上线、监控告警

### Phase 8 — 视频生成（v2，2 周）
- [ ] Video worker（长任务、轮询上游）
- [ ] 视频播放、下载、续费
- [ ] 视频内容审核（绿网视频）

---

## 9. 风险与对策

| 风险 | 影响 | 对策 |
|---|---|---|
| ICP 备案未通过 | 无法上线 | 先备案，海外节点跑 demo |
| 上游中转站抖动 | 用户体验差 | 多渠道 + 自动切换 + 熔断 |
| 并发扣费错乱 | 资损 | 账本模式 + 行锁 + idempotency |
| 支付回调重放 | 重复发货 | outTradeNo 唯一 + 状态机 |
| 内容违规 | 平台关停 | 文本/图像审核必接，违规日志留存 |
| 用户大额白嫖（注册送积分） | 资损 | 短信验证 + 设备指纹 + 风控规则 |
| 上游 key 泄露 | 巨额损失 | DB 加密、最小化展示、操作审计 |
| 长任务进程崩溃 | 任务丢失 | BullMQ 持久化 + 重试 + 死信队列 |
| AI 生成内容侵权 | 法律风险 | 用户协议明确归属、保留输入日志 |

---

## 10. 开发约定

- **包管理**：pnpm
- **Node**：>= 20 LTS
- **代码风格**：ESLint + Prettier + simple-import-sort
- **提交规范**：Conventional Commits（`feat:`, `fix:`, `chore:` ...）
- **分支**：`main`（生产）/ `dev`（集成）/ `feat/*` `fix/*`
- **PR**：必须过 CI，至少自审；重大改动写 ADR 落到 `docs/adr/`
- **密钥**：本地 `.env.local`；生产用阿里云 KMS / 环境变量；**禁止入库**
- **测试**：核心金额相关代码必须有单元测试（Vitest）；E2E 用 Playwright（后期）

---

## 11. 下一步行动（执行清单）

1. 确认本文档无误
2. 启动 Phase 0：初始化 Turborepo + Next.js + Prisma 骨架
3. 并行启动备案、商户号、短信签名申请（线下流程）
4. 按 Phase 1–7 推进，每个 Phase 结束做一次 demo + 复盘

---

_本文档为活文档，随实现迭代更新。重大决策变更需在 `docs/adr/` 留档。_
