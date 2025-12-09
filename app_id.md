### appId的传递

### 统一传递方式（唯一标准）

**规则：只从 HTTP Header 获取，通过 Context 传播**

## 1. 客户端传递方式

### 前端应用（dev-share-web, table-plan-web）

**推荐方式：从路由参数获取，通过 Header 传递**

```typescript
// 1. 从路由参数获取 appId（动态）
const appId = params.appId as string  // 例如：/dashboard/apps/[appId]/...

// 2. 调用 API 时显式传递 appId
const response = await api.listPlans(appId)
const response = await api.listUsers({ page: 1 }, appId)

// 3. API 函数通过 Header 传递
// client.ts 会自动将 appId 添加到 X-App-Id header
```

**注意：** 必须显式传递 appId，不支持从 localStorage 获取

### 其他客户端

- **HTTP Header（唯一来源）**
  - 客户端发送：`X-App-Id`（应用ID）
  - API Gateway 验证后透传：`X-App-Id`（应用ID）

## 2. 后端处理方式

### 微服务中间件

- 微服务中间件从 Header 提取 `X-App-Id` 并放入 Context
- 统一的中间件：`app_id.Middleware()`
- 支持 HTTP Header 和 gRPC metadata 两种方式

### Context 传播（内部传递）

- 统一的 Context Key：`appId`
- 统一的工具函数：`app_id.GetAppIDFromContext(ctx)`
- 业务逻辑层只从 Context 获取，不再从请求体或请求参数获取

### 错误处理

- 如果 Context 中没有 `appId`，返回明确的错误：`app_id is required, please provide X-App-Id header`
- 不再支持从请求体或请求参数获取 `appId`，保持代码简洁统一

## 3. 设计原则

- **灵活性**：前端支持动态传递 appId（从路由参数获取），适应多应用场景
- **统一性**：后端统一只从 Header 获取，通过 Context 传播
- **简洁性**：不依赖固定的 localStorage，避免切换应用时的混乱