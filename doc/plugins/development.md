# 插件开发指南

## 概述

NocoBase 采用插件化架构，所有功能都通过插件实现。插件可以扩展前端界面、后端 API、数据模型等各个方面。本指南基于官方插件的实现模式，提供详细的开发范式和代码示例。

## 插件分类体系

### 按功能类型分类

1. **字段插件** - 扩展数据字段类型和界面组件
2. **工作流节点插件** - 扩展工作流执行节点
3. **配置界面插件** - 提供插件配置和管理界面
4. **区块插件** - 扩展页面展示区块
5. **页面路由插件** - 添加自定义页面和路由
6. **数据源连接器插件** - 连接外部数据源

### 按复杂度分类

1. **简单插件** - 单一功能，代码量小
2. **中等插件** - 多个相关功能组合
3. **复杂插件** - 完整的业务系统

## 插件结构

### 标准插件目录结构

```
my-plugin/
├── package.json          # 插件配置
├── README.md            # 插件说明
├── src/
│   ├── client/          # 前端代码
│   │   ├── index.tsx    # 前端入口
│   │   ├── components/  # 前端组件
│   │   ├── interfaces/  # 字段接口
│   │   ├── blocks/      # 区块组件
│   │   └── settings/    # 配置界面
│   ├── server/          # 后端代码
│   │   ├── index.ts     # 后端入口
│   │   ├── actions/     # 操作处理
│   │   ├── collections/ # 数据模型
│   │   ├── middlewares/ # 中间件
│   │   ├── instructions/# 工作流节点
│   │   └── triggers/    # 工作流触发器
│   ├── common/          # 共享代码
│   │   ├── constants.ts # 常量定义
│   │   └── types.ts     # 类型定义
│   └── locale/          # 国际化文件
│       ├── zh-CN.json
│       └── en-US.json
├── tests/               # 测试文件
│   ├── client/
│   └── server/
└── dist/                # 构建输出
```

### package.json 配置

```json
{
  "name": "@my-org/nocobase-plugin-my-plugin",
  "version": "1.0.0",
  "description": "My NocoBase Plugin",
  "main": "dist/server/index.js",
  "keywords": ["nocobase", "plugin"],
  "license": "MIT",
  "nocobase": {
    "displayName": "My Plugin",
    "description": "A sample plugin for NocoBase",
    "homepage": "https://github.com/my-org/my-plugin",
    "packageManager": "yarn"
  },
  "dependencies": {
    "@nocobase/client": "^1.0.0",
    "@nocobase/server": "^1.0.0"
  }
}
```

## 后端插件开发

### 基础插件类

