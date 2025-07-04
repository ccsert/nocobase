# 前端 API 参考

## APIClient

### 概述

APIClient 是 NocoBase 前端的核心 HTTP 客户端，基于 Axios 封装，提供了资源操作和常规 HTTP 请求的统一接口。

### 初始化

```typescript
import { APIClient } from '@nocobase/client';

// 基础初始化
const apiClient = new APIClient();

// 带配置初始化
const apiClient = new APIClient({
  baseURL: '/api/',
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// 使用现有 Axios 实例
import axios from 'axios';
const instance = axios.create({ baseURL: '/api/' });
const apiClient = new APIClient(instance);
```

### 配置选项

```typescript
interface APIClientOptions {
  baseURL?: string;
  timeout?: number;
  headers?: Record<string, string>;
  storageType?: 'localStorage' | 'sessionStorage';
  storagePrefix?: string;
}
```

### 常用操作符

NocoBase 支持丰富的查询操作符：

```typescript
// 比较操作符
{
  age: { $eq: 25 },        // 等于
  age: { $ne: 25 },        // 不等于
  age: { $gt: 18 },        // 大于
  age: { $gte: 18 },       // 大于等于
  age: { $lt: 65 },        // 小于
  age: { $lte: 65 },       // 小于等于
}

// 包含操作符
{
  name: { $like: '%john%' },     // 模糊匹配
  name: { $iLike: '%JOHN%' },    // 忽略大小写模糊匹配
  tags: { $in: ['tech', 'news'] }, // 包含在数组中
  tags: { $notIn: ['spam'] },    // 不包含在数组中
}

// 逻辑操作符
{
  $and: [
    { status: 'active' },
    { age: { $gte: 18 } }
  ],
  $or: [
    { role: 'admin' },
    { verified: true }
  ],
  $not: {
    status: 'banned'
  }
}

// 空值操作符
{
  deletedAt: { $null: true },     // 为空
  deletedAt: { $notNull: true },  // 不为空
}
```

## 资源操作 API

### 基础资源操作

```typescript
// 获取资源实例
const userResource = apiClient.resource('users');
const postResource = apiClient.resource('posts');

// 列表查询
const response = await userResource.list({
  page: 1,
  pageSize: 20,
  sort: ['-createdAt'],
  filter: {
    status: 'active',
  },
  appends: ['profile'],
});

// 获取单个资源
const user = await userResource.get({
  filterByTk: 1,
  appends: ['posts', 'profile'],
});

// 创建资源
const newUser = await userResource.create({
  values: {
    name: 'John Doe',
    email: 'john@example.com',
    password: 'password123',
  },
});

// 更新资源
const updatedUser = await userResource.update({
  filterByTk: 1,
  values: {
    name: 'Jane Doe',
    email: 'jane@example.com',
  },
});

// 删除资源
await userResource.destroy({
  filterByTk: 1,
});

// 批量删除
await userResource.destroy({
  filter: {
    status: 'inactive',
  },
});
```

### 关系资源操作

```typescript
// 获取关系资源实例
const commentResource = apiClient.resource('posts.comments', 1);

// 获取文章的评论列表
const comments = await commentResource.list({
  sort: ['-createdAt'],
  appends: ['user'],
});

// 为文章创建评论
const newComment = await commentResource.create({
  values: {
    content: 'Great post!',
    userId: 2,
  },
});

// 更新评论
await commentResource.update({
  filterByTk: 1,
  values: {
    content: 'Updated comment',
  },
});
```

### 自定义操作

```typescript
// 调用自定义操作
const result = await userResource.action('customAction', {
  param1: 'value1',
  param2: 'value2',
});

// 等价于
const result = await apiClient.request({
  url: 'users:customAction',
  method: 'POST',
  data: {
    param1: 'value1',
    param2: 'value2',
  },
});
```

## HTTP 请求 API

### 基础请求

