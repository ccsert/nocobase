# 前端开发指南

## 概述

NocoBase 前端基于 React 18 + TypeScript + Ant Design 构建，采用插件化架构，支持高度自定义和扩展。

## 技术栈详解

### 核心依赖

```json
{
  "react": "^18.0.0",
  "react-dom": "^18.0.0",
  "typescript": "5.1.3",
  "antd": "5.24.2",
  "@formily/antd-v5": "1.2.3",
  "@formily/core": "^2.2.27",
  "@formily/react": "^2.2.27",
  "axios": "^1.7.0",
  "react-router-dom": "^6.11.2"
}
```

### 项目结构

```
packages/core/client/src/
├── api-client/           # API 客户端
├── application/          # 应用核心
├── schema-component/     # Schema 组件系统
├── block-provider/       # 区块提供者
├── collection-manager/   # 集合管理
├── plugin-manager/       # 插件管理
├── route-switch/         # 路由切换
├── user/                 # 用户管理
└── locale/               # 国际化
```

## 应用初始化

### 创建应用实例

```typescript
import { Application } from '@nocobase/client';
import { NocoBaseClientPresetPlugin } from '@nocobase/preset-nocobase/client';

const app = new Application({
  apiClient: {
    baseURL: '/api/',
    storageType: 'localStorage',
    storagePrefix: 'NOCOBASE_',
  },
  publicPath: '/',
  plugins: [NocoBaseClientPresetPlugin],
  ws: {
    url: '',
    basename: '/ws',
  },
});

export default app;
```

### 应用配置选项

```typescript
interface ApplicationOptions {
  apiClient?: {
    baseURL?: string;
    storageType?: 'localStorage' | 'sessionStorage';
    storagePrefix?: string;
  };
  publicPath?: string;
  plugins?: any[];
  ws?: {
    url?: string;
    basename?: string;
  };
}
```

## API 客户端

### 基本使用

```typescript
import { useAPIClient } from '@nocobase/client';

function MyComponent() {
  const api = useAPIClient();
  
  // 常规 HTTP 请求
  const fetchData = async () => {
    const response = await api.request({
      url: '/users',
      method: 'GET',
    });
    return response.data;
  };
  
  // 资源操作
  const fetchUsers = async () => {
    const response = await api.resource('users').list({
      pageSize: 20,
      page: 1,
    });
    return response.data;
  };
  
  return <div>My Component</div>;
}
```

### useRequest Hook

```typescript
import { useRequest } from '@nocobase/client';

function UserList() {
  // 使用 useRequest 进行数据请求
  const { data, loading, error, run, refresh } = useRequest({
    resource: 'users',
    action: 'list',
    params: {
      pageSize: 10,
      sort: ['-createdAt'],
    },
  });
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;
  
  return (
    <div>
      <button onClick={refresh}>刷新</button>
      <ul>
        {data?.data?.map(user => (
          <li key={user.id}>{user.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

### 资源操作

```typescript
// 获取资源实例
const userResource = api.resource('users');
const postResource = api.resource('posts');
const commentResource = api.resource('posts.comments', 1); // 关系资源

// CRUD 操作
await userResource.list({ pageSize: 20 });
await userResource.get({ filterByTk: 1 });
await userResource.create({ values: { name: 'John' } });
await userResource.update({ filterByTk: 1, values: { name: 'Jane' } });
await userResource.destroy({ filterByTk: 1 });

// 关系资源操作
await commentResource.list(); // 获取文章 ID=1 的评论列表
await commentResource.create({ values: { content: 'Great post!' } });
```

## Schema 组件系统

### Schema 基础概念

Schema 是 NocoBase 前端的核心概念，用于描述界面结构和行为。

```typescript
const schema = {
  type: 'void',
  'x-component': 'Card',
  'x-component-props': {
    title: '用户列表',
  },
  properties: {
    table: {
      type: 'array',
      'x-component': 'Table',
      'x-component-props': {
        rowKey: 'id',
      },
      properties: {
        name: {
          type: 'void',
          'x-component': 'Table.Column',
          'x-component-props': {
            title: '姓名',
          },
          properties: {
            name: {
              'x-component': 'Input',
              'x-read-pretty': true,
            },
          },
        },
      },
    },
  },
};
```

### 使用 SchemaComponent

```typescript
import { SchemaComponent } from '@nocobase/client';

function MyPage() {
  return (
    <SchemaComponent
      schema={schema}
      scope={{
        // 自定义作用域变量
        customVariable: 'value',
      }}
      components={{
        // 自定义组件
        MyCustomComponent,
      }}
    />
  );
}
```

### 自定义组件

```typescript
import React from 'react';
import { observer, useField } from '@formily/react';

const MyCustomComponent = observer((props) => {
  const field = useField();
  
  return (
    <div className="my-custom-component">
      <h3>{props.title}</h3>
      <div>{field.value}</div>
    </div>
  );
});

// 注册组件
app.addComponents({
  MyCustomComponent,
});
```

## 表单开发

### Formily 表单

```typescript
import { FormProvider, FormLayout, FormItem, Input, Submit } from '@formily/antd-v5';
import { createForm } from '@formily/core';

const form = createForm();

