# 后端 API 参考

## 资源管理器 API

### 资源定义

```typescript
import { Application } from '@nocobase/server';

const app = new Application();

// 定义基础资源
app.resource({
  name: 'users',
  actions: {
    list: async (ctx, next) => {
      const repository = ctx.db.getRepository('users');
      const { filter, sort, page, pageSize, appends } = ctx.action.params;
      
      const options = {
        filter,
        sort,
        page,
        pageSize,
        appends,
      };
      
      const [data, count] = await repository.findAndCount(options);
      
      ctx.body = {
        data,
        meta: {
          count,
          page: page || 1,
          pageSize: pageSize || 20,
          totalPage: Math.ceil(count / (pageSize || 20)),
        },
      };
      
      await next();
    },
    
    get: async (ctx, next) => {
      const repository = ctx.db.getRepository('users');
      const { filterByTk, appends } = ctx.action.params;
      
      const data = await repository.findOne({
        filterByTk,
        appends,
      });
      
      if (!data) {
        ctx.throw(404, 'User not found');
      }
      
      ctx.body = data;
      await next();
    },
    
    create: async (ctx, next) => {
      const repository = ctx.db.getRepository('users');
      const { values } = ctx.action.params;
      
      const data = await repository.create({
        values,
      });
      
      ctx.body = data;
      await next();
    },
    
    update: async (ctx, next) => {
      const repository = ctx.db.getRepository('users');
      const { filterByTk, values } = ctx.action.params;
      
      const [data] = await repository.update({
        filterByTk,
        values,
      });
      
      ctx.body = data;
      await next();
    },
    
    destroy: async (ctx, next) => {
      const repository = ctx.db.getRepository('users');
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
  name: 'users.posts',
  actions: {
    list: async (ctx, next) => {
      const repository = ctx.db.getRepository('posts');
      const { associatedIndex, filter = {}, ...params } = ctx.action.params;
      
      // 添加关联过滤条件
      filter.userId = associatedIndex;
      
      const [data, count] = await repository.findAndCount({
        filter,
        ...params,
      });
      
      ctx.body = { data, meta: { count } };
      await next();
    },
    
    create: async (ctx, next) => {
      const repository = ctx.db.getRepository('posts');
      const { associatedIndex, values } = ctx.action.params;
      
      // 设置关联 ID
      values.userId = associatedIndex;
      
      const data = await repository.create({ values });
      ctx.body = data;
      await next();
    },
  },
});
```

### 自定义操作

```typescript
app.resource({
  name: 'users',
  actions: {
    // 自定义操作：激活用户
    activate: async (ctx, next) => {
      const repository = ctx.db.getRepository('users');
      const { filterByTk } = ctx.action.params;
      
      const user = await repository.findOne({ filterByTk });
      if (!user) {
        ctx.throw(404, 'User not found');
      }
      
      await repository.update({
        filterByTk,
        values: { status: 'active', activatedAt: new Date() },
      });
      
      ctx.body = { message: 'User activated successfully' };
      await next();
    },
    
    // 批量操作
    batchUpdate: async (ctx, next) => {
      const repository = ctx.db.getRepository('users');
      const { filter, values } = ctx.action.params;
      
      const count = await repository.update({
        filter,
        values,
      });
      
      ctx.body = { updated: count };
      await next();
    },
    
    // 统计操作
    statistics: async (ctx, next) => {
      const repository = ctx.db.getRepository('users');
      
      const total = await repository.count();
      const active = await repository.count({
        filter: { status: 'active' },
      });
      const inactive = await repository.count({
        filter: { status: 'inactive' },
      });
      
      ctx.body = {
        total,
        active,
        inactive,
        activeRate: total > 0 ? (active / total * 100).toFixed(2) : 0,
      };
      
      await next();
    },
  },
});
```

## 数据库 API

### Repository 操作

```typescript
// 获取 Repository
const userRepository = ctx.db.getRepository('users');
const postRepository = ctx.db.getRepository('posts');

// 查询操作
const users = await userRepository.find({
  filter: {
    status: 'active',
    age: { $gte: 18 },
  },
  sort: ['-createdAt'],
  page: 1,
  pageSize: 20,
  appends: ['profile', 'posts'],
});

// 查询单个记录
const user = await userRepository.findOne({
  filter: { email: 'user@example.com' },
  appends: ['profile'],
});

// 根据主键查询
const user = await userRepository.findByPk(1, {
  appends: ['posts'],
});

// 查询并计数
const [users, count] = await userRepository.findAndCount({
  filter: { status: 'active' },
  page: 1,
  pageSize: 20,
});

// 计数
const count = await userRepository.count({
  filter: { status: 'active' },
});

// 创建记录
const user = await userRepository.create({
  values: {
    name: 'John Doe',
    email: 'john@example.com',
    password: 'hashed_password',
  },
});

// 批量创建
const users = await userRepository.createMany({
  records: [
    { name: 'User 1', email: 'user1@example.com' },
    { name: 'User 2', email: 'user2@example.com' },
  ],
});

// 更新记录
const [updatedUser] = await userRepository.update({
  filterByTk: 1,
  values: {
    name: 'Jane Doe',
    updatedAt: new Date(),
  },
});

// 批量更新
const count = await userRepository.update({
  filter: { status: 'pending' },
  values: { status: 'active' },
});

// 删除记录
await userRepository.destroy({
  filterByTk: 1,
});

// 批量删除
const count = await userRepository.destroy({
  filter: { status: 'inactive' },
});
```

