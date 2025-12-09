### 关于response

response 在go-pkg包中间件中处理, 返回数据结构统一为:

```json
{
    "success": true,
    "data": ...,
    "errorCode": "",
    "errorMessage": "",
    "showType": 0,
    "traceId": "20251209155459-cccccccc",
    "host": "localhost:8101"
}
```

结构体定义见:
/Users/gaoyong/Documents/work/xinyuan_tech/go-pkg/middleware/response/structure.go

