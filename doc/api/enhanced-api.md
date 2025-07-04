# 增强 API 文档

## 概述

本文档提供 NocoBase API 的增强版本，包含完整的请求/响应示例、错误处理、性能优化建议和业务场景下的API组合使用示例。

## API 设计原则

### RESTful 设计

NocoBase 遵循 RESTful API 设计原则，采用资源和操作的模式：

```
# 基础格式
[GET|POST|PUT|DELETE] /api/{resource}[/{resourceId}][:{action}]

# 关系资源格式
[GET|POST|PUT|DELETE] /api/{resource}/{resourceId}/{relationResource}[/{relationResourceId}][:{action}]
```

### 统一响应格式

```typescript
interface APIResponse<T = any> {
  data?: T;
  meta?: {
    count?: number;
    page?: number;
    pageSize?: number;
    totalPage?: number;
  };
  errors?: Array<{
    message: string;
    code?: string;
    field?: string;
  }>;
}
```

## 完整 API 示例

### 1. 用户管理 API

#### 获取用户列表

**请求示例:**
```http
GET /api/users?page=1&pageSize=20&sort=-createdAt&filter[status]=active
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

**响应示例:**
```json
{
  "data": [
    {
      "id": 1,
      "name": "张三",
      "email": "zhangsan@example.com",
      "status": "active",
      "avatar": "https://example.com/avatars/1.jpg",
      "department": {
        "id": 2,
        "name": "技术部"
      },
      "roles": [
        {
          "id": 1,
          "name": "developer",
          "title": "开发者"
        }
      ],
      "createdAt": "2023-12-01T10:00:00.000Z",
      "updatedAt": "2023-12-01T10:00:00.000Z"
    }
  ],
  "meta": {
    "count": 150,
    "page": 1,
    "pageSize": 20,
    "totalPage": 8
  }
}
```

**错误响应示例:**
```json
{
  "errors": [
    {
      "message": "Invalid page parameter",
      "code": "INVALID_PARAMETER",
      "field": "page"
    }
  ]
}
```

#### 创建用户

**请求示例:**
```http
POST /api/users
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "name": "李四",
  "email": "lisi@example.com",
  "password": "password123",
  "departmentId": 2,
  "roles": [1, 2]
}
```

**响应示例:**
```json
{
  "data": {
    "id": 151,
    "name": "李四",
    "email": "lisi@example.com",
    "status": "active",
    "departmentId": 2,
    "createdAt": "2023-12-01T15:30:00.000Z",
    "updatedAt": "2023-12-01T15:30:00.000Z"
  }
}
```

**验证错误示例:**
```json
{
  "errors": [
    {
      "message": "Email already exists",
      "code": "DUPLICATE_EMAIL",
      "field": "email"
    },
    {
      "message": "Password must be at least 6 characters",
      "code": "INVALID_PASSWORD",
      "field": "password"
    }
  ]
}
```

### 2. 文件管理 API

#### 文件上传

**请求示例:**
```http
POST /api/attachments:upload
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: multipart/form-data

--boundary123
Content-Disposition: form-data; name="file"; filename="document.pdf"
Content-Type: application/pdf

[文件二进制数据]
--boundary123--
```

**响应示例:**
```json
{
  "data": {
    "id": 1,
    "filename": "document_20231201_153045.pdf",
    "originalname": "document.pdf",
    "mimetype": "application/pdf",
    "size": 1048576,
    "url": "https://example.com/uploads/document_20231201_153045.pdf",
    "path": "/uploads/2023/12/01/document_20231201_153045.pdf",
    "uploadedBy": 1,
    "createdAt": "2023-12-01T15:30:45.000Z"
  }
}
```

#### 批量文件上传

**请求示例:**
```http
POST /api/attachments:batchUpload
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: multipart/form-data

--boundary123
Content-Disposition: form-data; name="files"; filename="file1.jpg"
Content-Type: image/jpeg

[文件1二进制数据]
--boundary123
Content-Disposition: form-data; name="files"; filename="file2.png"
Content-Type: image/png

