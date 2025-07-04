# 插件开发示例

## 示例 1: 简单的 Hello World 插件

### 后端实现

```typescript
// src/server/index.ts
import { InstallOptions, Plugin } from '@nocobase/server';

export class HelloPlugin extends Plugin {
  async load() {
    // 定义简单的 API 端点
    this.app.resource({
      name: 'hello',
      actions: {
        sayHello: async (ctx, next) => {
          const { name = 'World' } = ctx.action.params;
          ctx.body = {
            message: `Hello, ${name}!`,
            timestamp: new Date().toISOString(),
          };
          await next();
        },
        
        getInfo: async (ctx, next) => {
          ctx.body = {
            plugin: 'hello-plugin',
            version: '1.0.0',
            description: 'A simple hello world plugin',
          };
          await next();
        },
      },
    });

    // 设置公开访问权限
    this.app.acl.allow('hello', '*', 'public');
  }

  async install(options?: InstallOptions) {
    // 插件安装时的初始化逻辑
    this.app.logger.info('Hello Plugin installed successfully');
  }
}

export default HelloPlugin;
```

### 前端实现

```typescript
// src/client/index.tsx
import React from 'react';
import { Plugin, useAPIClient } from '@nocobase/client';
import { Card, Button, Input, message } from 'antd';

const HelloComponent = () => {
  const [name, setName] = React.useState('');
  const [response, setResponse] = React.useState('');
  const api = useAPIClient();

  const handleSayHello = async () => {
    try {
      const result = await api.resource('hello').sayHello({ name });
      setResponse(result.data.message);
      message.success('Hello sent successfully!');
    } catch (error) {
      message.error('Failed to send hello');
    }
  };

  return (
    <Card title="Hello World Plugin" style={{ margin: 16 }}>
      <div style={{ marginBottom: 16 }}>
        <Input
          placeholder="Enter your name"
          value={name}
          onChange={(e) => setName(e.target.value)}
          style={{ width: 200, marginRight: 8 }}
        />
        <Button type="primary" onClick={handleSayHello}>
          Say Hello
        </Button>
      </div>
      {response && (
        <div style={{ padding: 16, background: '#f0f0f0', borderRadius: 4 }}>
          {response}
        </div>
      )}
    </Card>
  );
};

export class HelloClientPlugin extends Plugin {
  async load() {
    // 注册组件
    this.app.addComponents({
      HelloComponent,
    });

    // 添加路由
    this.router.add('hello', {
      path: '/hello',
      Component: 'HelloComponent',
    });

    // 添加到插件设置
    this.app.pluginSettingsManager.add('hello', {
      title: 'Hello Plugin',
      icon: 'SmileOutlined',
      Component: 'HelloComponent',
    });
  }
}

export default HelloClientPlugin;
```

## 示例 2: 任务管理插件

### 数据模型定义

```typescript
// src/server/collections/tasks.ts
export default {
  name: 'tasks',
  title: '任务',
  fields: [
    {
      name: 'id',
      type: 'bigInt',
      autoIncrement: true,
      primaryKey: true,
      allowNull: false,
    },
    {
      name: 'title',
      type: 'string',
      allowNull: false,
      validate: {
        len: [1, 200],
      },
    },
    {
      name: 'description',
      type: 'text',
    },
    {
      name: 'status',
      type: 'string',
      defaultValue: 'pending',
      validate: {
        isIn: [['pending', 'in_progress', 'completed', 'cancelled']],
      },
    },
    {
      name: 'priority',
      type: 'string',
      defaultValue: 'medium',
      validate: {
        isIn: [['low', 'medium', 'high', 'urgent']],
      },
    },
    {
      name: 'dueDate',
      type: 'date',
    },
    {
      name: 'assignee',
      type: 'belongsTo',
      target: 'users',
      foreignKey: 'assigneeId',
    },
    {
      name: 'creator',
      type: 'belongsTo',
      target: 'users',
      foreignKey: 'creatorId',
    },
    {
      name: 'tags',
      type: 'belongsToMany',
      target: 'tags',
      through: 'task_tags',
    },
    {
      name: 'comments',
      type: 'hasMany',
      target: 'task_comments',
      foreignKey: 'taskId',
    },
  ],
};
```

### 后端插件实现

