# 数据库操作 API

## 概述

NocoBase 基于 Sequelize ORM 构建了强大的数据库抽象层，提供了集合 (Collection) 和仓库 (Repository) 的概念来管理数据模型和操作。

## 集合 (Collection) 定义

### 基础集合定义

```typescript
import { Database } from '@nocobase/database';

const db = new Database(/* 配置 */);

// 定义用户集合
db.collection({
  name: 'users',
  fields: [
    {
      type: 'string',
      name: 'name',
      allowNull: false,
      validate: {
        len: [2, 50],
      },
    },
    {
      type: 'string',
      name: 'email',
      allowNull: false,
      unique: true,
      validate: {
        isEmail: true,
      },
    },
    {
      type: 'string',
      name: 'password',
      allowNull: false,
      validate: {
        len: [6, 100],
      },
    },
    {
      type: 'string',
      name: 'status',
      defaultValue: 'active',
      validate: {
        isIn: [['active', 'inactive', 'banned']],
      },
    },
    {
      type: 'date',
      name: 'lastLoginAt',
    },
  ],
});
```

### 字段类型

#### 基础类型

```typescript
// 字符串类型
{
  type: 'string',
  name: 'title',
  allowNull: false,
  defaultValue: '',
  validate: {
    len: [1, 255],
  },
}

// 文本类型
{
  type: 'text',
  name: 'content',
  allowNull: true,
}

// 整数类型
{
  type: 'integer',
  name: 'age',
  allowNull: false,
  validate: {
    min: 0,
    max: 150,
  },
}

// 浮点数类型
{
  type: 'float',
  name: 'price',
  allowNull: false,
  validate: {
    min: 0,
  },
}

// 布尔类型
{
  type: 'boolean',
  name: 'isActive',
  defaultValue: true,
}

// 日期类型
{
  type: 'date',
  name: 'createdAt',
  defaultValue: () => new Date(),
}

// JSON 类型
{
  type: 'json',
  name: 'metadata',
  defaultValue: {},
}

// 数组类型
{
  type: 'array',
  name: 'tags',
  defaultValue: [],
}
```

#### 关系类型

```typescript
// 一对一关系 (hasOne)
{
  type: 'hasOne',
  name: 'profile',
  target: 'profiles',
  foreignKey: 'userId',
}

// 属于关系 (belongsTo)
{
  type: 'belongsTo',
  name: 'user',
  target: 'users',
  foreignKey: 'userId',
}

// 一对多关系 (hasMany)
{
  type: 'hasMany',
  name: 'posts',
  target: 'posts',
  foreignKey: 'userId',
}

// 多对多关系 (belongsToMany)
{
  type: 'belongsToMany',
  name: 'roles',
  target: 'roles',
  through: 'user_roles',
  foreignKey: 'userId',
  otherKey: 'roleId',
}
```

### 集合选项

```typescript
db.collection({
  name: 'posts',
  tableName: 'blog_posts', // 自定义表名
  timestamps: true, // 自动添加 createdAt 和 updatedAt
  paranoid: true, // 软删除，添加 deletedAt
  underscored: true, // 使用下划线命名
  fields: [
    // 字段定义
  ],
  indexes: [
    // 索引定义
    {
      fields: ['title'],
    },
    {
      fields: ['userId', 'status'],
      name: 'idx_user_status',
    },
    {
      fields: ['createdAt'],
      name: 'idx_created_at',
    },
  ],
});
```

## 仓库 (Repository) 操作

### 获取仓库

```typescript
// 获取仓库实例
const userRepository = db.getRepository('users');
const postRepository = db.getRepository('posts');
```

### 查询操作

#### 基础查询

```typescript
// 查找所有记录
const users = await userRepository.find();

// 带条件查询
const activeUsers = await userRepository.find({
  filter: {
    status: 'active',
  },
});

// 复杂条件查询
const users = await userRepository.find({
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

// 排序
const users = await userRepository.find({
  sort: ['-createdAt', 'name'],
});

// 分页
const users = await userRepository.find({
  page: 1,
  pageSize: 20,
});

// 字段选择
const users = await userRepository.find({
  fields: ['id', 'name', 'email'],
});

// 排除字段
const users = await userRepository.find({
  except: ['password', 'token'],
});
```

#### 关联查询

```typescript
// 包含关联数据
const users = await userRepository.find({
  appends: ['profile', 'posts'],
});

// 嵌套关联
const users = await userRepository.find({
  appends: ['posts.comments.user'],
});

// 关联过滤
const users = await userRepository.find({
  filter: {
    'posts.status': 'published',
    'profile.verified': true,
  },
  appends: ['posts', 'profile'],
});
```

