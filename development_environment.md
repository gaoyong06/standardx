### 开发环境配置

### 目前开发环境架构

```
┌─────────────────────────────────────────┐
│         Docker Compose                  │
│  ┌──────────┐  ┌──────────┐            │
│  │  APISIX  │  │   etcd   │            │
│  └────┬─────┘  └──────────┘            │
│       │                                 │
│  ┌────▼─────┐                          │
│  │ RocketMQ │                          │
│  └──────────┘                          │
└─────────────────────────────────────────┘
         │
         │ 通过 host.docker.internal
         │
┌────────▼────────────────────────────────┐
│         本地开发环境                      │
│  ┌──────────┐  ┌──────────┐            │
│  │  MySQL   │  │  Redis   │            │
│  └──────────┘  └──────────┘            │
│                                         │
│  ┌──────────────┐  ┌──────────────┐    │
│  │passport-svc  │  │payment-svc   │    │
│  └──────────────┘  └──────────────┘    │
│  ┌──────────────┐  ┌──────────────┐    │
│  │subscription  │  │notification  │    │
│  └──────────────┘  └──────────────┘    │
│  ┌──────────────┐  ┌──────────────┐    │
│  │asset-svc     │  │marketing-svc │    │
│  └──────────────┘  └──────────────┘    │
│  ┌──────────────┐  ┌──────────────┐    │
│  │api-key-svc   │  │billing-svc   │    │
│  └──────────────┘  └──────────────┘    │
│  ┌──────────────┐                       │
│  │plugin-runner │                       │
│  └──────────────┘                       │
└─────────────────────────────────────────┘
```

### 配置文件

1. 开发环境 Docker Compose
```yaml
/Users/gaoyong/Documents/work/xinyuan_tech/apisix-docker/example/docker-compose.dev.yml
```

2. 微服务配置
```yaml
# cd到各个微服务中, 如 cd /Users/gaoyong/Documents/work/xinyuan_tech/api-key-service
configs/config.yaml
```

3. APISIX 配置
```yaml
/Users/gaoyong/Documents/work/xinyuan_tech/apisix-docker/example/apisix_conf/config.yaml
```

4. 插件配置（Go Plugin Runner）
```yaml
/Users/gaoyong/Documents/work/xinyuan_tech/apisix-devshare-plugin-runner/configs/config.yaml
```
注意：此配置文件包含插件的白名单路由配置（`public_routes`），用于定义哪些 API 不需要 API Key 验证

#### 3. 开发工作流

```bash
# 1. 启动基础设施（Docker 容器：APISIX、etcd、RocketMQ）
cd apisix-docker/example
docker-compose -f docker-compose.dev.yml up -d

# 2. 启动所有微服务（使用 devops-tools）
cd ../../devops-tools
make restart-all
# 或者在任何微服务目录下（使用 Makefile.common）：
# make restart-all

# 3. 构建插件（Linux 版本，用于 Docker 容器）
cd ../apisix-devshare-plugin-runner
GOOS=linux GOARCH=amd64 go build -o bin/go-runner/go-runner ./cmd/go-runner

# 4. 配置 APISIX 路由（首次运行或路由变更后需要执行）
cd ../devops-tools/scripts
./setup-apisix.sh

# 5. 测试
# 注意：登录接口需要使用 POST 方法
curl -X POST http://localhost:9080/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"contactType":"email","contact":"test@example.com","password":"test"}'
```

**注意**：
- `make restart-all` 定义在 `devops-tools/Makefile.common` 中
- 所有微服务通过 `include` 引入 `Makefile.common`，可以使用 `make restart-all`
- 插件需要构建 Linux 版本才能在 Docker 容器中运行

### RocketMQ
使用 docker-compose 管理 RocketMQ：
启动：docker-compose -f docker-compose.dev.yml up -d
重启：docker-compose -f docker-compose.dev.yml restart rocketmq-broker

### API 调用示例

#### 示例：调用 Passport Service 登录接口

**接口定义**（`passport-service/api/passport/v1/passport.proto`）：
```protobuf
rpc Login (LoginRequest) returns (TokenResponse) {
  option (google.api.http) = {
    post: "/v1/auth/login"
    body: "*"
  };
}
```

**请求流程**：

1. **客户端请求地址**（通过 APISIX Gateway）：
   ```bash
   POST http://localhost:9080/v1/auth/login
   ```

2. **APISIX 路由匹配**：
   - Route: `/v1/auth/*` → Service: `passport-service` → Upstream: `passport-upstream`

3. **最终调用地址**（passport-service）：
   - 在 Docker 容器内：`http://host.docker.internal:8100/v1/auth/login`
   - 在宿主机：`http://localhost:8100/v1/auth/login`

**完整请求示例**：

```bash
# 通过 APISIX Gateway 调用（推荐）
# 注意：/v1/auth/login 是公开 API，不需要 X-API-Key header
curl -X POST http://localhost:9080/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "contact_type": "email",
    "contact": "user@example.com",
    "password": "password123"
  }'

# 直接调用 passport-service（仅用于调试）
curl -X POST http://localhost:8100/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "contact_type": "email",
    "contact": "user@example.com",
    "password": "password123"
  }'
```

**公开 API vs 需要 API Key 的 API**：

| API 类型 | 示例路由 | 是否需要 API Key | 说明 |
|---------|---------|----------------|------|
| 公开 API | `/v1/auth/login`<br>`/v1/auth/register`<br>`/v1/auth/captcha` | ❌ 不需要 | 用户注册、登录等公开接口，配置在插件白名单中 |
| 业务 API | `/v1/users`<br>`/v1/payment`<br>`/v1/billing` | ✅ 需要 | 需要开发者 API Key 验证的业务接口 |