[文件2二进制数据]
--boundary123--
```

**响应示例:**
```json
{
  "data": {
    "successful": [
      {
        "id": 2,
        "filename": "file1_20231201_153100.jpg",
        "originalname": "file1.jpg",
        "url": "https://example.com/uploads/file1_20231201_153100.jpg"
      },
      {
        "id": 3,
        "filename": "file2_20231201_153100.png",
        "originalname": "file2.png",
        "url": "https://example.com/uploads/file2_20231201_153100.png"
      }
    ],
    "failed": []
  },
  "meta": {
    "total": 2,
    "successful": 2,
    "failed": 0
  }
}
```

### 3. 工作流 API

#### 触发工作流

**请求示例:**
```http
POST /api/workflows/1:trigger
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "context": {
    "userId": 1,
    "formData": {
      "title": "请假申请",
      "reason": "家庭事务",
      "startDate": "2023-12-05",
      "endDate": "2023-12-07"
    }
  },
  "sync": false
}
```

**响应示例:**
```json
{
  "data": {
    "executionId": "exec_20231201_153200_001",
    "workflowId": 1,
    "status": "started",
    "context": {
      "userId": 1,
      "formData": {
        "title": "请假申请",
        "reason": "家庭事务",
        "startDate": "2023-12-05",
        "endDate": "2023-12-07"
      }
    },
    "startedAt": "2023-12-01T15:32:00.000Z"
  }
}
```

#### 获取工作流执行状态

**请求示例:**
```http
GET /api/workflows/executions/exec_20231201_153200_001
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**响应示例:**
```json
{
  "data": {
    "id": "exec_20231201_153200_001",
    "workflowId": 1,
    "status": "running",
    "currentNode": "approval_node_1",
    "context": {
      "userId": 1,
      "formData": {
        "title": "请假申请",
        "reason": "家庭事务",
        "startDate": "2023-12-05",
        "endDate": "2023-12-07"
      }
    },
    "jobs": [
      {
        "id": "job_001",
        "nodeId": "start_node",
        "status": "resolved",
        "result": {},
        "startedAt": "2023-12-01T15:32:00.000Z",
        "endedAt": "2023-12-01T15:32:01.000Z"
      },
      {
        "id": "job_002",
        "nodeId": "approval_node_1",
        "status": "pending",
        "result": {
          "approvalTasks": [1, 2],
          "mode": "any"
        },
        "startedAt": "2023-12-01T15:32:01.000Z"
      }
    ],
    "startedAt": "2023-12-01T15:32:00.000Z"
  }
}
```

### 4. 数据统计 API

#### 获取仪表板数据

**请求示例:**
```http
POST /api/analytics:dashboard
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "widgets": [
    {
      "type": "metric",
      "config": {
        "collection": "users",
        "aggregation": "count",
        "filter": {
          "status": "active"
        }
      }
    },
    {
      "type": "chart",
      "config": {
        "collection": "orders",
        "chartType": "line",
        "xField": "createdAt",
        "yField": "amount",
        "groupBy": "day",
        "dateRange": {
          "start": "2023-11-01",
          "end": "2023-11-30"
        }
      }
    }
  ]
}
```

**响应示例:**
```json
{
  "data": {
    "widgets": [
      {
        "type": "metric",
        "result": {
          "value": 1250,
          "label": "活跃用户数",
          "trend": {
            "direction": "up",
            "percentage": 12.5,
            "period": "vs last month"
          }
        }
      },
      {
        "type": "chart",
        "result": {
          "data": [
            {
              "date": "2023-11-01",
              "amount": 15000
            },
            {
              "date": "2023-11-02",
              "amount": 18000
            }
          ],
          "summary": {
            "total": 450000,
            "average": 15000,
            "peak": 25000
          }
        }
      }
    ]
  }
}
```

## 错误处理和异常情况

### 标准错误码

```typescript
enum ErrorCodes {
  // 认证错误 (401)
  UNAUTHORIZED = 'UNAUTHORIZED',
  INVALID_TOKEN = 'INVALID_TOKEN',
  TOKEN_EXPIRED = 'TOKEN_EXPIRED',

  // 权限错误 (403)
  FORBIDDEN = 'FORBIDDEN',
  INSUFFICIENT_PERMISSIONS = 'INSUFFICIENT_PERMISSIONS',

  // 请求错误 (400)
  INVALID_PARAMETER = 'INVALID_PARAMETER',
  MISSING_PARAMETER = 'MISSING_PARAMETER',
  VALIDATION_ERROR = 'VALIDATION_ERROR',

  // 资源错误 (404)
  RESOURCE_NOT_FOUND = 'RESOURCE_NOT_FOUND',

  // 冲突错误 (409)
  DUPLICATE_RESOURCE = 'DUPLICATE_RESOURCE',
  RESOURCE_CONFLICT = 'RESOURCE_CONFLICT',

  // 服务器错误 (500)
  INTERNAL_ERROR = 'INTERNAL_ERROR',
  DATABASE_ERROR = 'DATABASE_ERROR',
  EXTERNAL_SERVICE_ERROR = 'EXTERNAL_SERVICE_ERROR',
}
```

### 错误响应格式

```typescript
interface ErrorResponse {
  errors: Array<{
    message: string;
    code: string;
    field?: string;
    details?: any;
  }>;
  meta?: {
    requestId: string;
    timestamp: string;
  };
}
```

### 常见错误示例

#### 认证失败 (401)

```json
{
  "errors": [
    {
      "message": "Authentication required",
      "code": "UNAUTHORIZED"
    }
  ],
  "meta": {
    "requestId": "req_20231201_153000_001",
    "timestamp": "2023-12-01T15:30:00.000Z"
  }
}
```

#### 权限不足 (403)

```json
{
  "errors": [
    {
      "message": "Insufficient permissions to access this resource",
      "code": "INSUFFICIENT_PERMISSIONS",
      "details": {
        "required": "users:update",
        "current": ["users:read"]
      }
    }
  ]
}
```

#### 验证错误 (422)

```json
{
  "errors": [
    {
      "message": "Email is required",
      "code": "VALIDATION_ERROR",
      "field": "email"
    },
    {
      "message": "Password must be at least 8 characters",
      "code": "VALIDATION_ERROR",
      "field": "password"
    }
  ]
}
```