#### 单个记录查询

```typescript
// 根据主键查询
const user = await userRepository.findByPk(1);

// 根据条件查询单个记录
const user = await userRepository.findOne({
  filter: {
    email: 'user@example.com',
  },
});

// 查询或创建
const [user, created] = await userRepository.findOrCreate({
  filter: {
    email: 'user@example.com',
  },
  values: {
    name: 'New User',
    email: 'user@example.com',
  },
});
```

#### 计数和统计

```typescript
// 计数
const count = await userRepository.count();

// 带条件计数
const activeCount = await userRepository.count({
  filter: {
    status: 'active',
  },
});

// 查询并计数
const [users, count] = await userRepository.findAndCount({
  filter: {
    status: 'active',
  },
  page: 1,
  pageSize: 20,
});
```

### 创建操作

```typescript
// 创建单个记录
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

// 创建时包含关联数据
const user = await userRepository.create({
  values: {
    name: 'John Doe',
    email: 'john@example.com',
    profile: {
      bio: 'Software Developer',
      avatar: 'avatar.jpg',
    },
  },
});
```

### 更新操作

```typescript
// 根据主键更新
const [updatedUser] = await userRepository.update({
  filterByTk: 1,
  values: {
    name: 'Jane Doe',
    updatedAt: new Date(),
  },
});

// 根据条件更新
const count = await userRepository.update({
  filter: {
    status: 'pending',
  },
  values: {
    status: 'active',
  },
});

// 更新或创建
const [user, created] = await userRepository.updateOrCreate({
  filter: {
    email: 'user@example.com',
  },
  values: {
    name: 'Updated Name',
    lastLoginAt: new Date(),
  },
});
```

### 删除操作

```typescript
// 根据主键删除
await userRepository.destroy({
  filterByTk: 1,
});

// 根据条件删除
const count = await userRepository.destroy({
  filter: {
    status: 'inactive',
  },
});

// 强制删除 (忽略软删除)
await userRepository.destroy({
  filterByTk: 1,
  force: true,
});

// 恢复软删除的记录
await userRepository.restore({
  filterByTk: 1,
});
```

## 查询操作符

### 比较操作符

```typescript
// 等于
{ age: 25 }
{ age: { $eq: 25 } }

// 不等于
{ age: { $ne: 25 } }

// 大于
{ age: { $gt: 18 } }

// 大于等于
{ age: { $gte: 18 } }

// 小于
{ age: { $lt: 65 } }

// 小于等于
{ age: { $lte: 65 } }

// 在范围内
{ age: { $between: [18, 65] } }

// 不在范围内
{ age: { $notBetween: [18, 65] } }
```

### 包含操作符

```typescript
// 包含在数组中
{ status: { $in: ['active', 'pending'] } }

// 不包含在数组中
{ status: { $notIn: ['banned', 'deleted'] } }

// 模糊匹配
{ name: { $like: '%john%' } }

// 忽略大小写模糊匹配
{ name: { $iLike: '%JOHN%' } }

// 不匹配
{ name: { $notLike: '%spam%' } }

// 正则表达式匹配
{ name: { $regexp: '^[A-Z]' } }
```

### 空值操作符

```typescript
// 为空
{ deletedAt: { $null: true } }
{ deletedAt: null }

// 不为空
{ deletedAt: { $notNull: true } }
```

### 逻辑操作符

```typescript
// AND 操作
{
  $and: [
    { status: 'active' },
    { age: { $gte: 18 } }
  ]
}

// OR 操作
{
  $or: [
    { role: 'admin' },
    { verified: true }
  ]
}

// NOT 操作
{
  $not: {
    status: 'banned'
  }
}
```

## 事务处理

### 手动事务

```typescript
const transaction = await db.sequelize.transaction();

try {
  // 在事务中创建用户
  const user = await userRepository.create({
    values: {
      name: 'John Doe',
      email: 'john@example.com',
    },
    transaction,
  });

  // 在事务中创建用户资料
  await profileRepository.create({
    values: {
      userId: user.id,
      bio: 'Software Developer',
    },
    transaction,
  });

  // 提交事务
  await transaction.commit();
} catch (error) {
  // 回滚事务
  await transaction.rollback();
  throw error;
}
```

### 自动事务

```typescript
await db.sequelize.transaction(async (transaction) => {
  const user = await userRepository.create({
    values: {
      name: 'John Doe',
      email: 'john@example.com',
    },
    transaction,
  });

  await profileRepository.create({
    values: {
      userId: user.id,
      bio: 'Software Developer',
    },
    transaction,
  });
});
```

