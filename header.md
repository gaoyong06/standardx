#### Header 规范

| Header 名称    | 用途         | 设置者         | 使用场景               |
| -------------- | ------------ | -------------- | ---------------------- |
| X-Developer-Id | 开发者 UID   | api-key 插件   | 配额扣费、服务调用权限 |
| X-End-User-Id  | 终端用户 UID | jwt-user 插件  | 文件存储、业务数据隔离 |
| X-App-Id       | 应用 ID      | 客户端/Gateway | 多应用隔离             |
| X-Service-Name | 微服务 ID    | api-key 插件   | 配额扣费               |
