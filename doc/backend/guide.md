# 后端开发指南

## 概述

NocoBase 后端基于 Node.js + Koa.js + TypeScript 构建，采用插件化架构和资源管理器模式，提供强大的扩展能力。

## 技术栈详解

### 核心依赖

```json
{
  "koa": "^2.15.4",
  "@koa/router": "^13.1.0",
  "@koa/cors": "^5.0.0",
  "sequelize": "^6.x",
  "typescript": "5.1.3",
  "axios": "^1.7.0",
  "ws": "^8.13.0",
  "dayjs": "^1.11.8"
}
```

### 项目结构

```
packages/core/server/src/
├── application.ts        # 应用核心
├── plugin-manager/       # 插件管理
├── gateway/             # 网关服务
├── commands/            # 命令行工具
├── helpers/             # 辅助工具
└── middlewares/         # 中间件
```

## 应用初始化

### 创建应用实例

```typescript
import { Application } from '@nocobase/server';

const app = new Application({
  database: {
    dialect: 'postgres',
    host: 'localhost',
    port: 5432,
    username: 'nocobase',
    password: 'password',
    database: 'nocobase',
    logging: console.log,
  },
  resourcer: {
    prefix: '/api',
  },
  plugins: [],
});

// 启动应用
app.runAsCLI();
```

### 应用配置选项

```typescript
interface ApplicationOptions {
  database?: DatabaseOptions;
  resourcer?: {
    prefix?: string;
  };
  plugins?: string[];
  acl?: boolean;
  logger?: LoggerOptions;
  cors?: CorsOptions;
}
```

## 插件开发

### 插件基础结构

```typescript
import { InstallOptions, Plugin } from '@nocobase/server';

export class MyPlugin extends Plugin {
  // 插件加载前
  async beforeLoad() {
    // 初始化配置
  }

  // 插件加载
  async load() {
    // 注册资源
    this.app.resource({
      name: 'my-resource',
      actions: {
        list: this.list.bind(this),
        get: this.get.bind(this),
        create: this.create.bind(this),
        update: this.update.bind(this),
        destroy: this.destroy.bind(this),
      },
    });

    // 注册中间件
    this.app.use(this.middleware.bind(this));

    // 定义数据库集合
    this.db.collection({
      name: 'my_table',
      fields: [
        {
          type: 'string',
          name: 'name',
          allowNull: false,
        },
        {
          type: 'text',
          name: 'description',
        },
      ],
    });
  }

  // 插件安装
  async install(options?: InstallOptions) {
    // 数据库迁移
    await this.db.sync();
  }

  // 插件启用
  async enable() {
    // 启用逻辑
  }

  // 插件禁用
  async disable() {
    // 禁用逻辑
  }

  // 自定义方法
  private async list(ctx, next) {
    const { page = 1, pageSize = 20 } = ctx.action.params;
    const repository = this.db.getRepository('my_table');
    
    const [data, count] = await repository.findAndCount({
      offset: (page - 1) * pageSize,
      limit: pageSize,
    });

    ctx.body = {
      data,
      meta: {
        count,
        page,
        pageSize,
        totalPage: Math.ceil(count / pageSize),
      },
    };
    
    await next();
  }

  private async middleware(ctx, next) {
    // 中间件逻辑
    console.log(`${ctx.method} ${ctx.path}`);
    await next();
  }
}

export default MyPlugin;
```

### 插件注册

```typescript
// 在应用中注册插件
app.pm.add(MyPlugin, {
  name: 'my-plugin',
  // 插件配置选项
});

// 或者通过字符串注册
app.pm.add('@my-org/my-plugin');
```

## 资源管理器

### 资源定义

```typescript
// 定义独立资源
app.resource({
  name: 'posts',
  actions: {
    list: async (ctx, next) => {
      const repository = ctx.db.getRepository('posts');
      const { filter, sort, page, pageSize } = ctx.action.params;
      
      const data = await repository.find({
        filter,
        sort,
        page,
        pageSize,
      });
      
      ctx.body = data;
      await next();
    },
    
    get: async (ctx, next) => {
      const repository = ctx.db.getRepository('posts');
      const { filterByTk } = ctx.action.params;
      
      const data = await repository.findOne({
        filterByTk,
      });
      
      ctx.body = data;
      await next();
    },
    
    create: async (ctx, next) => {
      const repository = ctx.db.getRepository('posts');
      const { values } = ctx.action.params;
      
      const data = await repository.create({
        values,
      });
      
      ctx.body = data;
      await next();
    },
    
    update: async (ctx, next) => {
      const repository = ctx.db.getRepository('posts');
      const { filterByTk, values } = ctx.action.params;
      
      const data = await repository.update({
        filterByTk,
        values,
      });
      
      ctx.body = data;
      await next();
    },
    
    destroy: async (ctx, next) => {
      const repository = ctx.db.getRepository('posts');
      const { filterByTk } = ctx.action.params;
      
      await repository.destroy({
        filterByTk,
      });
      
      ctx.body = 'ok';
      await next();
    },
  },
});
```