```typescript
// src/server/index.ts
import { InstallOptions, Plugin } from '@nocobase/server';

export class TaskPlugin extends Plugin {
  async load() {
    // 加载数据模型
    await this.importCollections(path.resolve(__dirname, 'collections'));

    // 定义资源
    this.app.resource({
      name: 'tasks',
      actions: {
        // 自定义操作：更改任务状态
        changeStatus: async (ctx, next) => {
          const { filterByTk, status } = ctx.action.params;
          const repository = ctx.db.getRepository('tasks');

          const task = await repository.findOne({ filterByTk });
          if (!task) {
            ctx.throw(404, 'Task not found');
          }

          // 检查权限：只有分配者或创建者可以更改状态
          const currentUserId = ctx.state.currentUser?.id;
          if (task.assigneeId !== currentUserId && task.creatorId !== currentUserId) {
            ctx.throw(403, 'Permission denied');
          }

          const updatedTask = await repository.update({
            filterByTk,
            values: { 
              status,
              updatedAt: new Date(),
            },
          });

          // 发送通知
          await this.sendStatusChangeNotification(task, status);

          ctx.body = updatedTask;
          await next();
        },

        // 分配任务
        assign: async (ctx, next) => {
          const { filterByTk, assigneeId } = ctx.action.params;
          const repository = ctx.db.getRepository('tasks');

          const task = await repository.update({
            filterByTk,
            values: { 
              assigneeId,
              status: 'in_progress',
              updatedAt: new Date(),
            },
          });

          // 发送分配通知
          await this.sendAssignmentNotification(task[0], assigneeId);

          ctx.body = task[0];
          await next();
        },

        // 获取我的任务
        myTasks: async (ctx, next) => {
          const repository = ctx.db.getRepository('tasks');
          const currentUserId = ctx.state.currentUser?.id;

          if (!currentUserId) {
            ctx.throw(401, 'Authentication required');
          }

          const { status, priority } = ctx.action.params;
          const filter = {
            $or: [
              { assigneeId: currentUserId },
              { creatorId: currentUserId },
            ],
          };

          if (status) {
            filter.status = status;
          }
          if (priority) {
            filter.priority = priority;
          }

          const [data, count] = await repository.findAndCount({
            filter,
            sort: ['-updatedAt'],
            appends: ['assignee', 'creator', 'tags'],
          });

          ctx.body = { data, meta: { count } };
          await next();
        },

        // 任务统计
        statistics: async (ctx, next) => {
          const repository = ctx.db.getRepository('tasks');
          const currentUserId = ctx.state.currentUser?.id;

          const stats = {
            total: await repository.count({
              filter: { assigneeId: currentUserId },
            }),
            pending: await repository.count({
              filter: { assigneeId: currentUserId, status: 'pending' },
            }),
            inProgress: await repository.count({
              filter: { assigneeId: currentUserId, status: 'in_progress' },
            }),
            completed: await repository.count({
              filter: { assigneeId: currentUserId, status: 'completed' },
            }),
            overdue: await repository.count({
              filter: {
                assigneeId: currentUserId,
                status: { $in: ['pending', 'in_progress'] },
                dueDate: { $lt: new Date() },
              },
            }),
          };

          ctx.body = stats;
          await next();
        },
      },
    });

    // 设置权限
    this.app.acl.allow('tasks', ['list', 'get', 'myTasks', 'statistics'], 'loggedIn');
    this.app.acl.allow('tasks', ['create', 'update', 'destroy', 'changeStatus', 'assign'], 'loggedIn');

    // 注册事件监听器
    this.registerEventListeners();
  }

  private registerEventListeners() {
    // 任务创建后发送通知
    this.db.on('tasks.afterCreate', async (model, options) => {
      if (model.assigneeId) {
        await this.sendTaskCreatedNotification(model);
      }
    });

    // 任务更新后检查截止日期
    this.db.on('tasks.afterUpdate', async (model, options) => {
      if (model.dueDate && new Date(model.dueDate) < new Date()) {
        await this.handleOverdueTask(model);
      }
    });
  }

  private async sendStatusChangeNotification(task, newStatus) {
    // 实现状态变更通知逻辑
    this.app.logger.info(`Task ${task.id} status changed to ${newStatus}`);
  }

  private async sendAssignmentNotification(task, assigneeId) {
    // 实现任务分配通知逻辑
    this.app.logger.info(`Task ${task.id} assigned to user ${assigneeId}`);
  }

  private async sendTaskCreatedNotification(task) {
    // 实现任务创建通知逻辑
    this.app.logger.info(`New task created: ${task.title}`);
  }

  private async handleOverdueTask(task) {
    // 处理过期任务
    this.app.logger.warn(`Task ${task.id} is overdue`);
  }

  async install(options?: InstallOptions) {
    await this.db.sync();
    
    // 创建默认标签
    const tagRepository = this.db.getRepository('tags');
    const defaultTags = [
      { name: 'Bug', color: '#ff4d4f' },
      { name: 'Feature', color: '#52c41a' },
      { name: 'Enhancement', color: '#1890ff' },
      { name: 'Documentation', color: '#722ed1' },
    ];

    for (const tag of defaultTags) {
      await tagRepository.create({ values: tag });
    }
  }
}

export default TaskPlugin;
```