```typescript
import { InstallOptions, Plugin } from '@nocobase/server';

export class MyPlugin extends Plugin {
  // 插件名称
  getName(): string {
    return 'my-plugin';
  }

  // 插件加载前
  async beforeLoad() {
    // 初始化配置
    this.app.logger.info('MyPlugin beforeLoad');
  }

  // 插件加载
  async load() {
    // 定义数据模型
    this.defineCollections();
    
    // 注册资源
    this.defineResources();
    
    // 注册中间件
    this.defineMiddlewares();
    
    // 注册事件监听器
    this.defineEventListeners();
  }

  // 插件加载后
  async afterLoad() {
    this.app.logger.info('MyPlugin afterLoad');
  }

  // 插件安装
  async install(options?: InstallOptions) {
    // 数据库同步
    await this.db.sync();
    
    // 初始化数据
    await this.initializeData();
  }

  // 插件启用
  async enable() {
    this.app.logger.info('MyPlugin enabled');
  }

  // 插件禁用
  async disable() {
    this.app.logger.info('MyPlugin disabled');
  }

  // 插件卸载
  async remove() {
    // 清理数据
    await this.cleanup();
  }

  // 定义数据模型
  private defineCollections() {
    this.db.collection({
      name: 'my_records',
      fields: [
        {
          type: 'string',
          name: 'title',
          allowNull: false,
        },
        {
          type: 'text',
          name: 'description',
        },
        {
          type: 'string',
          name: 'status',
          defaultValue: 'active',
        },
        {
          type: 'belongsTo',
          name: 'user',
          target: 'users',
        },
      ],
    });
  }

  // 定义资源
  private defineResources() {
    this.app.resource({
      name: 'my_records',
      actions: {
        list: this.listRecords.bind(this),
        get: this.getRecord.bind(this),
        create: this.createRecord.bind(this),
        update: this.updateRecord.bind(this),
        destroy: this.destroyRecord.bind(this),
        // 自定义操作
        activate: this.activateRecord.bind(this),
        deactivate: this.deactivateRecord.bind(this),
      },
    });

    // 设置权限
    this.app.acl.allow('my_records', ['list', 'get'], 'loggedIn');
    this.app.acl.allow('my_records', ['create', 'update', 'destroy'], 'admin');
  }

  // 定义中间件
  private defineMiddlewares() {
    // 全局中间件
    this.app.use(async (ctx, next) => {
      // 添加自定义头部
      ctx.set('X-Plugin', 'my-plugin');
      await next();
    });
  }

  // 定义事件监听器
  private defineEventListeners() {
    // 监听数据库事件
    this.db.on('my_records.afterCreate', async (model, options) => {
      this.app.logger.info('New record created:', model.id);
      
      // 发送通知
      await this.sendNotification('record_created', {
        recordId: model.id,
        title: model.title,
      });
    });

    // 监听应用事件
    this.app.on('user.signin', async (user) => {
      this.app.logger.info('User signed in:', user.email);
    });
  }

  // 操作处理方法
  private async listRecords(ctx, next) {
    const repository = this.db.getRepository('my_records');
    const { filter, sort, page, pageSize } = ctx.action.params;

    const [data, count] = await repository.findAndCount({
      filter,
      sort,
      page,
      pageSize,
      appends: ['user'],
    });

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
  }

  private async createRecord(ctx, next) {
    const repository = this.db.getRepository('my_records');
    const { values } = ctx.action.params;

    // 设置创建者
    values.userId = ctx.state.currentUser?.id;

    const record = await repository.create({ values });
    ctx.body = record;

    await next();
  }

  private async activateRecord(ctx, next) {
    const repository = this.db.getRepository('my_records');
    const { filterByTk } = ctx.action.params;

    const [record] = await repository.update({
      filterByTk,
      values: { status: 'active' },
    });

    ctx.body = record;
    await next();
  }

  // 辅助方法
  private async initializeData() {
    const repository = this.db.getRepository('my_records');
    
    // 检查是否已有数据
    const count = await repository.count();
    if (count === 0) {
      // 创建初始数据
      await repository.create({
        values: {
          title: 'Welcome Record',
          description: 'This is a sample record created during plugin installation.',
          status: 'active',
        },
      });
    }
  }

  private async sendNotification(type: string, data: any) {
    // 实现通知逻辑
    this.app.logger.info('Notification sent:', { type, data });
  }

  private async cleanup() {
    // 清理插件数据
    const repository = this.db.getRepository('my_records');
    await repository.destroy({
      filter: {},
      force: true,
    });
  }
}

export default MyPlugin;
```

### 数据模型定义

```typescript
// collections/my-records.ts
export default {
  name: 'my_records',
  title: 'My Records',
  fields: [
    {
      name: 'id',
      type: 'bigInt',
      autoIncrement: true,
      primaryKey: true,
      allowNull: false,
      interface: 'id',
    },
    {
      name: 'title',
      type: 'string',
      allowNull: false,
      interface: 'input',
      uiSchema: {
        type: 'string',
        title: '标题',
        'x-component': 'Input',
        required: true,
      },
    },
    {
      name: 'description',
      type: 'text',
      interface: 'textarea',
      uiSchema: {
        type: 'string',
        title: '描述',
        'x-component': 'Input.TextArea',
      },
    },
    {
      name: 'status',
      type: 'string',
      defaultValue: 'active',
      interface: 'select',
      uiSchema: {
        type: 'string',
        title: '状态',
        'x-component': 'Select',
        enum: [
          { label: '激活', value: 'active' },
          { label: '禁用', value: 'inactive' },
        ],
      },
    },
    {
      name: 'user',
      type: 'belongsTo',
      target: 'users',
      foreignKey: 'userId',
      interface: 'm2o',
      uiSchema: {
        title: '用户',
        'x-component': 'AssociationField',
      },
    },
    {
      name: 'createdAt',
      type: 'date',
      field: 'created_at',
      interface: 'createdAt',
    },
    {
      name: 'updatedAt',
      type: 'date',
      field: 'updated_at',
      interface: 'updatedAt',
    },
  ],
};
```

