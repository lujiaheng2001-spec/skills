# AI-Native PRD 模板

> 这个模板专为 AI agent 时代设计。传统 PRD 写给人类工程师，允许模糊和留白；
> 这个模板写给人 + AI，要求每个决策都显式声明——因为 AI 不会"猜"你的意思，
> 它只实现你明确说的东西。

## 使用指南

复制下面的模板，把 `[方括号]` 中的内容替换为你的具体信息。
删除所有注释行（以 `//` 开头的行）。
不需要的 section 可以标注"本期不涉及"而不是直接删除——
让 AI 知道你有意省略了某项，而不是忘记了。

---

## 模板正文

```markdown
# [产品/功能名称] PRD

## 0. 元信息
- 作者：[你的名字]
- 日期：[YYYY-MM-DD]
- 版本：[v0.1]
- 状态：[草稿 / 评审中 / 已确认]
- AI 工具：[Cursor / Claude Code / Replit / 其他]

---

## 1. 目标与背景

### 1.1 一句话说清楚
// 用一句话描述这个产品/功能解决什么问题。如果你说不清楚，说明你还没想清楚。
[例：让用户可以通过自然语言搜索自己的历史订单]

### 1.2 用户是谁
// 不要写"所有用户"。越具体，AI 生成的代码越贴合。
- 主要用户：[例：电商平台的回购用户，每月至少下单 2 次]
- 次要用户：[例：客服人员，需要帮用户查找订单]

### 1.3 成功指标
// 必须是可量化的。AI 无法优化"用户体验变好了"这种目标。
- [例：订单查找耗时从平均 45 秒降低到 10 秒以内]
- [例：客服相关工单减少 30%]

### 1.4 非目标（同样重要）
// 明确告诉 AI 什么不做，防止它过度实现。
- [例：不做推荐系统，不做订单修改功能，不做支付相关任何操作]

---

## 2. 技术栈（锁定，不可自行变更）

// 这一节极其重要。不写的话，AI 会自己选技术栈，可能选到不存在的包。
// 锁定主版本号，防止 AI 引入不兼容的版本。

| 层级 | 技术 | 版本 | 备注 |
|------|------|------|------|
| 前端 | [Next.js] | [14.x] | [App Router] |
| 样式 | [Tailwind CSS] | [3.x] | [不使用 CSS Modules] |
| 语言 | [TypeScript] | [5.x] | [strict mode] |
| 后端 | [Next.js API Routes] | [-] | [或独立 Express 服务] |
| ORM | [Prisma] | [5.x] | [-] |
| 数据库 | [PostgreSQL] | [15+] | [通过 Supabase 托管] |
| 认证 | [NextAuth.js] | [v5] | [支持 Google OAuth] |
| 部署 | [Vercel] | [-] | [-] |
| 监控 | [Sentry] | [-] | [免费方案即可] |

**依赖管理规则：**
- 不得引入上表未列出的依赖（如确实需要，暂停并说明理由）
- 安装前验证包在 npm 上真实存在、最近 6 个月有更新、周下载量 > 1000
- 禁止使用 `@latest` 标签，必须锁定具体版本

---

## 3. 数据模型

// 在写任何代码之前，先把数据结构定义清楚。这是 AI 生成代码质量最高的前提条件。

### 3.1 核心实体

```
// 用伪代码或 Prisma schema 格式定义
// AI 阅读结构化 schema 的效果远好于读自然语言描述

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  role      Role     @default(USER)
  orders    Order[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Order {
  id          String      @id @default(cuid())
  userId      String
  user        User        @relation(fields: [userId], references: [id])
  status      OrderStatus @default(PENDING)
  totalAmount Decimal     @db.Decimal(10, 2)
  items       OrderItem[]
  createdAt   DateTime    @default(now())
  updatedAt   DateTime    @updatedAt

  @@index([userId])
  @@index([createdAt])
}
```

### 3.2 关系说明
// 写出实体之间的关系，以及级联删除规则。AI 不会自动处理"删除用户时订单怎么办"这种问题。
- User 1:N Order（删除用户时：软删除，保留订单数据）
- Order 1:N OrderItem（删除订单时：级联删除关联的 OrderItem）

### 3.3 索引策略
// 告诉 AI 哪些字段会被高频查询，需要建索引。
- User.email（唯一索引，用于登录）
- Order.userId + Order.createdAt（复合索引，用于"我的订单"列表）

---

## 4. API 设计

// 逐个列出所有 API endpoint。不要只写"CRUD for orders"——AI 会按自己的理解实现，
// 可能缺少你需要的字段或返回你不想暴露的数据。

### 4.1 端点列表

| 方法 | 路径 | 认证 | 说明 |
|------|------|------|------|
| GET | /api/orders | 必须 | 获取当前用户的订单列表（分页） |
| GET | /api/orders/:id | 必须 | 获取单个订单详情（仅限本人） |
| POST | /api/orders/search | 必须 | 自然语言搜索订单 |

### 4.2 详细定义（每个端点都要写）

```
POST /api/orders/search
Authorization: Bearer <token>（必须）
Rate Limit: 10 次/分钟/用户

Request Body:
{
  "query": string,      // 必填，用户的自然语言查询
  "page": number,       // 可选，默认 1
  "pageSize": number    // 可选，默认 20，最大 100
}

Success Response (200):
{
  "data": Order[],       // 按相关度排序
  "total": number,
  "page": number,
  "pageSize": number
}

Error Responses:
- 400: { "error": "INVALID_QUERY", "message": "查询内容不能为空" }
- 401: { "error": "UNAUTHORIZED", "message": "请先登录" }
- 429: { "error": "RATE_LIMITED", "message": "请求过于频繁，请稍后再试" }
```

---

## 5. 用户流程（逐步骤）

// 不要只写"用户搜索订单"。把每一步交互写清楚，包括加载状态、错误状态、空状态。

### 5.1 正常流程（Happy Path）

1. 用户在搜索框输入自然语言（如"上个月买的耳机"）
2. 前端展示加载指示器（骨架屏，不是转圈）
3. 后端返回匹配的订单列表
4. 前端展示结果卡片（每张卡片显示：订单号、日期、金额、状态、商品缩略图）
5. 用户点击卡片，进入订单详情页

### 5.2 异常流程（必须覆盖）

| 场景 | 用户看到什么 | 技术处理 |
|------|-------------|---------|
| 搜索无结果 | "没有找到匹配的订单，试试换个关键词？" | 返回空数组，不是 404 |
| 网络错误 | "网络连接失败，请检查网络后重试" + 重试按钮 | catch fetch error，显示重试 UI |
| 搜索超时 | "搜索超时，请稍后再试" | 5 秒超时，abort controller |
| 未登录 | 跳转到登录页，登录后返回搜索页 | 302 重定向，保存 returnUrl |

---

## 6. 安全要求（不可降级）

// 这一节是红线。AI 经常为了"让代码跑起来"而跳过安全措施。
// 把它们写在 PRD 里，确保 AI 不会省略。

### 6.1 认证
- 所有 /api/* 端点必须验证 JWT token（除了 /api/auth/*）
- Token 通过 HTTP-only cookie 传递，不存 localStorage
- Refresh token 机制：access token 15 分钟过期，refresh token 7 天

### 6.2 授权
- 用户只能访问自己的数据（所有查询必须包含 userId 过滤）
- 管理员角色检查必须在服务端（不是前端 if/else）
- API 返回 403 而不是 404 来区分"没权限"和"不存在"
  // 注意：有些安全方案建议对未授权的资源统一返回 404 以防止枚举，
  // 根据你的场景选择策略，但要统一且有意识。

### 6.3 输入校验
- 前端校验用于即时反馈（UX 目的）
- 后端校验是安全边界（使用 zod 或 joi，不要手写 if/else）
- 搜索查询内容必须转义，防止注入

### 6.4 数据保护
- 密码使用 bcrypt（cost factor ≥ 12）
- 所有 PII 字段加密存储
- 日志中禁止出现：密码、token、信用卡号、身份证号
- 数据库连接使用 SSL

---

## 7. 测试要求

// 不写测试要求 = AI 不会写测试。明确告诉它测什么。

### 7.1 必须覆盖的测试

| 类型 | 覆盖范围 | 工具 |
|------|---------|------|
| 单元测试 | 数据校验函数、权限检查函数 | Jest / Vitest |
| API 测试 | 每个端点的正常和异常响应 | Supertest |
| 权限测试 | 用户 A 不能访问用户 B 的数据 | 自定义测试 |
| E2E | 搜索订单完整流程 | Playwright |

### 7.2 测试数据
- 使用 seed 脚本生成测试数据（不要用生产数据）
- 每次测试前重置数据库状态
- 测试环境使用独立的数据库实例

---

## 8. 交付里程碑

// 按 Phase 拆分，每个 Phase 是一次完整的 agent 会话。
// 每个 Phase 完成后，人工 review 代码再进入下一个 Phase。

| Phase | 交付内容 | 验收标准 |
|-------|---------|---------|
| 1 | 数据模型 + Migration | schema 可以正常创建，migration 可回滚 |
| 2 | API endpoints | 所有端点的正常/异常 case 通过 Postman 测试 |
| 3 | 认证系统 | 登录/登出/token 刷新正常工作 |
| 4 | 搜索功能 | 5 个测试查询都返回合理结果 |
| 5 | 前端页面 | 所有用户流程可走通，包括异常状态 |
| 6 | 测试 + 安全加固 | 测试通过率 > 90%，无高危漏洞 |
```

---

## 附录：PRD 写作检查清单

写完 PRD 后，对照这个清单确认：

- [ ] 每个 API 端点都有输入/输出定义和错误码
- [ ] 技术栈已锁定，包含版本号
- [ ] 数据模型包含字段类型、关系、索引
- [ ] 安全要求有独立的 section（不是散落在各处）
- [ ] 异常流程（空状态、错误、超时、未授权）都有定义
- [ ] 非目标已明确（AI 不做什么）
- [ ] 测试要求已明确（覆盖哪些场景）
- [ ] 交付分阶段，每阶段有验收标准
- [ ] 没有使用模糊词汇（"合适的""良好的""安全的"——这些对 AI 没有信息量）