### 前端组件实现

```typescript
// src/client/components/TaskBoard.tsx
import React from 'react';
import { Card, Col, Row, Tag, Button, Modal, Form, Input, Select, DatePicker } from 'antd';
import { useRequest, useAPIClient } from '@nocobase/client';

const { TextArea } = Input;
const { Option } = Select;

const statusColors = {
  pending: 'orange',
  in_progress: 'blue',
  completed: 'green',
  cancelled: 'red',
};

const priorityColors = {
  low: 'green',
  medium: 'orange',
  high: 'red',
  urgent: 'purple',
};

export const TaskBoard = () => {
  const [createModalVisible, setCreateModalVisible] = React.useState(false);
  const [form] = Form.useForm();
  const api = useAPIClient();

  const { data: tasks, loading, refresh } = useRequest({
    resource: 'tasks',
    action: 'myTasks',
  });

  const { run: createTask } = useRequest(
    {
      resource: 'tasks',
      action: 'create',
    },
    {
      manual: true,
      onSuccess: () => {
        setCreateModalVisible(false);
        form.resetFields();
        refresh();
      },
    }
  );

  const handleCreateTask = async (values) => {
    await createTask({ values });
  };

  const handleStatusChange = async (taskId, newStatus) => {
    try {
      await api.resource('tasks').changeStatus({
        filterByTk: taskId,
        status: newStatus,
      });
      refresh();
    } catch (error) {
      console.error('Failed to change status:', error);
    }
  };

  const groupedTasks = React.useMemo(() => {
    const groups = {
      pending: [],
      in_progress: [],
      completed: [],
      cancelled: [],
    };

    tasks?.data?.forEach(task => {
      groups[task.status]?.push(task);
    });

    return groups;
  }, [tasks]);

  const renderTaskCard = (task) => (
    <Card
      key={task.id}
      size="small"
      style={{ marginBottom: 8 }}
      actions={[
        <Button
          size="small"
          onClick={() => handleStatusChange(task.id, 'in_progress')}
          disabled={task.status === 'in_progress'}
        >
          开始
        </Button>,
        <Button
          size="small"
          onClick={() => handleStatusChange(task.id, 'completed')}
          disabled={task.status === 'completed'}
        >
          完成
        </Button>,
      ]}
    >
      <div>
        <h4>{task.title}</h4>
        <p>{task.description}</p>
        <div>
          <Tag color={priorityColors[task.priority]}>{task.priority}</Tag>
          {task.dueDate && (
            <Tag color={new Date(task.dueDate) < new Date() ? 'red' : 'blue'}>
              {new Date(task.dueDate).toLocaleDateString()}
            </Tag>
          )}
        </div>
        {task.assignee && (
          <div style={{ marginTop: 8 }}>
            分配给: {task.assignee.name}
          </div>
        )}
      </div>
    </Card>
  );

  const renderColumn = (status, title, tasks) => (
    <Col span={6} key={status}>
      <Card
        title={
          <span>
            {title}
            <Tag color={statusColors[status]} style={{ marginLeft: 8 }}>
              {tasks.length}
            </Tag>
          </span>
        }
        style={{ height: '100%' }}
      >
        {tasks.map(renderTaskCard)}
      </Card>
    </Col>
  );

  return (
    <div style={{ padding: 16 }}>
      <div style={{ marginBottom: 16 }}>
        <Button
          type="primary"
          onClick={() => setCreateModalVisible(true)}
        >
          创建任务
        </Button>
      </div>

      <Row gutter={16}>
        {renderColumn('pending', '待处理', groupedTasks.pending)}
        {renderColumn('in_progress', '进行中', groupedTasks.in_progress)}
        {renderColumn('completed', '已完成', groupedTasks.completed)}
        {renderColumn('cancelled', '已取消', groupedTasks.cancelled)}
      </Row>

      <Modal
        title="创建任务"
        open={createModalVisible}
        onCancel={() => setCreateModalVisible(false)}
        onOk={() => form.submit()}
      >
        <Form
          form={form}
          layout="vertical"
          onFinish={handleCreateTask}
        >
          <Form.Item
            name="title"
            label="标题"
            rules={[{ required: true, message: '请输入任务标题' }]}
          >
            <Input />
          </Form.Item>
          
          <Form.Item name="description" label="描述">
            <TextArea rows={4} />
          </Form.Item>
          
          <Form.Item name="priority" label="优先级" initialValue="medium">
            <Select>
              <Option value="low">低</Option>
              <Option value="medium">中</Option>
              <Option value="high">高</Option>
              <Option value="urgent">紧急</Option>
            </Select>
          </Form.Item>
          
          <Form.Item name="dueDate" label="截止日期">
            <DatePicker style={{ width: '100%' }} />
          </Form.Item>
        </Form>
      </Modal>
    </div>
  );
};
```