### 关系资源

```typescript
// 定义关系资源
app.resource({
  name: 'posts.comments',
  actions: {
    list: async (ctx, next) => {
      const repository = ctx.db.getRepository('comments');
      const { associatedIndex } = ctx.action.params;
      
      const data = await repository.find({
        filter: {
          postId: associatedIndex,
        },
      });
      
      ctx.body = data;
      await next();
    },
  },
});
```

### 中间件

```typescript
// 全局中间件
app.use(async (ctx, next) => {
  console.log(`${ctx.method} ${ctx.path}`);
  await next();
});

// 资源中间件
app.resource({
  name: 'posts',
  middleware: [
    async (ctx, next) => {
      // 权限检查
      if (!ctx.state.currentUser) {
        ctx.throw(401, 'Unauthorized');
      }
      await next();
    },
  ],
  actions: {
    // 操作定义
  },
});

// 操作中间件
app.resource({
  name: 'posts',
  actions: {
    create: {
      middleware: [
        async (ctx, next) => {
          // 数据验证
          const { values } = ctx.action.params;
          if (!values.title) {
            ctx.throw(400, 'Title is required');
          }
          await next();
        },
      ],
      handler: async (ctx, next) => {
        // 创建逻辑
        await next();
      },
    },
  },
});
```

## 数据库操作

### 集合定义

```typescript
// 定义集合
db.collection({
  name: 'posts',
  fields: [
    {
      type: 'string',
      name: 'title',
      allowNull: false,
    },
    {
      type: 'text',
      name: 'content',
    },
    {
      type: 'string',
      name: 'status',
      defaultValue: 'draft',
    },
    {
      type: 'belongsTo',
      name: 'user',
      target: 'users',
      foreignKey: 'userId',
    },
    {
      type: 'hasMany',
      name: 'comments',
      target: 'comments',
      foreignKey: 'postId',
    },
    {
      type: 'belongsToMany',
      name: 'tags',
      target: 'tags',
      through: 'post_tags',
    },
  ],
});
```

### 仓库操作

```typescript
// 获取仓库
const postRepository = db.getRepository('posts');

// 查询操作
const posts = await postRepository.find({
  filter: {
    status: 'published',
  },
  sort: ['-createdAt'],
  page: 1,
  pageSize: 10,
  appends: ['user', 'comments'],
});

// 创建记录
const post = await postRepository.create({
  values: {
    title: 'Hello World',
    content: 'This is my first post.',
    userId: 1,
  },
});

// 更新记录
await postRepository.update({
  filterByTk: 1,
  values: {
    title: 'Updated Title',
  },
});

// 删除记录
await postRepository.destroy({
  filterByTk: 1,
});

// 关联操作
const post = await postRepository.findOne({
  filterByTk: 1,
  appends: ['user', 'comments.user'],
});
```

### 事务处理

```typescript
// 使用事务
await db.sequelize.transaction(async (transaction) => {
  const postRepository = db.getRepository('posts');
  const userRepository = db.getRepository('users');
  
  // 创建用户
  const user = await userRepository.create({
    values: { name: 'John' },
    transaction,
  });
  
  // 创建文章
  await postRepository.create({
    values: {
      title: 'My Post',
      userId: user.id,
    },
    transaction,
  });
});
```

## 认证授权

### JWT 认证