```typescript
// GET 请求
const response = await apiClient.request({
  url: '/users',
  method: 'GET',
  params: {
    page: 1,
    pageSize: 20,
  },
});

// POST 请求
const response = await apiClient.request({
  url: '/users',
  method: 'POST',
  data: {
    name: 'John Doe',
    email: 'john@example.com',
  },
});

// PUT 请求
const response = await apiClient.request({
  url: '/users/1',
  method: 'PUT',
  data: {
    name: 'Jane Doe',
  },
});

// DELETE 请求
await apiClient.request({
  url: '/users/1',
  method: 'DELETE',
});
```

### 请求拦截器

```typescript
// 请求拦截器
apiClient.axios.interceptors.request.use(
  (config) => {
    // 添加认证头
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);

// 响应拦截器
apiClient.axios.interceptors.response.use(
  (response) => {
    return response;
  },
  (error) => {
    if (error.response?.status === 401) {
      // 处理认证失败
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

## React Hooks API

### useAPIClient

```typescript
import { useAPIClient } from '@nocobase/client';

function MyComponent() {
  const api = useAPIClient();
  
  const handleSubmit = async (values) => {
    try {
      await api.resource('users').create({ values });
      message.success('创建成功');
    } catch (error) {
      message.error('创建失败');
    }
  };
  
  return <div>My Component</div>;
}
```

### useRequest

```typescript
import { useRequest } from '@nocobase/client';

function UserList() {
  // 基础用法
  const { data, loading, error, run, refresh } = useRequest({
    resource: 'users',
    action: 'list',
    params: {
      pageSize: 10,
      sort: ['-createdAt'],
    },
  });
  
  // 手动触发
  const { run: createUser } = useRequest(
    {
      resource: 'users',
      action: 'create',
    },
    {
      manual: true,
      onSuccess: () => {
        message.success('创建成功');
        refresh(); // 刷新列表
      },
    }
  );
  
  // 条件请求
  const { data: userDetail } = useRequest(
    {
      resource: 'users',
      action: 'get',
      params: { filterByTk: userId },
    },
    {
      ready: !!userId, // 只有当 userId 存在时才发起请求
    }
  );
  
  return (
    <div>
      {loading && <div>Loading...</div>}
      {error && <div>Error: {error.message}</div>}
      {data?.data?.map(user => (
        <div key={user.id}>{user.name}</div>
      ))}
      <button onClick={() => createUser({ values: { name: 'New User' } })}>
        创建用户
      </button>
    </div>
  );
}
```

### useResource

```typescript
import { useResource } from '@nocobase/client';

function PostComments({ postId }) {
  const commentResource = useResource('posts.comments', postId);
  
  const { data, loading } = useRequest({
    resource: commentResource,
    action: 'list',
  });
  
  const handleAddComment = async (content) => {
    await commentResource.create({
      values: { content },
    });
  };
  
  return (
    <div>
      {data?.data?.map(comment => (
        <div key={comment.id}>{comment.content}</div>
      ))}
    </div>
  );
}
```

## 查询参数

### 过滤器 (Filter)

```typescript
// 基础过滤
const users = await api.resource('users').list({
  filter: {
    status: 'active',
    age: { $gt: 18 },
  },
});

// 复杂过滤
const users = await api.resource('users').list({
  filter: {
    $and: [
      { status: 'active' },
      { 
        $or: [
          { age: { $gte: 18 } },
          { verified: true }
        ]
      }
    ],
  },
});

// 关联过滤
const posts = await api.resource('posts').list({
  filter: {
    'user.status': 'active',
    'comments.approved': true,
  },
});
```

### 排序 (Sort)

```typescript
// 单字段排序
const users = await api.resource('users').list({
  sort: ['name'], // 升序
});

const users = await api.resource('users').list({
  sort: ['-createdAt'], // 降序
});

// 多字段排序
const users = await api.resource('users').list({
  sort: ['status', '-createdAt'],
});
```

### 分页 (Pagination)

```typescript
// 基础分页
const users = await api.resource('users').list({
  page: 1,
  pageSize: 20,
});

// 偏移分页
const users = await api.resource('users').list({
  offset: 0,
  limit: 20,
});
```

### 关联查询 (Appends)

```typescript
// 包含关联数据
const users = await api.resource('users').list({
  appends: ['profile', 'posts'],
});

// 嵌套关联
const users = await api.resource('users').list({
  appends: ['posts.comments.user'],
});