## 前端插件开发

### 基础插件类

```typescript
import React from 'react';
import { Plugin } from '@nocobase/client';
import { Card, Button } from 'antd';

// 自定义组件
const MyComponent = () => {
  return (
    <Card title="My Plugin Component">
      <p>This is a custom component from my plugin.</p>
      <Button type="primary">Click Me</Button>
    </Card>
  );
};

// 区块初始化器
const MyBlockInitializer = (props) => {
  const { insert } = props;
  
  return (
    <Button
      onClick={() => {
        insert({
          type: 'void',
          'x-component': 'MyComponent',
          'x-component-props': {
            title: 'My Block',
          },
        });
      }}
    >
      Add My Block
    </Button>
  );
};

// 设置页面
const MyPluginSettings = () => {
  return (
    <Card title="My Plugin Settings">
      <p>Configure your plugin settings here.</p>
    </Card>
  );
};

export class MyClientPlugin extends Plugin {
  async load() {
    // 注册组件
    this.addComponents();
    
    // 添加路由
    this.addRoutes();
    
    // 添加区块初始化器
    this.addBlockInitializers();
    
    // 添加设置页面
    this.addSettings();
    
    // 添加国际化
    this.addTranslations();
  }

  private addComponents() {
    this.app.addComponents({
      MyComponent,
      MyBlockInitializer,
      MyPluginSettings,
    });
  }

  private addRoutes() {
    this.router.add('my-plugin', {
      path: '/my-plugin',
      Component: 'MyComponent',
    });
    
    this.router.add('my-plugin.detail', {
      path: '/my-plugin/:id',
      Component: 'MyComponent',
    });
  }

  private addBlockInitializers() {
    this.app.schemaInitializerManager.addItem('page:addBlock', 'otherBlocks.myBlock', {
      title: 'My Block',
      Component: 'MyBlockInitializer',
    });
  }

  private addSettings() {
    this.app.pluginSettingsManager.add('my-plugin', {
      title: 'My Plugin',
      icon: 'SettingOutlined',
      Component: 'MyPluginSettings',
      aclSnippet: 'pm.my-plugin.settings',
    });
  }

  private addTranslations() {
    this.app.i18n.addResources('zh-CN', 'my-plugin', {
      'My Plugin': '我的插件',
      'My Block': '我的区块',
      'Settings': '设置',
    });
    
    this.app.i18n.addResources('en-US', 'my-plugin', {
      'My Plugin': 'My Plugin',
      'My Block': 'My Block',
      'Settings': 'Settings',
    });
  }
}

export default MyClientPlugin;
```

### Schema 组件开发

```typescript
import React from 'react';
import { observer, useField, useFieldSchema } from '@formily/react';
import { Card, Table, Button } from 'antd';
import { useRequest, useAPIClient } from '@nocobase/client';

// 自定义表格组件
export const MyTable = observer((props) => {
  const field = useField();
  const schema = useFieldSchema();
  const api = useAPIClient();
  
  const { data, loading, refresh } = useRequest({
    resource: 'my_records',
    action: 'list',
    params: {
      pageSize: 20,
      sort: ['-createdAt'],
    },
  });

  const columns = [
    {
      title: 'ID',
      dataIndex: 'id',
      key: 'id',
    },
    {
      title: '标题',
      dataIndex: 'title',
      key: 'title',
    },
    {
      title: '状态',
      dataIndex: 'status',
      key: 'status',
      render: (status) => (
        <span style={{ color: status === 'active' ? 'green' : 'red' }}>
          {status === 'active' ? '激活' : '禁用'}
        </span>
      ),
    },
    {
      title: '操作',
      key: 'actions',
      render: (_, record) => (
        <Button
          size="small"
          onClick={() => handleToggleStatus(record)}
        >
          {record.status === 'active' ? '禁用' : '激活'}
        </Button>
      ),
    },
  ];

  const handleToggleStatus = async (record) => {
    const action = record.status === 'active' ? 'deactivate' : 'activate';
    
    try {
      await api.resource('my_records').action(action, {
        filterByTk: record.id,
      });
      refresh();
    } catch (error) {
      console.error('Failed to toggle status:', error);
    }
  };

  return (
    <Card title={props.title || 'My Records'}>
      <Table
        dataSource={data?.data || []}
        columns={columns}
        loading={loading}
        rowKey="id"
        pagination={{
          total: data?.meta?.count,
          pageSize: 20,
        }}
      />
    </Card>
  );
});

// 注册组件
export const MyTablePlugin = {
  Component: MyTable,
  schema: {
    type: 'void',
    'x-component': 'MyTable',
    'x-component-props': {
      title: 'My Records',
    },
  },
};
```