```typescript
import jwt from 'jsonwebtoken';

// 生成 Token
const generateToken = (user) => {
  return jwt.sign(
    { 
      userId: user.id,
      email: user.email,
    },
    process.env.JWT_SECRET,
    { expiresIn: '7d' }
  );
};

// 验证 Token
const verifyToken = (token) => {
  try {
    return jwt.verify(token, process.env.JWT_SECRET);
  } catch (error) {
    throw new Error('Invalid token');
  }
};

// 认证中间件
app.use(async (ctx, next) => {
  const token = ctx.get('Authorization')?.replace('Bearer ', '');
  
  if (token) {
    try {
      const payload = verifyToken(token);
      const userRepository = ctx.db.getRepository('users');
      const user = await userRepository.findByPk(payload.userId);
      ctx.state.currentUser = user;
    } catch (error) {
      ctx.throw(401, 'Invalid token');
    }
  }
  
  await next();
});
```

### ACL 权限控制

```typescript
// 定义权限
app.acl.define({
  role: 'user',
  resource: 'posts',
  actions: ['list', 'get'],
});

app.acl.define({
  role: 'admin',
  resource: 'posts',
  actions: '*',
});

// 权限检查中间件
app.use(async (ctx, next) => {
  const { resourceName, actionName } = ctx.action;
  const userRole = ctx.state.currentUser?.role || 'anonymous';
  
  const allowed = app.acl.can({
    role: userRole,
    resource: resourceName,
    action: actionName,
  });
  
  if (!allowed) {
    ctx.throw(403, 'Forbidden');
  }
  
  await next();
});
```

## WebSocket 支持

### WebSocket 服务器

```typescript
import WebSocket from 'ws';

// 创建 WebSocket 服务器
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws) => {
  console.log('Client connected');
  
  ws.on('message', (message) => {
    console.log('Received:', message);
    
    // 广播消息
    wss.clients.forEach((client) => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(message);
      }
    });
  });
  
  ws.on('close', () => {
    console.log('Client disconnected');
  });
});
```

### 实时通知

```typescript
// 在插件中集成 WebSocket
export class NotificationPlugin extends Plugin {
  private wss: WebSocket.Server;
  
  async load() {
    // 创建 WebSocket 服务器
    this.wss = new WebSocket.Server({ 
      server: this.app.server 
    });
    
    // 监听数据库事件
    this.db.on('posts.afterCreate', (model) => {
      this.broadcast('post:created', {
        id: model.id,
        title: model.title,
      });
    });
  }
  
  private broadcast(event: string, data: any) {
    const message = JSON.stringify({ event, data });
    
    this.wss.clients.forEach((client) => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(message);
      }
    });
  }
}
```

## 错误处理

### 全局错误处理

```typescript
// 错误处理中间件
app.use(async (ctx, next) => {
  try {
    await next();
  } catch (error) {
    ctx.status = error.status || 500;
    ctx.body = {
      error: {
        message: error.message,
        code: error.code,
        ...(process.env.NODE_ENV === 'development' && {
          stack: error.stack,
        }),
      },
    };
    
    // 记录错误日志
    app.logger.error(error);
  }
});
```

### 自定义错误类

```typescript
export class ValidationError extends Error {
  constructor(message: string, public field?: string) {
    super(message);
    this.name = 'ValidationError';
  }
}

export class NotFoundError extends Error {
  constructor(resource: string) {
    super(`${resource} not found`);
    this.name = 'NotFoundError';
  }
}

// 使用自定义错误
const post = await postRepository.findOne({ filterByTk: id });
if (!post) {
  throw new NotFoundError('Post');
}
```

## 日志记录

### 日志配置

```typescript
import { createLogger } from '@nocobase/logger';

const logger = createLogger({
  level: 'info',
  format: 'json',
  transports: [
    {
      type: 'console',
    },
    {
      type: 'file',
      filename: 'app.log',
    },
  ],
});

// 在应用中使用
app.logger = logger;
```

### 日志使用

```typescript
// 在插件中使用日志
export class MyPlugin extends Plugin {
  async load() {
    this.app.logger.info('Plugin loaded', { plugin: this.name });
    
    this.app.resource({
      name: 'posts',
      actions: {
        create: async (ctx, next) => {
          try {
            // 业务逻辑
            this.app.logger.info('Post created', { 
              userId: ctx.state.currentUser?.id,
              postId: result.id,
            });
          } catch (error) {
            this.app.logger.error('Failed to create post', {
              error: error.message,
              userId: ctx.state.currentUser?.id,
            });
            throw error;
          }
          
          await next();
        },
      },
    });
  }
}
```

---

*本指南涵盖了 NocoBase 后端开发的核心概念和最佳实践，帮助开发者快速上手后端开发。*