// 关联过滤
const users = await api.resource('users').list({
  appends: ['posts'],
  filter: {
    'posts.status': 'published',
  },
});
```

### 字段选择 (Fields)

```typescript
// 选择特定字段
const users = await api.resource('users').list({
  fields: ['id', 'name', 'email'],
});

// 排除字段
const users = await api.resource('users').list({
  except: ['password', 'token'],
});
```

## 错误处理

### 错误类型

```typescript
interface APIError {
  message: string;
  code?: string;
  status?: number;
  errors?: ValidationError[];
}

interface ValidationError {
  field: string;
  message: string;
  code: string;
}
```

### 错误处理示例

```typescript
try {
  const user = await api.resource('users').create({
    values: { email: 'invalid-email' },
  });
} catch (error) {
  if (error.response?.status === 400) {
    // 验证错误
    const validationErrors = error.response.data.errors;
    validationErrors.forEach(err => {
      console.log(`${err.field}: ${err.message}`);
    });
  } else if (error.response?.status === 401) {
    // 认证错误
    console.log('请先登录');
  } else {
    // 其他错误
    console.log('操作失败:', error.message);
  }
}
```

## 缓存机制

### 请求缓存

```typescript
// 使用 useRequest 的缓存功能
const { data } = useRequest(
  {
    resource: 'users',
    action: 'list',
  },
  {
    cacheKey: 'user-list',
    staleTime: 5 * 60 * 1000, // 5分钟内不重新请求
  }
);
```

### 服务缓存

```typescript
// 使用 service 缓存
const api = useAPIClient();

// 第一次调用
const { data, loading } = api.service('user-list');

// 在其他组件中复用
function AnotherComponent() {
  const api = useAPIClient();
  const { data, refresh } = api.service('user-list'); // 复用缓存
  
  return (
    <div>
      <button onClick={refresh}>刷新</button>
      {/* 渲染数据 */}
    </div>
  );
}
```

## 最佳实践

### 1. 统一错误处理

```typescript
// 创建统一的 API 客户端
const createAPIClient = () => {
  const client = new APIClient({
    baseURL: '/api/',
  });
  
  // 统一错误处理
  client.axios.interceptors.response.use(
    (response) => response,
    (error) => {
      const { status, data } = error.response || {};
      
      switch (status) {
        case 401:
          message.error('请先登录');
          // 跳转到登录页
          break;
        case 403:
          message.error('权限不足');
          break;
        case 422:
          // 处理验证错误
          data.errors?.forEach(err => {
            message.error(`${err.field}: ${err.message}`);
          });
          break;
        default:
          message.error(data?.message || '操作失败');
      }
      
      return Promise.reject(error);
    }
  );
  
  return client;
};
```

### 2. 类型安全

```typescript
// 定义 API 响应类型
interface User {
  id: number;
  name: string;
  email: string;
  createdAt: string;
}

interface ListResponse<T> {
  data: T[];
  meta: {
    count: number;
    page: number;
    pageSize: number;
    totalPage: number;
  };
}

// 使用类型化的请求
const { data } = useRequest<ListResponse<User>>({
  resource: 'users',
  action: 'list',
});

// data 现在有完整的类型信息
data?.data.forEach(user => {
  console.log(user.name); // TypeScript 会提供智能提示
});
```

### 3. 请求优化

```typescript
// 使用防抖优化搜索
import { useDebounceFn } from 'ahooks';

function UserSearch() {
  const [keyword, setKeyword] = useState('');
  
  const { run: search } = useRequest(
    {
      resource: 'users',
      action: 'list',
      params: {
        filter: {
          name: { $like: `%${keyword}%` },
        },
      },
    },
    {
      manual: true,
    }
  );
  
  const { run: debouncedSearch } = useDebounceFn(search, {
    wait: 300,
  });
  
  useEffect(() => {
    if (keyword) {
      debouncedSearch();
    }
  }, [keyword, debouncedSearch]);
  
  return (
    <input
      value={keyword}
      onChange={(e) => setKeyword(e.target.value)}
      placeholder="搜索用户"
    />
  );
}
```

---

*本文档提供了 NocoBase 前端 API 的完整参考，帮助开发者高效地进行前端开发。*
