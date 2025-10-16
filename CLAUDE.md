# CLAUDE.md

此文件为 Claude Code (claude.ai/code) 在此代码库中工作时提供指导。

## 项目概述

Raphael Starter Kit 是一个 Next.js SaaS 启动套件，包含以下功能：
- AI 驱动的中文名字生成（核心产品功能）
- Supabase 身份验证和数据库
- Creem.io 支付/订阅系统
- 基于积分和订阅的双重变现模式
- 完整的暗色/亮色主题支持
- 使用 shadcn/ui 组件的响应式设计

## 开发命令

```bash
# 开发环境
npm i                 # 安装依赖
npm run dev          # 启动开发服务器 (localhost:3000)

# 生产环境
npm run build        # 构建生产版本
npm run start        # 启动生产服务器
```

## 架构概览

### 身份验证流程
- 使用 Supabase Auth，支持邮箱/密码和 OAuth（Google、GitHub）
- 中间件（`utils/supabase/middleware.ts`）保护 `/dashboard/*` 路由
- 认证回调在 `app/auth/callback/route.ts` 处理
- 公开页面允许匿名用户访问，通过 IP 地址进行速率限制

### 支付与订阅架构
应用使用 **Creem.io** 进行支付，具有两种变现模式：

1. **订阅模式**（`subscriptions` 表）
   - 循环订阅计划
   - 通过 Creem.io webhooks 管理
   - 跟踪状态：active（活跃）、canceled（已取消）、expired（已过期）、trialing（试用中）

2. **积分模式**（`customers.credits`、`credits_history` 表）
   - 一次性积分购买
   - 按生成次数计费（1 积分 = 标准版，4 积分 = 高级版）
   - 每次生成时扣除积分余额

**Webhook 处理**（`app/api/webhooks/creem/route.ts`）：
- 使用 `CREEM_WEBHOOK_SECRET` 验证签名
- 处理事件：`checkout.completed`、`subscription.active`、`subscription.paid`、`subscription.canceled`、`subscription.expired`、`subscription.trialing`
- 通过 `utils/supabase/subscriptions.ts` 更新 `customers` 和 `subscriptions` 表
- 一次性购买时添加积分（metadata: `product_type: "credits"`）

### 中文名字生成系统
**核心 API**：`app/api/chinese-names/generate/route.ts`

**功能特性**：
- 使用 OpenAI/OpenRouter API（通过 `OPENAI_BASE_URL` 配置）
- 支持两种计划类型：标准版（1 积分）和高级版（4 积分）
- 匿名用户生成 3 个名字，认证用户生成 6 个名字
- 免费用户基于 IP 的速率限制（每天 3 次，通过 `check_ip_rate_limit` RPC）
- 通过 `generation_batches` 和 `generated_names` 表跟踪批量生成
- 支持批次续生成（为相同参数生成更多名字）

**数据模型**：
- `generation_batches`：存储生成会话元数据
- `generated_names`：单个名字及 generation_round 追踪
- `name_generation_logs`：分析日志
- `saved_names`：用户收藏的名字

### 数据库结构（Supabase）

**关键表**：
- `customers`：关联 user_id 到 Creem 客户，存储积分余额
- `subscriptions`：活跃订阅状态和周期
- `credits_history`：积分交易日志（添加/扣除）
- `generation_batches`：名字生成会话
- `generated_names`：生成的中文名字及文化元数据
- `saved_names`：用户收藏的名字

**重要的 RPC 函数**：
- `check_ip_rate_limit(p_client_ip)`：返回免费层限制的布尔值

### 组件架构

**布局组件**（`components/`）：
- `header.tsx`：顶部导航，包含认证状态
- `footer.tsx`：网站页脚
- `mobile-nav.tsx`：移动端导航抽屉
- `theme-switcher.tsx`：暗色/亮色模式切换

**仪表板组件**（`components/dashboard/`）：
- `credits-balance-card.tsx`：显示剩余积分
- `subscription-status-card.tsx`：活跃订阅信息
- `generation-history-card.tsx`：过往生成批次
- `my-names-card.tsx`：已保存/收藏的名字
- `subscription-portal-dialog.tsx`：通过 Creem 门户管理订阅

**产品组件**（`components/product/`）：
- `generator/name-generator-form.tsx`：主要生成表单
- `results/names-grid.tsx`：显示生成的名字
- `results/name-card.tsx`：单个名字卡片，含保存/PDF 功能
- `pricing/chinese-name-pricing.tsx`：定价展示

**UI 组件**（`components/ui/`）：
- shadcn/ui 组件（button、card、dialog、form、input 等）
- 使用 Tailwind CSS 和 class-variance-authority 处理变体
- 通过 next-themes 实现主题感知

## 环境变量

必需的变量（参见 `.env.example`）：

**Supabase**：
- `NEXT_PUBLIC_SUPABASE_URL`
- `NEXT_PUBLIC_SUPABASE_ANON_KEY`
- `SUPABASE_SERVICE_ROLE_KEY`