### 前端插件主文件

```typescript
// src/client/index.tsx
import { Plugin } from '@nocobase/client';
import { TaskBoard } from './components/TaskBoard';

export class TaskClientPlugin extends Plugin {
  async load() {
    // 注册组件
    this.app.addComponents({
      TaskBoard,
    });

    // 添加路由
    this.router.add('tasks', {
      path: '/tasks',
      Component: 'TaskBoard',
    });

    // 添加到主菜单
    this.app.pluginSettingsManager.add('tasks', {
      title: '任务管理',
      icon: 'CheckSquareOutlined',
      Component: 'TaskBoard',
    });

    // 添加国际化
    this.app.i18n.addResources('zh-CN', 'tasks', {
      'Tasks': '任务',
      'Task Management': '任务管理',
      'Create Task': '创建任务',
      'My Tasks': '我的任务',
    });
  }
}

export default TaskClientPlugin;
```

## 示例 3: 文件上传插件

### 后端实现

```typescript
// src/server/index.ts
import { InstallOptions, Plugin } from '@nocobase/server';
import multer from '@koa/multer';
import path from 'path';
import fs from 'fs-extra';

export class FileUploadPlugin extends Plugin {
  async load() {
    // 配置文件上传中间件
    const upload = multer({
      storage: multer.diskStorage({
        destination: (req, file, cb) => {
          const uploadDir = path.join(process.cwd(), 'storage/uploads');
          fs.ensureDirSync(uploadDir);
          cb(null, uploadDir);
        },
        filename: (req, file, cb) => {
          const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1E9);
          cb(null, file.fieldname + '-' + uniqueSuffix + path.extname(file.originalname));
        },
      }),
      limits: {
        fileSize: 10 * 1024 * 1024, // 10MB
      },
      fileFilter: (req, file, cb) => {
        // 允许的文件类型
        const allowedTypes = /jpeg|jpg|png|gif|pdf|doc|docx|txt/;
        const extname = allowedTypes.test(path.extname(file.originalname).toLowerCase());
        const mimetype = allowedTypes.test(file.mimetype);

        if (mimetype && extname) {
          return cb(null, true);
        } else {
          cb(new Error('不支持的文件类型'));
        }
      },
    });

    // 定义文件资源
    this.app.resource({
      name: 'files',
      actions: {
        upload: {
          middleware: [upload.single('file')],
          handler: async (ctx, next) => {
            if (!ctx.file) {
              ctx.throw(400, 'No file uploaded');
            }

            const repository = ctx.db.getRepository('files');
            const file = await repository.create({
              values: {
                filename: ctx.file.filename,
                originalName: ctx.file.originalname,
                mimetype: ctx.file.mimetype,
                size: ctx.file.size,
                path: ctx.file.path,
                uploadedBy: ctx.state.currentUser?.id,
              },
            });

            ctx.body = file;
            await next();
          },
        },

        download: async (ctx, next) => {
          const { filterByTk } = ctx.action.params;
          const repository = ctx.db.getRepository('files');
          
          const file = await repository.findOne({ filterByTk });
          if (!file) {
            ctx.throw(404, 'File not found');
          }

          if (!fs.existsSync(file.path)) {
            ctx.throw(404, 'File not found on disk');
          }

          ctx.set('Content-Disposition', `attachment; filename="${file.originalName}"`);
          ctx.set('Content-Type', file.mimetype);
          ctx.body = fs.createReadStream(file.path);
          
          await next();
        },
      },
    });

    // 设置权限
    this.app.acl.allow('files', ['upload', 'list', 'get'], 'loggedIn');
    this.app.acl.allow('files', 'download', 'public');
  }

  async install(options?: InstallOptions) {
    // 创建文件表
    this.db.collection({
      name: 'files',
      fields: [
        { name: 'id', type: 'bigInt', autoIncrement: true, primaryKey: true },
        { name: 'filename', type: 'string', allowNull: false },
        { name: 'originalName', type: 'string', allowNull: false },
        { name: 'mimetype', type: 'string' },
        { name: 'size', type: 'integer' },
        { name: 'path', type: 'string', allowNull: false },
        { name: 'uploadedBy', type: 'bigInt' },
        { name: 'createdAt', type: 'date' },
        { name: 'updatedAt', type: 'date' },
      ],
    });

    await this.db.sync();
  }
}

export default FileUploadPlugin;
```