## 插件配置

### 环境变量配置

```typescript
// 在插件中使用环境变量
export class MyPlugin extends Plugin {
  async load() {
    const config = {
      apiKey: process.env.MY_PLUGIN_API_KEY,
      endpoint: process.env.MY_PLUGIN_ENDPOINT || 'https://api.example.com',
      timeout: parseInt(process.env.MY_PLUGIN_TIMEOUT) || 5000,
    };

    // 验证配置
    if (!config.apiKey) {
      throw new Error('MY_PLUGIN_API_KEY is required');
    }

    // 使用配置
    this.initializeService(config);
  }

  private initializeService(config) {
    // 初始化外部服务
  }
}
```

### 插件选项

```typescript
interface MyPluginOptions {
  enabled?: boolean;
  apiKey?: string;
  features?: {
    notifications?: boolean;
    analytics?: boolean;
  };
}

export class MyPlugin extends Plugin<MyPluginOptions> {
  async load() {
    const options = this.options || {};
    
    if (!options.enabled) {
      this.app.logger.info('MyPlugin is disabled');
      return;
    }

    if (options.features?.notifications) {
      this.enableNotifications();
    }

    if (options.features?.analytics) {
      this.enableAnalytics();
    }
  }

  private enableNotifications() {
    // 启用通知功能
  }

  private enableAnalytics() {
    // 启用分析功能
  }
}

// 使用插件时传递选项
app.pm.add(MyPlugin, {
  name: 'my-plugin',
  enabled: true,
  apiKey: 'your-api-key',
  features: {
    notifications: true,
    analytics: false,
  },
});
```

## 插件测试

### 单元测试

```typescript
import { MockServer, createMockServer } from '@nocobase/test';
import MyPlugin from '../server';

describe('MyPlugin', () => {
  let app: MockServer;

  beforeEach(async () => {
    app = await createMockServer({
      plugins: [MyPlugin],
    });
  });

  afterEach(async () => {
    await app.destroy();
  });

  it('should create record', async () => {
    const response = await app
      .agent()
      .post('/api/my_records')
      .send({
        title: 'Test Record',
        description: 'Test Description',
      })
      .expect(200);

    expect(response.body.title).toBe('Test Record');
  });

  it('should list records', async () => {
    // 创建测试数据
    await app.db.getRepository('my_records').create({
      values: {
        title: 'Test Record',
        description: 'Test Description',
      },
    });

    const response = await app
      .agent()
      .get('/api/my_records')
      .expect(200);

    expect(response.body.data).toHaveLength(1);
    expect(response.body.data[0].title).toBe('Test Record');
  });
});
```

### 集成测试

```typescript
import { Application } from '@nocobase/server';
import MyPlugin from '../server';

describe('MyPlugin Integration', () => {
  let app: Application;

  beforeAll(async () => {
    app = new Application({
      database: {
        dialect: 'sqlite',
        storage: ':memory:',
      },
      plugins: [MyPlugin],
    });

    await app.load();
    await app.install();
    await app.start();
  });

  afterAll(async () => {
    await app.stop();
    await app.destroy();
  });

  it('should handle complete workflow', async () => {
    // 测试完整的业务流程
    const agent = app.agent();

    // 1. 创建记录
    const createResponse = await agent
      .post('/api/my_records')
      .send({
        title: 'Workflow Test',
        description: 'Testing complete workflow',
      })
      .expect(200);

    const recordId = createResponse.body.id;

    // 2. 激活记录
    await agent
      .post(`/api/my_records/${recordId}:activate`)
      .expect(200);

    // 3. 验证状态
    const getResponse = await agent
      .get(`/api/my_records/${recordId}`)
      .expect(200);

    expect(getResponse.body.status).toBe('active');
  });
});
```

---

*本指南详细介绍了 NocoBase 插件开发的各个方面，从基础结构到高级功能，帮助开发者创建功能丰富的插件。*