**Creem.io**：
- `CREEM_WEBHOOK_SECRET`：Webhook 签名验证
- `CREEM_API_KEY`：API 认证
- `CREEM_API_URL`：`https://test-api.creem.io/v1`（测试）或 `https://api.creem.io`（生产）
- `CREEM_SUCCESS_URL`：支付后重定向 URL

**AI 服务**：
- `OPENROUTER_API_KEY` 或 `OPENAI_API_KEY`：用于名字生成
- `OPENAI_BASE_URL`：默认为 OpenRouter（`https://openrouter.ai/api/v1`）
- `DOUBAO_TTS_APPID`、`DOUBAO_TTS_ACCESS_TOKEN`：可选的 TTS 服务

**URLs**：
- `NEXT_PUBLIC_SITE_URL`：前端 URL，用于认证回调
- `BASE_URL`：不含协议的基础 URL

## 开发模式

### TypeScript 路径别名
- 使用 `@/*` 从项目根目录导入：`import { Button } from '@/components/ui/button'`

### 服务端 vs 客户端组件
- 默认使用服务端组件（无需 `"use client"`）
- 对于交互性功能使用 `"use client"`（表单、状态、事件处理器、浏览器 API）
- 认证操作的服务端 Actions 在 `app/actions.ts` 中

### API 路由
- `app/api/` 中的所有路由都是服务端组件
- 对于认证请求使用 `@/utils/supabase/server` 中的 `createClient()`
- 对于管理员操作（webhooks、积分管理）使用 `createServiceRoleClient()`

### 表单处理
- 使用 `react-hook-form` 配合 `zod` 进行验证
- 认证表单使用服务端 Actions
- 生成和支付操作使用 API 路由

### 错误处理
- 所有异步操作使用 try-catch 块
- 返回带有适当状态码的 JSON 错误
- 记录带上下文的错误以便调试

### 样式
- Tailwind CSS 实用类
- 通过 CSS 自定义属性定义主题变量
- 使用 `cn()` 工具（tailwind-merge + clsx）处理条件类
- 通过 `next-themes` ThemeProvider 实现暗色/亮色模式

## 关键集成点

### Creem.io Webhook 安全
始终在处理前验证 webhook 签名：
```typescript
verifyCreemWebhookSignature(body, signature, CREEM_WEBHOOK_SECRET)
```

### 积分管理
使用 `utils/supabase/subscriptions.ts` 中的辅助函数：
- `addCreditsToCustomer()`：添加积分并记录交易历史
- `useCredits()`：扣除积分并进行验证和记录
- `getCustomerCredits()`：获取当前余额

### 名字生成 API
调用 `/api/chinese-names/generate` 时：
- 包含 `planType: '1' | '4'` 指定定价层
- 可选：`continueBatch: true` 和 `batchId` 以向现有批次添加更多名字
- 响应包含 `batchId` 用于跟踪和续生成

### 速率限制
免费用户通过 IP 地址限制：
- 通过 `check_ip_rate_limit` Supabase RPC 检查
- 超出限制时返回 429 状态
- 认证用户绕过 IP 限制但使用积分

## 测试注意事项

测试时：
- 使用测试信用卡：`4242 4242 4242 4242`（Creem 测试模式）
- 在 Creem.io 仪表板中启用测试模式
- 使用 `CREEM_API_URL=https://test-api.creem.io/v1`
- 使用 ngrok 或 Hookdeck 等工具在本地测试 webhook
- 验证 webhook 签名验证正常工作
- 测试免费（匿名）和认证用户流程
- 测试积分扣除和积分不足场景
- 测试生成批次续生成功能

## 常见任务

### 添加新的 API 路由
1. 在 `app/api/[route]/route.ts` 中创建路由处理器
2. 导入 `createClient()` 用于认证，或 `createServiceRoleClient()` 用于管理
3. 验证输入并检查认证/积分
4. 使用 try-catch 处理错误
5. 返回 NextResponse.json()

### 添加数据库表
1. 在 Supabase SQL 编辑器中创建迁移 SQL
2. 在 `types/` 目录中添加 TypeScript 类型
3. 如需要，更新 `utils/supabase/` 中的服务函数

### 更新 Webhook 处理器
1. 修改 `app/api/webhooks/creem/route.ts`
2. 在 switch 语句中添加新的事件类型
3. 按照现有模式创建处理函数
4. 如需要，更新 `types/creem.ts`

### 切换到生产环境
1. 禁用 Creem 测试模式
2. 将 `CREEM_API_URL` 更新为 `https://api.creem.io`
3. 创建生产环境 Creem 产品并更新代码中的 ID
4. 将 `NEXT_PUBLIC_SITE_URL` 和 `CREEM_SUCCESS_URL` 更新为生产域名
5. 在 Creem.io 仪表板中更新 webhook URL 为生产端点
6. 验证所有环境变量已在生产环境（Vercel/托管平台）中设置

## 文件路径参考

- 数据库工具：`utils/supabase/`
- Creem 工具：`utils/creem/`
- 类型定义：`types/`
- 认证 actions：`app/actions.ts`
- API 路由：`app/api/`
- 受保护路由：`app/dashboard/`
- 公开路由：`app/(auth-pages)/`、`app/product/`
- 中间件：`middleware.ts`