## 数据库迁移

### 创建迁移

```typescript
// migrations/20231201000000-create-users-table.ts
import { Migration } from '@nocobase/server';

export default class extends Migration {
  async up() {
    const { db } = this.context;
    
    await db.sequelize.getQueryInterface().createTable('users', {
      id: {
        type: db.sequelize.Sequelize.BIGINT,
        primaryKey: true,
        autoIncrement: true,
      },
      name: {
        type: db.sequelize.Sequelize.STRING,
        allowNull: false,
      },
      email: {
        type: db.sequelize.Sequelize.STRING,
        allowNull: false,
        unique: true,
      },
      password: {
        type: db.sequelize.Sequelize.STRING,
        allowNull: false,
      },
      status: {
        type: db.sequelize.Sequelize.STRING,
        defaultValue: 'active',
      },
      created_at: {
        type: db.sequelize.Sequelize.DATE,
        allowNull: false,
      },
      updated_at: {
        type: db.sequelize.Sequelize.DATE,
        allowNull: false,
      },
    });

    // 创建索引
    await db.sequelize.getQueryInterface().addIndex('users', ['email']);
    await db.sequelize.getQueryInterface().addIndex('users', ['status']);
  }

  async down() {
    const { db } = this.context;
    await db.sequelize.getQueryInterface().dropTable('users');
  }
}
```

### 运行迁移

```bash
# 运行所有待执行的迁移
yarn nocobase db:migrate

# 回滚最后一次迁移
yarn nocobase db:migrate:undo

# 回滚所有迁移
yarn nocobase db:migrate:undo:all
```

## 数据库事件

### 监听数据库事件

```typescript
// 监听模型事件
db.on('users.afterCreate', async (model, options) => {
  console.log('New user created:', model.id);
  
  // 发送欢迎邮件
  await sendWelcomeEmail(model.email);
});

db.on('users.beforeUpdate', async (model, options) => {
  // 记录更新日志
  console.log('User updating:', model.id);
});

db.on('users.afterDestroy', async (model, options) => {
  // 清理相关数据
  await cleanupUserData(model.id);
});

// 监听所有模型的事件
db.on('*.afterCreate', async (model, options) => {
  console.log('Record created in:', model.constructor.name);
});
```

### 自定义事件

```typescript
// 在插件中触发自定义事件
export class MyPlugin extends Plugin {
  async load() {
    this.app.resource({
      name: 'posts',
      actions: {
        publish: async (ctx, next) => {
          const repository = ctx.db.getRepository('posts');
          const post = await repository.update({
            filterByTk: ctx.action.params.filterByTk,
            values: { status: 'published' },
          });

          // 触发自定义事件
          ctx.db.emit('posts.published', post[0]);

          ctx.body = post[0];
          await next();
        },
      },
    });

    // 监听自定义事件
    this.db.on('posts.published', async (post) => {
      // 发送发布通知
      await this.sendPublishNotification(post);
    });
  }
}
```

## 性能优化

### 索引优化

```typescript
// 在集合定义中添加索引
db.collection({
  name: 'posts',
  fields: [
    // 字段定义
  ],
  indexes: [
    // 单字段索引
    {
      fields: ['title'],
    },
    // 复合索引
    {
      fields: ['userId', 'status'],
      name: 'idx_user_status',
    },
    // 唯一索引
    {
      fields: ['slug'],
      unique: true,
    },
    // 部分索引
    {
      fields: ['email'],
      where: {
        status: 'active',
      },
    },
  ],
});
```

### 查询优化

```typescript
// 使用字段选择减少数据传输
const users = await userRepository.find({
  fields: ['id', 'name', 'email'], // 只选择需要的字段
});

// 使用分页避免大量数据
const users = await userRepository.find({
  page: 1,
  pageSize: 20,
});

// 合理使用关联查询
const posts = await postRepository.find({
  appends: ['user'], // 只包含需要的关联
  filter: {
    status: 'published',
  },
});

// 使用原生 SQL 进行复杂查询
const results = await db.sequelize.query(`
  SELECT u.name, COUNT(p.id) as post_count
  FROM users u
  LEFT JOIN posts p ON u.id = p.user_id
  WHERE u.status = 'active'
  GROUP BY u.id, u.name
  ORDER BY post_count DESC
  LIMIT 10
`, {
  type: db.sequelize.QueryTypes.SELECT,
});
```

---

*本文档详细介绍了 NocoBase 数据库操作的各个方面，从基础的 CRUD 操作到高级的事务处理和性能优化。*