### 前端实现

```typescript
// src/client/components/FileUpload.tsx
import React from 'react';
import { Upload, Button, List, message } from 'antd';
import { UploadOutlined, DownloadOutlined, DeleteOutlined } from '@ant-design/icons';
import { useRequest, useAPIClient } from '@nocobase/client';

export const FileUpload = () => {
  const api = useAPIClient();
  
  const { data: files, loading, refresh } = useRequest({
    resource: 'files',
    action: 'list',
    params: {
      sort: ['-createdAt'],
    },
  });

  const handleUpload = async (file) => {
    const formData = new FormData();
    formData.append('file', file);

    try {
      await api.request({
        url: '/files:upload',
        method: 'POST',
        data: formData,
        headers: {
          'Content-Type': 'multipart/form-data',
        },
      });
      
      message.success('文件上传成功');
      refresh();
    } catch (error) {
      message.error('文件上传失败');
    }

    return false; // 阻止默认上传行为
  };

  const handleDownload = (file) => {
    window.open(`/api/files/${file.id}:download`);
  };

  const handleDelete = async (file) => {
    try {
      await api.resource('files').destroy({
        filterByTk: file.id,
      });
      message.success('文件删除成功');
      refresh();
    } catch (error) {
      message.error('文件删除失败');
    }
  };

  return (
    <div style={{ padding: 16 }}>
      <Upload
        beforeUpload={handleUpload}
        showUploadList={false}
        multiple
      >
        <Button icon={<UploadOutlined />}>上传文件</Button>
      </Upload>

      <List
        loading={loading}
        dataSource={files?.data || []}
        renderItem={(file) => (
          <List.Item
            actions={[
              <Button
                icon={<DownloadOutlined />}
                onClick={() => handleDownload(file)}
              >
                下载
              </Button>,
              <Button
                icon={<DeleteOutlined />}
                danger
                onClick={() => handleDelete(file)}
              >
                删除
              </Button>,
            ]}
          >
            <List.Item.Meta
              title={file.originalName}
              description={`大小: ${(file.size / 1024).toFixed(2)} KB | 上传时间: ${new Date(file.createdAt).toLocaleString()}`}
            />
          </List.Item>
        )}
      />
    </div>
  );
};
```

这些示例展示了不同类型的插件开发模式：

1. **Hello World 插件**: 展示了最基础的插件结构和 API 定义
2. **任务管理插件**: 展示了复杂的业务逻辑、数据模型和前端组件开发
3. **文件上传插件**: 展示了文件处理、中间件使用和权限控制

每个示例都包含了完整的前后端实现，可以作为开发自定义插件的参考模板。

---

*这些示例涵盖了 NocoBase 插件开发的常见场景，帮助开发者快速理解和掌握插件开发技巧。*