### 关联操作

```typescript
// 获取关联数据
const user = await userRepository.findOne({
  filterByTk: 1,
  appends: ['posts', 'profile', 'posts.comments'],
});

// 创建关联记录
const post = await postRepository.create({
  values: {
    title: 'My Post',
    content: 'Post content',
    userId: 1, // 设置外键
  },
});

// 通过关联创建
const user = await userRepository.findByPk(1);
const post = await user.createPost({
  title: 'My Post',
  content: 'Post content',
});

// 关联查询
const posts = await postRepository.find({
  filter: {
    'user.status': 'active',
    'comments.approved': true,
  },
  appends: ['user', 'comments'],
});
```

### 事务处理

```typescript
// 手动事务
const transaction = await ctx.db.sequelize.transaction();

try {
  const user = await userRepository.create({
    values: { name: 'John', email: 'john@example.com' },
    transaction,
  });
  
  await postRepository.create({
    values: { title: 'First Post', userId: user.id },
    transaction,
  });
  
  await transaction.commit();
} catch (error) {
  await transaction.rollback();
  throw error;
}

// 自动事务
await ctx.db.sequelize.transaction(async (transaction) => {
  const user = await userRepository.create({
    values: { name: 'John', email: 'john@example.com' },
    transaction,
  });
  
  await postRepository.create({
    values: { title: 'First Post', userId: user.id },
    transaction,
  });
});
```

## 中间件 API

### 全局中间件

```typescript
// 日志中间件
app.use(async (ctx, next) => {
  const start = Date.now();
  console.log(`--> ${ctx.method} ${ctx.path}`);
  
  await next();
  
  const ms = Date.now() - start;
  console.log(`<-- ${ctx.method} ${ctx.path} ${ctx.status} ${ms}ms`);
});

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
      },
    };
    
    // 记录错误日志
    console.error('API Error:', error);
  }
});

// 认证中间件
app.use(async (ctx, next) => {
  const token = ctx.get('Authorization')?.replace('Bearer ', '');
  
  if (token) {
    try {
      const payload = jwt.verify(token, process.env.JWT_SECRET);
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

### 资源中间件

```typescript
app.resource({
  name: 'posts',
  middleware: [
    // 权限检查中间件
    async (ctx, next) => {
      if (!ctx.state.currentUser) {
        ctx.throw(401, 'Authentication required');
      }
      await next();
    },
    
    // 数据验证中间件
    async (ctx, next) => {
      const { action } = ctx;
      
      if (['create', 'update'].includes(action.actionName)) {
        const { values } = ctx.action.params;
        
        if (!values.title) {
          ctx.throw(400, 'Title is required');
        }
        
        if (values.title.length < 3) {
          ctx.throw(400, 'Title must be at least 3 characters');
        }
      }
      
      await next();
    },
  ],
  actions: {
    // 操作定义
  },
});
```

### 操作中间件

```typescript
app.resource({
  name: 'posts',
  actions: {
    create: {
      middleware: [
        // 操作特定中间件
        async (ctx, next) => {
          const { values } = ctx.action.params;
          
          // 设置作者
          values.authorId = ctx.state.currentUser.id;
          
          // 设置默认状态
          if (!values.status) {
            values.status = 'draft';
          }
          
          await next();
        },
      ],
      handler: async (ctx, next) => {
        const repository = ctx.db.getRepository('posts');
        const { values } = ctx.action.params;
        
        const post = await repository.create({ values });
        ctx.body = post;
        
        await next();
      },
    },
    
    update: {
      middleware: [
        // 权限检查：只能编辑自己的文章
        async (ctx, next) => {
          const repository = ctx.db.getRepository('posts');
          const { filterByTk } = ctx.action.params;
          
          const post = await repository.findOne({ filterByTk });
          
          if (!post) {
            ctx.throw(404, 'Post not found');
          }
          
          if (post.authorId !== ctx.state.currentUser.id) {
            ctx.throw(403, 'Permission denied');
          }
          
          await next();
        },
      ],
      handler: async (ctx, next) => {
        // 更新逻辑
        await next();
      },
    },
  },
});
```

---

*本文档提供了 NocoBase 后端 API 的完整参考，涵盖了资源管理、数据库操作和中间件使用的各个方面。*
