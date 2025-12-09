### appId的传递

### 统一传递方式（唯一标准）

**规则：只从 HTTP Header 获取，通过 Context 传播**

1. **HTTP Header（唯一来源）**
   - 客户端发送：`X-App-Id`（应用ID）
   - API Gateway 验证后透传：`X-App-Id`（应用ID）
   - 微服务中间件从 Header 提取并放入 Context

2. **Context 传播（内部传递）**
   - 统一的 Context Key：`appId`
   - 统一的工具函数：`app_id.GetAppIDFromContext(ctx)`
   - 业务逻辑层只从 Context 获取，不再从请求体或请求参数获取

3. **错误处理**
   - 如果 Context 中没有 `appId`，返回明确的错误：`app_id is required, please provide X-App-Id header`
   - 不再支持从请求体或请求参数获取 `appId`，保持代码简洁统一