function UserForm() {
  return (
    <FormProvider form={form}>
      <FormLayout layout="vertical">
        <FormItem
          name="name"
          title="姓名"
          required
          component={[Input]}
        />
        <FormItem
          name="email"
          title="邮箱"
          required
          component={[Input]}
        />
        <Submit onSubmit={console.log}>提交</Submit>
      </FormLayout>
    </FormProvider>
  );
}
```

### Schema 表单

```typescript
const formSchema = {
  type: 'object',
  properties: {
    name: {
      type: 'string',
      title: '姓名',
      'x-component': 'Input',
      'x-decorator': 'FormItem',
      required: true,
    },
    email: {
      type: 'string',
      title: '邮箱',
      'x-component': 'Input',
      'x-decorator': 'FormItem',
      'x-component-props': {
        type: 'email',
      },
      required: true,
    },
    submit: {
      type: 'void',
      'x-component': 'Submit',
      title: '提交',
    },
  },
};

function SchemaForm() {
  return <SchemaComponent schema={formSchema} />;
}
```

## 路由管理

### 路由配置

```typescript
import { RouteSwitch } from '@nocobase/client';

// 在插件中添加路由
class MyPlugin extends Plugin {
  async load() {
    this.router.add('my-page', {
      path: '/my-page',
      Component: MyPageComponent,
    });
    
    this.router.add('my-page.detail', {
      path: '/my-page/:id',
      Component: MyDetailComponent,
    });
  }
}
```

### 路由导航

```typescript
import { useNavigate, useParams } from 'react-router-dom';

function MyComponent() {
  const navigate = useNavigate();
  const params = useParams();
  
  const handleClick = () => {
    navigate('/my-page/123');
  };
  
  return (
    <div>
      <p>当前 ID: {params.id}</p>
      <button onClick={handleClick}>导航到详情页</button>
    </div>
  );
}
```

## 状态管理

### Context 使用

```typescript
import React, { createContext, useContext } from 'react';

const MyContext = createContext(null);

export const MyProvider = ({ children }) => {
  const [state, setState] = useState({});
  
  return (
    <MyContext.Provider value={{ state, setState }}>
      {children}
    </MyContext.Provider>
  );
};

export const useMyContext = () => {
  const context = useContext(MyContext);
  if (!context) {
    throw new Error('useMyContext must be used within MyProvider');
  }
  return context;
};
```

### 全局状态

```typescript
// 在应用中添加 Provider
app.use(MyProvider);

// 在组件中使用
function MyComponent() {
  const { state, setState } = useMyContext();
  
  return <div>{state.value}</div>;
}
```

## 国际化

### 添加翻译

```typescript
// 在插件中添加翻译
class MyPlugin extends Plugin {
  async load() {
    this.app.i18n.addResources('zh-CN', 'my-plugin', {
      'Hello': '你好',
      'Welcome': '欢迎',
    });
    
    this.app.i18n.addResources('en-US', 'my-plugin', {
      'Hello': 'Hello',
      'Welcome': 'Welcome',
    });
  }
}
```

### 使用翻译

```typescript
import { useTranslation } from 'react-i18next';

function MyComponent() {
  const { t } = useTranslation('my-plugin');
  
  return (
    <div>
      <h1>{t('Hello')}</h1>
      <p>{t('Welcome')}</p>
    </div>
  );
}
```

## 最佳实践

### 1. 组件开发规范

```typescript
// 使用 TypeScript 定义 Props
interface MyComponentProps {
  title: string;
  data?: any[];
  onAction?: (item: any) => void;
}

// 使用 React.memo 优化性能
const MyComponent = React.memo<MyComponentProps>(({ title, data, onAction }) => {
  // 使用 useCallback 优化回调函数
  const handleClick = useCallback((item) => {
    onAction?.(item);
  }, [onAction]);
  
  return (
    <div>
      <h2>{title}</h2>
      {data?.map(item => (
        <div key={item.id} onClick={() => handleClick(item)}>
          {item.name}
        </div>
      ))}
    </div>
  );
});
```

### 2. 错误处理

```typescript
import { ErrorBoundary } from 'react-error-boundary';

function ErrorFallback({ error, resetErrorBoundary }) {
  return (
    <div role="alert">
      <h2>出错了:</h2>
      <pre>{error.message}</pre>
      <button onClick={resetErrorBoundary}>重试</button>
    </div>
  );
}

function MyApp() {
  return (
    <ErrorBoundary FallbackComponent={ErrorFallback}>
      <MyComponent />
    </ErrorBoundary>
  );
}
```

### 3. 性能优化

```typescript
// 使用 React.lazy 进行代码分割
const LazyComponent = React.lazy(() => import('./LazyComponent'));

// 使用 Suspense 处理加载状态
function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <LazyComponent />
    </Suspense>
  );
}

// 使用 useMemo 缓存计算结果
function ExpensiveComponent({ data }) {
  const expensiveValue = useMemo(() => {
    return data.reduce((sum, item) => sum + item.value, 0);
  }, [data]);
  
  return <div>{expensiveValue}</div>;
}
```

---

*本指南涵盖了 NocoBase 前端开发的核心概念和最佳实践，帮助开发者快速上手前端开发。*