#### 资源冲突 (409)

```json
{
  "errors": [
    {
      "message": "Email address already exists",
      "code": "DUPLICATE_RESOURCE",
      "field": "email",
      "details": {
        "conflictingId": 123
      }
    }
  ]
}
```

### 客户端错误处理

```typescript
// 统一错误处理函数
export const handleAPIError = (error: any) => {
  if (error.response) {
    const { status, data } = error.response;

    switch (status) {
      case 401:
        // 清除本地token，跳转到登录页
        localStorage.removeItem('token');
        window.location.href = '/login';
        break;

      case 403:
        message.error('权限不足');
        break;

      case 422:
        // 显示验证错误
        if (data.errors) {
          data.errors.forEach(err => {
            if (err.field) {
              message.error(`${err.field}: ${err.message}`);
            } else {
              message.error(err.message);
            }
          });
        }
        break;

      case 429:
        message.error('请求过于频繁，请稍后再试');
        break;

      case 500:
        message.error('服务器内部错误');
        break;

      default:
        message.error(data?.errors?.[0]?.message || '请求失败');
    }
  } else if (error.request) {
    message.error('网络连接失败');
  } else {
    message.error('请求配置错误');
  }
};

// 在API客户端中使用
apiClient.axios.interceptors.response.use(
  (response) => response,
  (error) => {
    handleAPIError(error);
    return Promise.reject(error);
  }
);
```

## 性能优化建议

### 1. 查询优化

#### 使用字段选择

```typescript
// 只获取需要的字段
const users = await api.resource('users').list({
  fields: ['id', 'name', 'email'], // 减少数据传输
  pageSize: 20,
});

// 排除敏感字段
const users = await api.resource('users').list({
  except: ['password', 'token'], // 排除敏感信息
});
```

#### 合理使用关联查询

```typescript
// 避免 N+1 查询问题
const posts = await api.resource('posts').list({
  appends: ['user', 'category'], // 一次性获取关联数据
  fields: ['id', 'title', 'content'],
  'user.fields': ['id', 'name'], // 限制关联字段
  'category.fields': ['id', 'name'],
});
```

#### 使用分页和排序

```typescript
// 合理的分页大小
const posts = await api.resource('posts').list({
  page: 1,
  pageSize: 20, // 避免过大的页面大小
  sort: ['-createdAt'], // 使用索引字段排序
});
```

### 2. 缓存策略

#### 客户端缓存

```typescript
// 使用 React Query 进行缓存
import { useQuery } from 'react-query';

const useUsers = (params) => {
  return useQuery(
    ['users', params],
    () => api.resource('users').list(params),
    {
      staleTime: 5 * 60 * 1000, // 5分钟内不重新请求
      cacheTime: 10 * 60 * 1000, // 缓存10分钟
    }
  );
};
```

#### 服务端缓存

```typescript
// 在后端使用 Redis 缓存
app.resource({
  name: 'users',
  actions: {
    list: {
      middleware: [
        async (ctx, next) => {
          const cacheKey = `users:list:${JSON.stringify(ctx.action.params)}`;
          const cached = await redis.get(cacheKey);

          if (cached) {
            ctx.body = JSON.parse(cached);
            return;
          }

          await next();

          // 缓存结果
          await redis.setex(cacheKey, 300, JSON.stringify(ctx.body));
        },
      ],
      handler: async (ctx, next) => {
        // 原始处理逻辑
      },
    },
  },
});
```

### 3. 批量操作

#### 批量创建

```typescript
// 批量创建用户
const users = await api.resource('users').createMany({
  records: [
    { name: 'User 1', email: 'user1@example.com' },
    { name: 'User 2', email: 'user2@example.com' },
    { name: 'User 3', email: 'user3@example.com' },
  ],
});
```

#### 批量更新

```typescript
// 批量更新状态
const result = await api.resource('users').batchUpdate({
  filter: {
    department: 'IT',
  },
  values: {
    status: 'active',
  },
});
```

### 4. 请求优化

#### 请求去重

```typescript
// 防止重复请求
const requestCache = new Map();

const deduplicateRequest = (key: string, requestFn: () => Promise<any>) => {
  if (requestCache.has(key)) {
    return requestCache.get(key);
  }

  const promise = requestFn().finally(() => {
    requestCache.delete(key);
  });

  requestCache.set(key, promise);
  return promise;
};

// 使用示例
const getUser = (id: number) => {
  return deduplicateRequest(`user:${id}`, () =>
    api.resource('users').get({ filterByTk: id })
  );
};
```

#### 请求合并

```typescript
// 合并多个单独的请求
const getUsersWithDetails = async (userIds: number[]) => {
  // 而不是多次单独请求
  // const users = await Promise.all(
  //   userIds.map(id => api.resource('users').get({ filterByTk: id }))
  // );

  // 使用批量查询
  const users = await api.resource('users').list({
    filter: {
      id: { $in: userIds },
    },
    appends: ['profile', 'department'],
  });

  return users.data;
};
```
