# RESTful API 设计指南

## 目录

- [1. 简介](#1-简介)
- [2. URI 设计规范](#2-uri-设计规范)
- [3. HTTP 方法使用规范](#3-http-方法使用规范)
- [4. 查询参数规范](#4-查询参数规范)
- [5. 请求与响应格式](#5-请求与响应格式)
- [6. 状态码使用规范](#6-状态码使用规范)
- [7. 错误处理规范](#7-错误处理规范)
- [8. 版本控制](#8-版本控制)
- [9. 安全性考虑](#9-安全性考虑)
- [10. 文档规范](#10-文档规范)
- [11. 最佳实践示例](#11-最佳实践示例)

## 1. 简介

### 1.1 目的

本文档旨在为所有项目提供统一的 RESTful API 设计规范，确保 API 的一致性、可读性和可维护性。遵循这些规范将有助于提高开发效率、减少沟通成本，并为客户端开发人员提供更好的体验。

### 1.2 适用范围

本规范适用于所有基于 HTTP 协议的 API 设计，包括但不限于：

- 面向客户端的公开 API
- 内部系统间的 API 交互
- 微服务之间的通信接口

## 2. URI 设计规范

### 2.1 基本结构

所有 API 的 URI 应遵循以下基本结构：

```
/{version}/{resources}/{resource_id}/{sub-resources}/{sub-resource_id}
```

例如：

```
/v1/events/123/guests/456
```

### 2.2 命名规则

- **使用名词复数**：资源名称应使用名词复数形式，例如 `events`、`guests`、`tables`
- **使用小写字母**：URI 中的所有字母均使用小写
- **使用连字符（-）**：多个单词组成的资源名称使用连字符连接，而不是下划线或驼峰命名，例如 `guest-groups`
- **避免文件扩展名**：不要在 URI 中包含文件扩展名，如 `.json`

### 2.3 资源层次结构

资源之间的层次关系应在 URI 中清晰表达：

- **主资源**：直接位于版本号之后，例如 `/v1/events`
- **子资源**：表示从属于主资源的实体，例如 `/v1/events/123/guests`
- **资源操作**：特殊的非 CRUD 操作应作为子资源，例如 `/v1/events/123/publish`

### 2.4 避免动词

 URI 中应避免使用动词，而应使用 HTTP 方法来表示操作。例如：

- 不推荐：`/v1/events/123/delete`
- 推荐：使用 DELETE 方法访问 `/v1/events/123`

特殊情况下，对于无法用标准 HTTP 方法表达的复杂操作，可以使用特定的资源名称：

- `/v1/events/123/seating-arrangements` (POST 方法生成座位安排)

## 3. HTTP 方法使用规范

### 3.1 方法定义

| 方法 | 用途 | 是否幂等 | 是否安全 |
|------|------|---------|---------|
| GET | 获取资源 | 是 | 是 |
| POST | 创建资源或执行操作 | 否 | 否 |
| PUT | 全量更新资源 | 是 | 否 |
| PATCH | 部分更新资源 | 否 | 否 |
| DELETE | 删除资源 | 是 | 否 |

### 3.2 方法使用场景

#### GET

- 获取单个资源：`GET /v1/events/123`
- 获取资源集合：`GET /v1/events`
- 获取子资源：`GET /v1/events/123/guests`

#### POST

- 创建新资源：`POST /v1/events`
- 执行复杂操作：`POST /v1/events/123/seating-arrangements`
- 批量操作：`POST /v1/events/batch-delete`

#### PUT

- 全量更新资源：`PUT /v1/events/123`（需提供完整的资源表示）
- 有条件创建资源：`PUT /v1/events/123`（如果资源不存在则创建）

#### PATCH

- 部分更新资源：`PATCH /v1/events/123`（只更新提供的字段）

#### DELETE

- 删除单个资源：`DELETE /v1/events/123`
- 删除子资源：`DELETE /v1/events/123/guests/456`

## 4. 查询参数规范

### 4.1 过滤

使用查询参数进行资源过滤，参数名应具有描述性：

```
/v1/guests?status=pending&department=sales
```

对于复杂过滤条件，可以使用以下格式：

```
/v1/guests?age_gt=30&age_lt=50
```

### 4.2 排序

使用 `sort` 参数指定排序字段，使用 `-` 前缀表示降序：

```
/v1/guests?sort=name         # 按名称升序排序
/v1/guests?sort=-created_at  # 按创建时间降序排序
```

支持多字段排序：

```
/v1/guests?sort=department,-name  # 先按部门升序，再按名称降序
```

### 4.3 分页

所有返回集合的 API 都应支持分页，使用以下参数：

- `page`：页码，从 1 开始（默认值：1）
- `page_size`：每页记录数（默认值：10，最大值：100）

```
/v1/events/123/guests?page=2&page_size=20
```

### 4.4 字段选择

使用 `fields` 参数允许客户端指定需要返回的字段：

```
/v1/guests/123?fields=id,name,email,phone
```

### 4.5 搜索

使用 `q` 参数进行全文搜索：

```
/v1/guests?q=张三
```

### 4.6 参数命名规则

- 使用小写字母
- 多个单词使用下划线连接（例如 `page_size`）
- 参数名应具有描述性
- 布尔参数使用 `is_` 前缀（例如 `is_active`）

## 5. 请求与响应格式

### 5.1 内容类型

- 请求和响应的默认内容类型为 `application/json`
- 请求头应包含 `Content-Type: application/json`
- 响应头应包含 `Content-Type: application/json; charset=utf-8`

### 5.2 请求体格式

请求体应为有效的 JSON 对象，字段名使用小驼峰命名法：

```json
{
  "name": "年会活动",
  "description": "公司年度庆典",
  "eventDate": "2025-12-31T18:00:00Z",
  "location": "北京国际会议中心"
}
```

### 5.3 响应体格式

所有响应应遵循统一的格式：

```json
{
  "data": {
    // 实际数据，可以是对象或数组
  },
  "meta": {
    // 元数据，如分页信息
    "pagination": {
      "total": 100,
      "page": 2,
      "pageSize": 10,
      "totalPages": 10
    }
  }
}
```

对于集合资源，`data` 字段应为数组：

```json
{
  "data": [
    { "id": 1, "name": "张三" },
    { "id": 2, "name": "李四" }
  ],
  "meta": {
    "pagination": {
      "total": 100,
      "page": 1,
      "pageSize": 10,
      "totalPages": 10
    }
  }
}
```

### 5.4 日期时间格式

所有日期时间应使用 ISO 8601 格式（UTC）：

```
YYYY-MM-DDThh:mm:ssZ
```

例如：`2025-05-28T06:16:18Z`

### 5.5 空值处理

- 使用 `null` 表示空值，而不是空字符串或 -1
- 不要在响应中包含值为 `null` 的字段，除非客户端明确需要

## 6. 状态码使用规范

### 6.1 常用状态码

| 状态码 | 含义 | 使用场景 |
|--------|------|---------|
| 200 OK | 请求成功 | GET、PUT、PATCH 请求成功 |
| 201 Created | 资源创建成功 | POST 请求创建资源成功 |
| 204 No Content | 请求成功但无返回内容 | DELETE 请求成功 |
| 400 Bad Request | 请求参数错误 | 请求参数不符合要求 |
| 401 Unauthorized | 未认证 | 未提供认证信息或认证失败 |
| 403 Forbidden | 权限不足 | 已认证但无权访问资源 |
| 404 Not Found | 资源不存在 | 请求的资源不存在 |
| 409 Conflict | 资源冲突 | 请求与当前资源状态冲突 |
| 422 Unprocessable Entity | 请求格式正确但语义错误 | 验证失败 |
| 429 Too Many Requests | 请求过于频繁 | 超出速率限制 |
| 500 Internal Server Error | 服务器内部错误 | 服务器发生未预期的错误 |

### 6.2 状态码分类

- 2xx：成功
- 3xx：重定向
- 4xx：客户端错误
- 5xx：服务器错误

## 7. 错误处理规范

### 7.1 错误响应格式

所有错误响应应遵循统一的格式：

```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "找不到指定的资源",
    "details": [
      {
        "field": "eventId",
        "message": "ID为123的活动不存在"
      }
    ]
  }
}
```

### 7.2 错误码规范

错误码应使用大写字母和下划线，具有描述性，例如：

- `INVALID_PARAMETER`：参数无效
- `RESOURCE_NOT_FOUND`：资源不存在
- `PERMISSION_DENIED`：权限不足
- `RATE_LIMIT_EXCEEDED`：超出速率限制
- `INTERNAL_ERROR`：内部错误

### 7.3 验证错误

对于验证错误，应返回 422 状态码，并在 `details` 中提供每个字段的错误信息：

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "请求参数验证失败",
    "details": [
      {
        "field": "name",
        "message": "活动名称不能为空"
      },
      {
        "field": "eventDate",
        "message": "活动日期必须是未来时间"
      }
    ]
  }
}
```

## 8. 版本控制

### 8.1 版本号格式

版本号应包含在 URI 路径中，使用 `v` 前缀加数字：

```
/v1/events
/v2/events
```

### 8.2 版本升级原则

- **主版本升级**：不兼容的 API 更改（例如删除字段、更改字段类型）
- **次版本升级**：向后兼容的功能添加（例如新增可选字段、新增端点）
- **补丁版本升级**：向后兼容的错误修复（通常不会更改 API 版本号）

### 8.3 版本共存

- 新版本发布后，旧版本应继续维护一段时间（至少 6 个月）
- 明确标注已废弃的 API，并在响应头中包含过期信息

## 9. 安全性考虑

### 9.1 认证与授权

- 使用 Bearer Token 进行认证：`Authorization: Bearer {token}`
- 对敏感操作要求更高级别的授权
- 使用 HTTPS 加密所有通信

### 9.2 速率限制

- 实施基于 IP 和用户的速率限制
- 在响应头中包含速率限制信息：
  - `X-RateLimit-Limit`：时间窗口内的最大请求数
  - `X-RateLimit-Remaining`：当前时间窗口内剩余的请求数
  - `X-RateLimit-Reset`：速率限制重置的时间戳

### 9.3 数据验证

- 严格验证所有输入数据
- 实施输入长度限制
- 防止 SQL 注入、XSS 等常见攻击

## 10. 文档规范

### 10.1 文档格式

- 使用 OpenAPI (Swagger) 规范编写 API 文档
- 文档应包含所有端点、参数、响应格式和错误码

### 10.2 文档内容要求

每个 API 端点的文档应包含：

- 端点描述
- 请求方法和 URL
- 路径参数和查询参数
- 请求体格式（如适用）
- 响应格式和状态码
- 错误码和错误消息
- 示例请求和响应
- 权限要求

### 10.3 文档维护

- API 文档应与代码同步更新
- 使用自动化工具从代码注释生成文档

## 11. 最佳实践示例

### 11.1 完整的 API 设计示例

以下是一个符合本规范的 API 设计示例：

#### 获取活动列表

```
GET /v1/events?page=1&page_size=10&sort=-created_at
```

响应：

```json
{
  "data": [
    {
      "id": 123,
      "name": "年会活动",
      "description": "公司年度庆典",
      "eventDate": "2025-12-31T18:00:00Z",
      "location": "北京国际会议中心",
      "createdAt": "2025-05-15T10:30:00Z",
      "updatedAt": "2025-05-28T06:16:18Z"
    },
    // 更多活动...
  ],
  "meta": {
    "pagination": {
      "total": 45,
      "page": 1,
      "pageSize": 10,
      "totalPages": 5
    }
  }
}
```

#### 创建活动

```
POST /v1/events
Content-Type: application/json

{
  "name": "产品发布会",
  "description": "新产品发布会",
  "eventDate": "2025-06-15T14:00:00Z",
  "location": "上海展览中心"
}
```

响应：

```json
{
  "data": {
    "id": 124,
    "name": "产品发布会",
    "description": "新产品发布会",
    "eventDate": "2025-06-15T14:00:00Z",
    "location": "上海展览中心",
    "createdAt": "2025-05-28T06:16:18Z",
    "updatedAt": "2025-05-28T06:16:18Z"
  }
}
```

#### 获取活动嘉宾列表

```
GET /v1/events/124/guests?status=pending&page=1&page_size=10
```

响应：

```json
{
  "data": [
    {
      "id": 456,
      "eventId": 124,
      "name": "张三",
      "email": "zhangsan@example.com",
      "phone": "+86 13800000000",
      "company": "ABC公司",
      "position": "技术总监",
      "rsvpStatus": "pending",
      "createdAt": "2025-05-28T06:16:18Z"
    },
    // 更多嘉宾...
  ],
  "meta": {
    "pagination": {
      "total": 35,
      "page": 1,
      "pageSize": 10,
      "totalPages": 4
    }
  }
}
```

#### 错误示例

```
GET /v1/events/999
```

响应：

```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "找不到指定的活动",
    "details": [
      {
        "field": "id",
        "message": "ID为999的活动不存在"
      }
    ]
  }
}
```

### 11.2 Proto 文件示例

```protobuf
syntax = "proto3";

package tableplan.v1;

import "google/api/annotations.proto";

option go_package = "github.com/xinyuan-tech/table-plan/api/tableplan/v1;v1";

// 活动服务
service EventService {
  // 获取活动列表
  rpc ListEvents (ListEventsRequest) returns (ListEventsReply) {
    option (google.api.http) = {
      get: "/v1/events"
    };
  }
  
  // 创建活动
  rpc CreateEvent (CreateEventRequest) returns (CreateEventReply) {
    option (google.api.http) = {
      post: "/v1/events"
      body: "*"
    };
  }
  
  // 获取活动详情
  rpc GetEvent (GetEventRequest) returns (GetEventReply) {
    option (google.api.http) = {
      get: "/v1/events/{id}"
    };
  }
  
  // 更新活动
  rpc UpdateEvent (UpdateEventRequest) returns (UpdateEventReply) {
    option (google.api.http) = {
      put: "/v1/events/{id}"
      body: "*"
    };
  }
  
  // 删除活动
  rpc DeleteEvent (DeleteEventRequest) returns (DeleteEventReply) {
    option (google.api.http) = {
      delete: "/v1/events/{id}"
    };
  }
  
  // 获取活动的嘉宾列表
  rpc ListEventGuests (ListEventGuestsRequest) returns (ListEventGuestsReply) {
    option (google.api.http) = {
      get: "/v1/events/{event_id}/guests"
    };
  }
}

// 获取活动列表请求
message ListEventsRequest {
  int32 page = 1;      // 页码，从1开始
  int32 page_size = 2; // 每页数量
  string sort = 3;     // 排序字段，例如 "name" 或 "-created_at"
  string q = 4;        // 搜索关键词
}

// 活动信息
message EventInfo {
  uint64 id = 1;           // 活动ID
  string name = 2;         // 活动名称
  string description = 3;  // 活动描述
  string event_date = 4;   // 活动日期，ISO8601格式
  string location = 5;     // 活动地点
  string created_at = 6;   // 创建时间，ISO8601格式
  string updated_at = 7;   // 更新时间，ISO8601格式
}

// 分页信息
message PaginationInfo {
  int32 total = 1;       // 总记录数
  int32 page = 2;        // 当前页码
  int32 page_size = 3;   // 每页数量
  int32 total_pages = 4; // 总页数
}

// 获取活动列表响应
message ListEventsReply {
  repeated EventInfo data = 1;     // 活动列表
  PaginationInfo pagination = 2;   // 分页信息
}
```

### 11.3 实现示例

```go
// 获取活动列表
func (s *EventService) ListEvents(ctx context.Context, req *v1.ListEventsRequest) (*v1.ListEventsReply, error) {
    // 设置默认分页参数
    page := int(req.Page)
    pageSize := int(req.PageSize)
    if page <= 0 {
        page = 1
    }
    if pageSize <= 0 {
        pageSize = 10 // 默认每页 10 条记录
    }
    if pageSize > 100 {
        pageSize = 100 // 最大每页 100 条记录
    }
    
    // 解析排序参数
    sortField := "created_at"
    sortOrder := "DESC"
    if req.Sort != "" {
        if strings.HasPrefix(req.Sort, "-") {
            sortField = strings.TrimPrefix(req.Sort, "-")
            sortOrder = "DESC"
        } else {
            sortField = req.Sort
            sortOrder = "ASC"
        }
    }
    
    // 调用业务层获取活动列表
    events, total, err := s.uc.ListEvents(ctx, page, pageSize, sortField, sortOrder, req.Q)
    if err != nil {
        return nil, err
    }
    
    // 构建响应
    eventInfos := make([]*v1.EventInfo, 0, len(events))
    for _, event := range events {
        eventInfos = append(eventInfos, convertToEventInfo(event))
    }
    
    totalPages := (total + pageSize - 1) / pageSize
    
    return &v1.ListEventsReply{
        Data: eventInfos,
        Pagination: &v1.PaginationInfo{
            Total:      int32(total),
            Page:       int32(page),
            PageSize:   int32(pageSize),
            TotalPages: int32(totalPages),
        },
    }, nil
}
```

---

本文档将作为 API 设计的标准指南，所有新开发的 API 都应遵循此规范。对于现有 API，应在后续迭代中逐步调整以符合规范。