**需要 API Key 的请求示例**：

```bash
# 调用需要 API Key 的 API（如用户管理）
curl -X GET http://localhost:9080/v1/users \
  -H "X-API-Key: your-api-key"
```

**请求时序图（公开 API，如登录）**：

```
客户端                    APISIX Gateway              Go Plugin Runner          passport-service
  │                           │                            │                          │
  │  POST /v1/auth/login      │                            │                          │
  │  (无需 X-API-Key)         │                            │                          │
  ├───────────────────────────>│                            │                          │
  │                           │                            │                          │
  │                           │  ext-plugin-pre-req        │                          │
  │                           ├───────────────────────────>│                          │
  │                           │                            │                          │
  │                           │                            │  1. api-key 插件         │
  │                           │                            │     - 检查白名单         │
  │                           │                            │     - 跳过 API Key 验证  │
  │                           │                            │                          │
  │                           │                            │  2. jwt-user 插件        │
  │                           │                            │     - 解析 JWT（如果有） │
  │                           │                            │     - 设置 X-End-User-Id │
  │                           │                            │                          │
  │                           │                            │  3. billing 插件        │
  │                           │                            │     - 检查白名单         │
  │                           │                            │     - 跳过计费           │
  │                           │                            │                          │
  │                           │  ext-plugin-pre-req 返回    │                          │
  │                           │<───────────────────────────┤                          │
  │                           │                            │                          │
  │                           │  转发请求到 upstream       │                          │
  │                           │  POST /v1/auth/login       │                          │
  │                           ├──────────────────────────────────────────────────────>│
  │                           │                            │                          │
  │                           │                            │  处理登录逻辑            │
  │                           │                            │  - 验证用户名密码        │
  │                           │                            │  - 生成 JWT Token        │
  │                           │                            │                          │
  │                           │  返回响应                  │                          │
  │                           │<──────────────────────────────────────────────────────┤
  │                           │                            │                          │
  │  返回响应                 │                            │                          │
  │<──────────────────────────┤                            │                          │
  │                           │                            │                          │
```

**请求时序图（需要 API Key 的业务 API）**：

```
客户端                    APISIX Gateway              Go Plugin Runner          backend-service
  │                           │                            │                          │
  │  GET /v1/users             │                            │                          │
  │  X-API-Key: xxx           │                            │                          │
  ├───────────────────────────>│                            │                          │
  │                           │                            │                          │
  │                           │  ext-plugin-pre-req        │                          │
  │                           ├───────────────────────────>│                          │
  │                           │                            │                          │
  │                           │                            │  1. api-key 插件验证     │
  │                           │                            │     - 验证 API Key        │
  │                           │                            │     - 设置 X-Developer-Id │
  │                           │                            │     - 设置 X-Service-Name│
  │                           │                            │                          │
  │                           │                            │  2. jwt-user 插件        │
  │                           │                            │     - 解析 JWT（如果有） │
  │                           │                            │     - 设置 X-End-User-Id │
  │                           │                            │                          │
  │                           │                            │  3. billing 插件        │
  │                           │                            │     - 扣减配额           │
  │                           │                            │                          │
  │                           │  ext-plugin-pre-req 返回    │                          │
  │                           │<───────────────────────────┤                          │
  │                           │                            │                          │
  │                           │  转发请求到 upstream       │                          │
  │                           │  GET /v1/users             │                          │
  │                           │  X-Developer-Id: xxx       │                          │
  │                           │  X-Service-Name: xxx       │                          │
  │                           ├──────────────────────────────────────────────────────>│
  │                           │                            │                          │
  │                           │                            │  处理业务逻辑            │
  │                           │                            │                          │
  │                           │  返回响应                  │                          │
  │                           │<──────────────────────────────────────────────────────┤
  │                           │                            │                          │
  │  返回响应                 │                            │                          │
  │<──────────────────────────┤                            │                          │
  │                           │                            │                          │
```

**路由配置说明**：

| 层级 | 配置项 | 值 | 说明 |
|------|--------|-----|------|
| Route | URI | `/v1/auth/*` | 匹配所有 `/v1/auth/` 开头的请求 |
| Route | Service ID | `passport-service` | 关联的服务配置 |
| Service | Upstream ID | `passport-upstream` | 关联的上游配置 |
| Upstream | Nodes | `host.docker.internal:8100` | 后端服务地址（Docker 容器访问宿主机） |
| Service | Plugins | `ext-plugin-pre-req` | 启用 Go 插件（api-key, jwt-user, billing） |

**注意事项**：

1. **API Key 验证**：
   - **公开 API**（如 `/v1/auth/login`、`/v1/auth/register`）：不需要 `X-API-Key` header，配置在插件白名单中
   - **业务 API**（如 `/v1/users`、`/v1/payment`）：需要 `X-API-Key` header 进行验证
   - 白名单路由配置在 `apisix-devshare-plugin-runner/configs/config.yaml` 的 `plugins.api_key.public_routes` 中
2. **路径匹配**：APISIX 路由使用前缀匹配（`/v1/auth/*`），会转发完整路径到后端
3. **插件执行顺序**：
   - api-key（优先级 1000）：检查白名单，非白名单路由验证 API Key
   - jwt-user（优先级 950）：解析 JWT Token（如果有）
   - billing（优先级 900）：检查白名单，非白名单路由扣减配额
4. **直接访问**：开发时可以直接访问 `http://localhost:8100` 绕过 APISIX 进行调试
5. **白名单配置**：公开 API 的路由列表在插件配置文件中统一管理，便于维护


### 地址

apisix 本地地址
http://127.0.0.1:9180/ui

rocketmq 本地地址
http://127.0.0.1:8080

