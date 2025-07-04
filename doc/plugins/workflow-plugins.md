# 工作流插件开发

## 概述

工作流插件是 NocoBase 中最复杂的插件类型之一，用于扩展工作流引擎的功能。本文档基于官方工作流插件的实现模式，详细介绍如何开发自定义工作流节点、触发器和执行器。

## 工作流架构分析

### 核心组件

1. **触发器 (Trigger)** - 工作流的启动条件
2. **节点 (Instruction)** - 工作流的执行单元
3. **处理器 (Processor)** - 工作流的执行引擎
4. **作业 (Job)** - 节点执行的实例

### 官方工作流插件架构

```typescript
// packages/plugins/@nocobase/plugin-workflow/src/server/Plugin.ts
export default class PluginWorkflowServer extends Plugin {
  instructions: Registry<InstructionInterface> = new Registry();
  triggers: Registry<Trigger> = new Registry();
  functions: Registry<CustomFunction> = new Registry();
  enabledCache: Map<number, WorkflowModel> = new Map();

  async load() {
    // 注册核心指令
    this.instructions.register('condition', ConditionInstruction);
    this.instructions.register('calculation', CalculationInstruction);
    this.instructions.register('create', CreateInstruction);
    this.instructions.register('update', UpdateInstruction);
    this.instructions.register('destroy', DestroyInstruction);
    this.instructions.register('query', QueryInstruction);
    this.instructions.register('end', EndInstruction);

    // 注册核心触发器
    this.triggers.register('collection', CollectionTrigger);
    this.triggers.register('schedule', ScheduleTrigger);
  }
}
```

## 触发器开发

### 触发器基类

```typescript
// src/server/triggers/BaseTrigger.ts
import { Trigger } from '@nocobase/plugin-workflow/server';
import { WorkflowModel } from '@nocobase/plugin-workflow/server';

export abstract class BaseTrigger extends Trigger {
  constructor(public readonly workflow: Plugin) {
    super(workflow);
  }

  // 工作流启用时调用
  abstract on(workflow: WorkflowModel): void;

  // 工作流禁用时调用
  abstract off(workflow: WorkflowModel): void;

  // 验证事件是否应该触发工作流
  validateEvent(workflow: WorkflowModel, context: any, options: any): boolean {
    return true;
  }

  // 复制配置（用于工作流版本管理）
  duplicateConfig?(workflow: WorkflowModel, options: any): object;

  // 验证上下文数据
  validateContext?(values: any, workflow: WorkflowModel): null | void | { [key: string]: string };

  // 是否同步执行
  sync?: boolean = false;

  // 手动执行工作流
  execute?(workflow: WorkflowModel, values: any, options: any): void | Processor | Promise<void | Processor>;
}
```

### HTTP请求触发器示例

```typescript
// src/server/triggers/HttpTrigger.ts
import { BaseTrigger } from './BaseTrigger';
import { Context, Next } from 'koa';

export class HttpTrigger extends BaseTrigger {
  static TYPE = 'http';
  sync = true;

  private routes = new Map<string, WorkflowModel>();

  constructor(workflow: Plugin) {
    super(workflow);

    // 注册HTTP中间件
    this.workflow.app.use(this.middleware.bind(this));
  }

  on(workflow: WorkflowModel) {
    const { path, method = 'POST' } = workflow.config;
    if (!path) return;

    const routeKey = `${method.toUpperCase()}:${path}`;
    this.routes.set(routeKey, workflow);

    this.workflow.app.logger.info(`HTTP trigger registered: ${routeKey}`);
  }

  off(workflow: WorkflowModel) {
    const { path, method = 'POST' } = workflow.config;
    if (!path) return;

    const routeKey = `${method.toUpperCase()}:${path}`;
    this.routes.delete(routeKey);

    this.workflow.app.logger.info(`HTTP trigger unregistered: ${routeKey}`);
  }

  private async middleware(ctx: Context, next: Next) {
    // 检查是否匹配工作流路径
    const routeKey = `${ctx.method}:${ctx.path}`;
    const workflow = this.routes.get(routeKey);

    if (!workflow) {
      return next();
    }

    // 验证权限
    if (!this.validatePermission(ctx, workflow)) {
      ctx.throw(403, 'Permission denied');
    }

    // 验证请求数据
    const validationResult = this.validateRequest(ctx, workflow);
    if (validationResult) {
      ctx.throw(400, validationResult);
    }

    try {
      // 执行工作流
      const processor = await this.workflow.trigger(workflow, {
        request: {
          headers: ctx.headers,
          query: ctx.query,
          body: ctx.request.body,
          params: ctx.params,
        },
        user: ctx.state.currentUser,
      }, { sync: true });

      if (processor) {
        // 等待工作流执行完成
        await processor.start();

        // 返回执行结果
        ctx.body = {
          success: true,
          executionId: processor.execution.id,
          result: processor.execution.context,
        };
      } else {
        ctx.body = { success: false, message: 'Workflow not triggered' };
      }
    } catch (error) {
      ctx.throw(500, error.message);
    }
  }

  private validatePermission(ctx: Context, workflow: WorkflowModel): boolean {
    const { auth } = workflow.config;

    if (auth === 'none') {
      return true;
    }

    if (auth === 'bearer') {
      const token = ctx.get('Authorization')?.replace('Bearer ', '');
      return token === workflow.config.token;
    }

    if (auth === 'user') {
      return !!ctx.state.currentUser;
    }

    return false;
  }

  private validateRequest(ctx: Context, workflow: WorkflowModel): string | null {
    const { validation } = workflow.config;

    if (!validation) return null;

    // 简单的JSON Schema验证示例
    const { body } = ctx.request;
    const { required = [] } = validation;

    for (const field of required) {
      if (!body || body[field] === undefined) {
        return `Required field '${field}' is missing`;
      }
    }

    return null;
  }

  duplicateConfig(workflow: WorkflowModel): object {
    return {
      ...workflow.config,
      // 生成新的token
      token: workflow.config.auth === 'bearer' ? this.generateToken() : undefined,
    };
  }

  private generateToken(): string {
    return Math.random().toString(36).substring(2) + Date.now().toString(36);
  }

  execute(workflow: WorkflowModel, values: any, options: any) {
    // 手动执行时的逻辑
    return this.workflow.trigger(workflow, {
      request: values,
      user: options.user,
    }, { manually: true });
  }
}
```

### 邮件触发器示例

```typescript
// src/server/triggers/EmailTrigger.ts
import { BaseTrigger } from './BaseTrigger';
import * as nodemailer from 'nodemailer';

export class EmailTrigger extends BaseTrigger {
  static TYPE = 'email';
  sync = false;

  private transporter: nodemailer.Transporter;
  private emailServer: any; // IMAP/POP3服务器连接

  constructor(workflow: Plugin) {
    super(workflow);
    this.initializeEmailServer();
  }

  private async initializeEmailServer() {
    // 初始化邮件服务器连接
    // 这里使用IMAP监听新邮件
  }

  on(workflow: WorkflowModel) {
    const { emailConfig } = workflow.config;

    // 开始监听指定邮箱
    this.startEmailMonitoring(workflow, emailConfig);
  }

  off(workflow: WorkflowModel) {
    // 停止监听邮箱
    this.stopEmailMonitoring(workflow);
  }

  private startEmailMonitoring(workflow: WorkflowModel, config: any) {
    // 实现邮件监听逻辑
    // 当收到新邮件时触发工作流
  }

  private stopEmailMonitoring(workflow: WorkflowModel) {
    // 停止邮件监听
  }

  validateEvent(workflow: WorkflowModel, context: any): boolean {
    const { filters } = workflow.config;

    if (!filters) return true;

    // 验证邮件是否符合过滤条件
    const { from, subject, body } = context.email;

    if (filters.from && !from.includes(filters.from)) {
      return false;
    }

    if (filters.subject && !subject.includes(filters.subject)) {
      return false;
    }

    return true;
  }
}
```

## 工作流节点开发

### 节点基类

```typescript
// src/server/instructions/BaseInstruction.ts
import { Instruction } from '@nocobase/plugin-workflow/server';
import { FlowNodeModel, JobModel, Processor } from '@nocobase/plugin-workflow/server';

export abstract class BaseInstruction extends Instruction {
  // 节点执行方法
  abstract async run(node: FlowNodeModel, prevJob: JobModel, processor: Processor): Promise<any>;

  // 节点恢复方法（用于暂停后恢复）
  async resume?(node: FlowNodeModel, job: JobModel, processor: Processor): Promise<any>;

  // 节点测试方法
  async test?(config: any): Promise<any>;

  // 获取节点的可用变量
  getScope?(node: FlowNodeModel, processor: Processor): any;
}
```

### 审批节点示例

```typescript
// src/server/instructions/ApprovalInstruction.ts
import { BaseInstruction } from './BaseInstruction';
import { JOB_STATUS } from '@nocobase/plugin-workflow/server';

export class ApprovalInstruction extends BaseInstruction {
  static TYPE = 'approval';

  async run(node: FlowNodeModel, prevJob: JobModel, processor: Processor) {
    const { assignees, mode = 'any', timeout } = node.config;

    if (!assignees || assignees.length === 0) {
      throw new Error('Approval node requires at least one assignee');
    }

    // 创建审批任务
    const approvalTasks = await this.createApprovalTasks(node, assignees, processor);

    // 设置超时
    if (timeout) {
      this.setApprovalTimeout(node, timeout, processor);
    }

    // 返回待处理状态，等待审批
    return {
      status: JOB_STATUS.PENDING,
      result: {
        approvalTasks: approvalTasks.map(task => task.id),
        mode,
        createdAt: new Date(),
      },
    };
  }

  async resume(node: FlowNodeModel, job: JobModel, processor: Processor) {
    const { mode } = node.config;
    const { approvalTasks } = job.result;

    // 获取所有审批任务的状态
    const tasks = await this.getApprovalTasks(approvalTasks);
    const approvedTasks = tasks.filter(task => task.status === 'approved');
    const rejectedTasks = tasks.filter(task => task.status === 'rejected');

    // 根据审批模式判断结果
    let finalStatus = JOB_STATUS.PENDING;
    let result = job.result;

    if (mode === 'any' && approvedTasks.length > 0) {
      // 任意一人同意即通过
      finalStatus = JOB_STATUS.RESOLVED;
      result = { ...result, approved: true, approver: approvedTasks[0].assignee };
    } else if (mode === 'all' && approvedTasks.length === tasks.length) {
      // 所有人同意才通过
      finalStatus = JOB_STATUS.RESOLVED;
      result = { ...result, approved: true, approvers: approvedTasks.map(t => t.assignee) };
    } else if (rejectedTasks.length > 0) {
      // 有人拒绝
      finalStatus = JOB_STATUS.RESOLVED;
      result = { ...result, approved: false, rejecter: rejectedTasks[0].assignee };
    }

    return {
      status: finalStatus,
      result,
    };
  }

  private async createApprovalTasks(node: FlowNodeModel, assignees: any[], processor: Processor) {
    const tasks = [];

    for (const assignee of assignees) {
      const task = await processor.db.getRepository('approvalTasks').create({
        values: {
          workflowId: processor.execution.workflowId,
          executionId: processor.execution.id,
          nodeId: node.id,
          assigneeId: assignee.id,
          assigneeType: assignee.type, // user, role, department
          status: 'pending',
          title: this.getApprovalTitle(node, processor),
          description: this.getApprovalDescription(node, processor),
          data: processor.execution.context,
        },
      });

      tasks.push(task);

      // 发送审批通知
      await this.sendApprovalNotification(task, processor);
    }

    return tasks;
  }

  private async getApprovalTasks(taskIds: number[]) {
    return await this.db.getRepository('approvalTasks').find({
      filter: { id: { $in: taskIds } },
    });
  }

  private getApprovalTitle(node: FlowNodeModel, processor: Processor): string {
    const { title } = node.config;
    return processor.getParsedValue(title, node.id) || '审批申请';
  }

  private getApprovalDescription(node: FlowNodeModel, processor: Processor): string {
    const { description } = node.config;
    return processor.getParsedValue(description, node.id) || '';
  }

  private async sendApprovalNotification(task: any, processor: Processor) {
    // 发送审批通知（邮件、短信、站内信等）
    const notificationPlugin = processor.app.pm.get('notification');
    if (notificationPlugin) {
      await notificationPlugin.send({
        type: 'approval',
        recipient: task.assigneeId,
        title: task.title,
        content: task.description,
        data: {
          taskId: task.id,
          executionId: task.executionId,
        },
      });
    }
  }

  private setApprovalTimeout(node: FlowNodeModel, timeout: number, processor: Processor) {
    // 设置审批超时处理
    setTimeout(async () => {
      const job = await processor.db.getRepository('jobs').findOne({
        filter: {
          executionId: processor.execution.id,
          nodeId: node.id,
          status: JOB_STATUS.PENDING,
        },
      });

      if (job) {
        // 超时处理：自动拒绝或其他逻辑
        await this.handleApprovalTimeout(job, processor);
      }
    }, timeout * 1000);
  }

  private async handleApprovalTimeout(job: JobModel, processor: Processor) {
    // 更新作业状态为超时
    await job.update({
      status: JOB_STATUS.RESOLVED,
      result: {
        ...job.result,
        approved: false,
        reason: 'timeout',
      },
    });

    // 继续执行工作流
    await processor.resume(job);
  }

  async test(config: any) {
    const { assignees, mode, timeout } = config;

    // 验证配置
    if (!assignees || assignees.length === 0) {
      throw new Error('至少需要一个审批人');
    }

    if (!['any', 'all'].includes(mode)) {
      throw new Error('审批模式必须是 any 或 all');
    }

    if (timeout && (timeout < 60 || timeout > 86400)) {
      throw new Error('超时时间必须在60秒到24小时之间');
    }

    return { valid: true };
  }
}
```

### 数据处理节点示例

```typescript
// src/server/instructions/DataProcessInstruction.ts
import { BaseInstruction } from './BaseInstruction';
import { JOB_STATUS } from '@nocobase/plugin-workflow/server';

export class DataProcessInstruction extends BaseInstruction {
  static TYPE = 'dataProcess';

  async run(node: FlowNodeModel, prevJob: JobModel, processor: Processor) {
    const { operations } = node.config;

    if (!operations || operations.length === 0) {
      return { status: JOB_STATUS.RESOLVED, result: prevJob.result };
    }

    let result = prevJob.result;

    try {
      for (const operation of operations) {
        result = await this.executeOperation(operation, result, processor);
      }

      return {
        status: JOB_STATUS.RESOLVED,
        result,
      };
    } catch (error) {
      return {
        status: JOB_STATUS.ERROR,
        result: { error: error.message },
      };
    }
  }

  private async executeOperation(operation: any, data: any, processor: Processor) {
    const { type, config } = operation;

    switch (type) {
      case 'filter':
        return this.filterData(data, config);
      case 'transform':
        return this.transformData(data, config);
      case 'aggregate':
        return this.aggregateData(data, config);
      case 'sort':
        return this.sortData(data, config);
      case 'group':
        return this.groupData(data, config);
      default:
        throw new Error(`Unknown operation type: ${type}`);
    }
  }

  private filterData(data: any, config: any) {
    const { field, operator, value } = config;

    if (!Array.isArray(data)) {
      throw new Error('Filter operation requires array data');
    }

    return data.filter(item => {
      const fieldValue = this.getFieldValue(item, field);
      return this.compareValues(fieldValue, operator, value);
    });
  }

  private transformData(data: any, config: any) {
    const { mapping } = config;

    if (Array.isArray(data)) {
      return data.map(item => this.transformItem(item, mapping));
    } else {
      return this.transformItem(data, mapping);
    }
  }

  private transformItem(item: any, mapping: any) {
    const result = {};

    for (const [targetField, sourceField] of Object.entries(mapping)) {
      if (typeof sourceField === 'string') {
        result[targetField] = this.getFieldValue(item, sourceField);
      } else if (typeof sourceField === 'object' && sourceField.expression) {
        result[targetField] = this.evaluateExpression(sourceField.expression, item);
      }
    }

    return result;
  }

  private aggregateData(data: any, config: any) {
    const { groupBy, aggregations } = config;

    if (!Array.isArray(data)) {
      throw new Error('Aggregate operation requires array data');
    }

    if (groupBy) {
      const groups = this.groupBy(data, groupBy);
      const result = {};

      for (const [key, items] of Object.entries(groups)) {
        result[key] = this.calculateAggregations(items, aggregations);
      }

      return result;
    } else {
      return this.calculateAggregations(data, aggregations);
    }
  }

  private calculateAggregations(data: any[], aggregations: any) {
    const result = {};

    for (const [field, operation] of Object.entries(aggregations)) {
      const values = data.map(item => this.getFieldValue(item, field)).filter(v => v != null);

      switch (operation) {
        case 'count':
          result[`${field}_count`] = values.length;
          break;
        case 'sum':
          result[`${field}_sum`] = values.reduce((sum, val) => sum + Number(val), 0);
          break;
        case 'avg':
          result[`${field}_avg`] = values.reduce((sum, val) => sum + Number(val), 0) / values.length;
          break;
        case 'min':
          result[`${field}_min`] = Math.min(...values.map(Number));
          break;
        case 'max':
          result[`${field}_max`] = Math.max(...values.map(Number));
          break;
      }
    }

    return result;
  }

  private getFieldValue(obj: any, path: string) {
    return path.split('.').reduce((current, key) => current?.[key], obj);
  }

  private compareValues(fieldValue: any, operator: string, compareValue: any): boolean {
    switch (operator) {
      case 'eq': return fieldValue === compareValue;
      case 'ne': return fieldValue !== compareValue;
      case 'gt': return fieldValue > compareValue;
      case 'gte': return fieldValue >= compareValue;
      case 'lt': return fieldValue < compareValue;
      case 'lte': return fieldValue <= compareValue;
      case 'in': return Array.isArray(compareValue) && compareValue.includes(fieldValue);
      case 'contains': return String(fieldValue).includes(String(compareValue));
      default: return false;
    }
  }

  private evaluateExpression(expression: string, context: any): any {
    // 简单的表达式求值器
    // 实际项目中应该使用更安全的表达式引擎
    try {
      const func = new Function('context', `with(context) { return ${expression}; }`);
      return func(context);
    } catch (error) {
      throw new Error(`Expression evaluation failed: ${error.message}`);
    }
  }
}
```

### 通知节点示例

```typescript
// src/server/instructions/NotificationInstruction.ts
import { BaseInstruction } from './BaseInstruction';
import { JOB_STATUS } from '@nocobase/plugin-workflow/server';

export class NotificationInstruction extends BaseInstruction {
  static TYPE = 'notification';

  async run(node: FlowNodeModel, prevJob: JobModel, processor: Processor) {
    const { channels, recipients, template } = node.config;

    if (!channels || channels.length === 0) {
      throw new Error('Notification node requires at least one channel');
    }

    const results = [];

    for (const channel of channels) {
      try {
        const result = await this.sendNotification(
          channel,
          recipients,
          template,
          processor.execution.context,
          processor
        );
        results.push({ channel, success: true, result });
      } catch (error) {
        results.push({ channel, success: false, error: error.message });
      }
    }

    return {
      status: JOB_STATUS.RESOLVED,
      result: { notifications: results },
    };
  }

  private async sendNotification(
    channel: string,
    recipients: any[],
    template: any,
    context: any,
    processor: Processor
  ) {
    switch (channel) {
      case 'email':
        return this.sendEmail(recipients, template, context, processor);
      case 'sms':
        return this.sendSMS(recipients, template, context, processor);
      case 'webhook':
        return this.sendWebhook(recipients, template, context, processor);
      case 'inApp':
        return this.sendInAppNotification(recipients, template, context, processor);
      default:
        throw new Error(`Unsupported notification channel: ${channel}`);
    }
  }

  private async sendEmail(recipients: any[], template: any, context: any, processor: Processor) {
    const emailPlugin = processor.app.pm.get('email');
    if (!emailPlugin) {
      throw new Error('Email plugin not available');
    }

    const { subject, body } = this.renderTemplate(template, context);
    const results = [];

    for (const recipient of recipients) {
      try {
        await emailPlugin.send({
          to: recipient.email,
          subject,
          html: body,
        });
        results.push({ recipient: recipient.email, success: true });
      } catch (error) {
        results.push({ recipient: recipient.email, success: false, error: error.message });
      }
    }

    return results;
  }

  private async sendSMS(recipients: any[], template: any, context: any, processor: Processor) {
    const smsPlugin = processor.app.pm.get('sms');
    if (!smsPlugin) {
      throw new Error('SMS plugin not available');
    }

    const { body } = this.renderTemplate(template, context);
    const results = [];

    for (const recipient of recipients) {
      try {
        await smsPlugin.send({
          to: recipient.phone,
          message: body,
        });
        results.push({ recipient: recipient.phone, success: true });
      } catch (error) {
        results.push({ recipient: recipient.phone, success: false, error: error.message });
      }
    }

    return results;
  }

  private async sendWebhook(recipients: any[], template: any, context: any, processor: Processor) {
    const { body } = this.renderTemplate(template, context);
    const results = [];

    for (const recipient of recipients) {
      try {
        const response = await fetch(recipient.url, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            ...recipient.headers,
          },
          body: JSON.stringify({
            ...JSON.parse(body),
            context,
          }),
        });

        if (!response.ok) {
          throw new Error(`HTTP ${response.status}: ${response.statusText}`);
        }

        results.push({ recipient: recipient.url, success: true });
      } catch (error) {
        results.push({ recipient: recipient.url, success: false, error: error.message });
      }
    }

    return results;
  }

  private async sendInAppNotification(recipients: any[], template: any, context: any, processor: Processor) {
    const { subject, body } = this.renderTemplate(template, context);
    const notificationRepo = processor.db.getRepository('notifications');
    const results = [];

    for (const recipient of recipients) {
      try {
        await notificationRepo.create({
          values: {
            userId: recipient.id,
            title: subject,
            content: body,
            type: 'workflow',
            data: context,
            read: false,
          },
        });
        results.push({ recipient: recipient.id, success: true });
      } catch (error) {
        results.push({ recipient: recipient.id, success: false, error: error.message });
      }
    }

    return results;
  }

  private renderTemplate(template: any, context: any) {
    const { subject, body } = template;

    return {
      subject: this.interpolateString(subject, context),
      body: this.interpolateString(body, context),
    };
  }

  private interpolateString(template: string, context: any): string {
    if (!template) return '';

    return template.replace(/\{\{([^}]+)\}\}/g, (match, path) => {
      const value = this.getNestedValue(context, path.trim());
      return value != null ? String(value) : match;
    });
  }

  private getNestedValue(obj: any, path: string) {
    return path.split('.').reduce((current, key) => current?.[key], obj);
  }
}
```

## 前端工作流组件开发

### 节点配置组件

```typescript
// src/client/components/ApprovalConfig.tsx
import React from 'react';
import { Form, Select, InputNumber, Input, Switch, Button, Space } from 'antd';
import { PlusOutlined, DeleteOutlined } from '@ant-design/icons';
import { useTranslation } from 'react-i18next';

const { TextArea } = Input;
const { Option } = Select;

export const ApprovalConfig = ({ value, onChange }) => {
  const { t } = useTranslation();
  const [form] = Form.useForm();

  const handleValuesChange = (changedValues, allValues) => {
    onChange?.(allValues);
  };

  return (
    <Form
      form={form}
      layout="vertical"
      initialValues={value}
      onValuesChange={handleValuesChange}
    >
      <Form.Item
        name="title"
        label={t('Approval Title')}
        rules={[{ required: true, message: t('Please enter approval title') }]}
      >
        <Input placeholder={t('Enter approval title')} />
      </Form.Item>

      <Form.Item name="description" label={t('Description')}>
        <TextArea rows={3} placeholder={t('Enter approval description')} />
      </Form.Item>

      <Form.Item
        name="mode"
        label={t('Approval Mode')}
        rules={[{ required: true }]}
      >
        <Select>
          <Option value="any">{t('Any one approves')}</Option>
          <Option value="all">{t('All must approve')}</Option>
        </Select>
      </Form.Item>

      <Form.Item label={t('Assignees')}>
        <Form.List name="assignees">
          {(fields, { add, remove }) => (
            <>
              {fields.map(({ key, name, ...restField }) => (
                <Space key={key} style={{ display: 'flex', marginBottom: 8 }} align="baseline">
                  <Form.Item
                    {...restField}
                    name={[name, 'type']}
                    rules={[{ required: true, message: t('Select assignee type') }]}
                  >
                    <Select placeholder={t('Type')} style={{ width: 100 }}>
                      <Option value="user">{t('User')}</Option>
                      <Option value="role">{t('Role')}</Option>
                      <Option value="dept">{t('Department')}</Option>
                    </Select>
                  </Form.Item>

                  <Form.Item
                    {...restField}
                    name={[name, 'id']}
                    rules={[{ required: true, message: t('Select assignee') }]}
                  >
                    <AssigneeSelector />
                  </Form.Item>

                  <DeleteOutlined onClick={() => remove(name)} />
                </Space>
              ))}
              <Form.Item>
                <Button type="dashed" onClick={() => add()} block icon={<PlusOutlined />}>
                  {t('Add Assignee')}
                </Button>
              </Form.Item>
            </>
          )}
        </Form.List>
      </Form.Item>

      <Form.Item name="enableTimeout" valuePropName="checked">
        <Switch /> {t('Enable Timeout')}
      </Form.Item>

      <Form.Item
        noStyle
        shouldUpdate={(prevValues, currentValues) =>
          prevValues.enableTimeout !== currentValues.enableTimeout
        }
      >
        {({ getFieldValue }) =>
          getFieldValue('enableTimeout') ? (
            <Form.Item
              name="timeout"
              label={t('Timeout (seconds)')}
              rules={[{ required: true, min: 60, max: 86400 }]}
            >
              <InputNumber min={60} max={86400} style={{ width: '100%' }} />
            </Form.Item>
          ) : null
        }
      </Form.Item>
    </Form>
  );
};

// 审批人选择器组件
const AssigneeSelector = ({ value, onChange }) => {
  const { t } = useTranslation();

  // 这里应该根据类型动态加载用户、角色或部门数据
  const options = [
    { label: t('Admin'), value: 1 },
    { label: t('Manager'), value: 2 },
  ];

  return (
    <Select
      value={value}
      onChange={onChange}
      placeholder={t('Select assignee')}
      style={{ width: 200 }}
      showSearch
      filterOption={(input, option) =>
        option?.label?.toLowerCase().includes(input.toLowerCase())
      }
      options={options}
    />
  );
};
```

### 数据处理节点配置

```typescript
// src/client/components/DataProcessConfig.tsx
import React from 'react';
import { Form, Select, Input, Button, Space, Card } from 'antd';
import { PlusOutlined, DeleteOutlined } from '@ant-design/icons';

const { Option } = Select;

export const DataProcessConfig = ({ value, onChange }) => {
  const [form] = Form.useForm();

  const operationTypes = [
    { label: '过滤', value: 'filter' },
    { label: '转换', value: 'transform' },
    { label: '聚合', value: 'aggregate' },
    { label: '排序', value: 'sort' },
    { label: '分组', value: 'group' },
  ];

  const operators = [
    { label: '等于', value: 'eq' },
    { label: '不等于', value: 'ne' },
    { label: '大于', value: 'gt' },
    { label: '大于等于', value: 'gte' },
    { label: '小于', value: 'lt' },
    { label: '小于等于', value: 'lte' },
    { label: '包含', value: 'contains' },
    { label: '在列表中', value: 'in' },
  ];

  return (
    <Form
      form={form}
      layout="vertical"
      initialValues={value}
      onValuesChange={(_, allValues) => onChange?.(allValues)}
    >
      <Form.List name="operations">
        {(fields, { add, remove }) => (
          <>
            {fields.map(({ key, name, ...restField }) => (
              <Card
                key={key}
                size="small"
                title={`操作 ${name + 1}`}
                extra={<DeleteOutlined onClick={() => remove(name)} />}
                style={{ marginBottom: 16 }}
              >
                <Form.Item
                  {...restField}
                  name={[name, 'type']}
                  label="操作类型"
                  rules={[{ required: true }]}
                >
                  <Select options={operationTypes} />
                </Form.Item>

                <Form.Item
                  noStyle
                  shouldUpdate={(prev, curr) =>
                    prev.operations?.[name]?.type !== curr.operations?.[name]?.type
                  }
                >
                  {({ getFieldValue }) => {
                    const operationType = getFieldValue(['operations', name, 'type']);
                    return <OperationConfig type={operationType} name={name} operators={operators} />;
                  }}
                </Form.Item>
              </Card>
            ))}
            <Button type="dashed" onClick={() => add()} block icon={<PlusOutlined />}>
              添加操作
            </Button>
          </>
        )}
      </Form.List>
    </Form>
  );
};

const OperationConfig = ({ type, name, operators }) => {
  switch (type) {
    case 'filter':
      return (
        <>
          <Form.Item name={[name, 'config', 'field']} label="字段" rules={[{ required: true }]}>
            <Input placeholder="输入字段路径，如 user.name" />
          </Form.Item>
          <Form.Item name={[name, 'config', 'operator']} label="操作符" rules={[{ required: true }]}>
            <Select options={operators} />
          </Form.Item>
          <Form.Item name={[name, 'config', 'value']} label="值" rules={[{ required: true }]}>
            <Input placeholder="比较值" />
          </Form.Item>
        </>
      );

    case 'transform':
      return (
        <Form.Item name={[name, 'config', 'mapping']} label="字段映射">
          <MappingEditor />
        </Form.Item>
      );

    case 'aggregate':
      return (
        <>
          <Form.Item name={[name, 'config', 'groupBy']} label="分组字段">
            <Input placeholder="分组字段，可选" />
          </Form.Item>
          <Form.Item name={[name, 'config', 'aggregations']} label="聚合配置">
            <AggregationEditor />
          </Form.Item>
        </>
      );

    default:
      return null;
  }
};

const MappingEditor = ({ value = {}, onChange }) => {
  const handleChange = (key, val) => {
    onChange?.({ ...value, [key]: val });
  };

  return (
    <div>
      {Object.entries(value).map(([key, val], index) => (
        <Space key={index} style={{ display: 'flex', marginBottom: 8 }}>
          <Input
            placeholder="目标字段"
            value={key}
            onChange={(e) => {
              const newValue = { ...value };
              delete newValue[key];
              newValue[e.target.value] = val;
              onChange?.(newValue);
            }}
          />
          <Input
            placeholder="源字段"
            value={val}
            onChange={(e) => handleChange(key, e.target.value)}
          />
          <DeleteOutlined
            onClick={() => {
              const newValue = { ...value };
              delete newValue[key];
              onChange?.(newValue);
            }}
          />
        </Space>
      ))}
      <Button
        type="dashed"
        onClick={() => handleChange(`field_${Date.now()}`, '')}
        icon={<PlusOutlined />}
      >
        添加映射
      </Button>
    </div>
  );
};

const AggregationEditor = ({ value = {}, onChange }) => {
  const aggregationTypes = [
    { label: '计数', value: 'count' },
    { label: '求和', value: 'sum' },
    { label: '平均值', value: 'avg' },
    { label: '最小值', value: 'min' },
    { label: '最大值', value: 'max' },
  ];

  return (
    <div>
      {Object.entries(value).map(([field, operation], index) => (
        <Space key={index} style={{ display: 'flex', marginBottom: 8 }}>
          <Input
            placeholder="字段名"
            value={field}
            onChange={(e) => {
              const newValue = { ...value };
              delete newValue[field];
              newValue[e.target.value] = operation;
              onChange?.(newValue);
            }}
          />
          <Select
            placeholder="聚合类型"
            value={operation}
            onChange={(val) => onChange?.({ ...value, [field]: val })}
            options={aggregationTypes}
            style={{ width: 120 }}
          />
          <DeleteOutlined
            onClick={() => {
              const newValue = { ...value };
              delete newValue[field];
              onChange?.(newValue);
            }}
          />
        </Space>
      ))}
      <Button
        type="dashed"
        onClick={() => onChange?.({ ...value, [`field_${Date.now()}`]: 'count' })}
        icon={<PlusOutlined />}
      >
        添加聚合
      </Button>
    </div>
  );
};
```

## 插件集成和注册

### 工作流插件主文件

```typescript
// src/client/index.tsx
import { Plugin } from '@nocobase/client';
import { ApprovalConfig } from './components/ApprovalConfig';
import { DataProcessConfig } from './components/DataProcessConfig';
import { NotificationConfig } from './components/NotificationConfig';

export class WorkflowExtensionPlugin extends Plugin {
  async load() {
    // 获取工作流插件实例
    const workflowPlugin = this.app.pm.get('workflow');

    if (!workflowPlugin) {
      console.warn('Workflow plugin not found');
      return;
    }

    // 注册自定义节点
    workflowPlugin.registerInstruction('approval', {
      title: '{{t("Approval")}}',
      type: 'approval',
      group: 'extended',
      description: '{{t("Approval node for workflow")}}',
      fieldset: {
        title: '{{t("Approval")}}',
        config: {
          type: 'void',
          'x-component': 'ApprovalConfig',
        },
      },
      component: ApprovalConfig,
      useVariables(node, options) {
        return [
          {
            name: 'approved',
            title: '{{t("Approved")}}',
            children: [
              { name: 'approved', title: '{{t("Is Approved")}}' },
              { name: 'approver', title: '{{t("Approver")}}' },
              { name: 'comment', title: '{{t("Approval Comment")}}' },
            ],
          },
        ];
      },
    });

    workflowPlugin.registerInstruction('dataProcess', {
      title: '{{t("Data Process")}}',
      type: 'dataProcess',
      group: 'extended',
      description: '{{t("Process and transform data")}}',
      fieldset: {
        title: '{{t("Data Process")}}',
        config: {
          type: 'void',
          'x-component': 'DataProcessConfig',
        },
      },
      component: DataProcessConfig,
      useVariables(node, options) {
        return [
          {
            name: 'result',
            title: '{{t("Process Result")}}',
            children: [
              { name: 'data', title: '{{t("Processed Data")}}' },
              { name: 'count', title: '{{t("Record Count")}}' },
            ],
          },
        ];
      },
    });

    workflowPlugin.registerInstruction('notification', {
      title: '{{t("Notification")}}',
      type: 'notification',
      group: 'extended',
      description: '{{t("Send notifications")}}',
      fieldset: {
        title: '{{t("Notification")}}',
        config: {
          type: 'void',
          'x-component': 'NotificationConfig',
        },
      },
      component: NotificationConfig,
    });

    // 注册自定义触发器
    workflowPlugin.registerTrigger('http', {
      title: '{{t("HTTP Request")}}',
      type: 'http',
      description: '{{t("Trigger workflow via HTTP request")}}',
      fieldset: {
        config: {
          type: 'void',
          'x-component': 'HttpTriggerConfig',
        },
      },
      component: HttpTriggerConfig,
      useVariables() {
        return [
          {
            name: 'request',
            title: '{{t("Request Data")}}',
            children: [
              { name: 'headers', title: '{{t("Headers")}}' },
              { name: 'query', title: '{{t("Query Parameters")}}' },
              { name: 'body', title: '{{t("Request Body")}}' },
            ],
          },
        ];
      },
    });

    // 注册组件
    this.app.addComponents({
      ApprovalConfig,
      DataProcessConfig,
      NotificationConfig,
      HttpTriggerConfig,
    });

    // 添加国际化
    this.app.i18n.addResources('zh-CN', 'workflow-extension', {
      'Approval': '审批',
      'Data Process': '数据处理',
      'Notification': '通知',
      'HTTP Request': 'HTTP请求',
      'Approval node for workflow': '工作流审批节点',
      'Process and transform data': '处理和转换数据',
      'Send notifications': '发送通知',
      'Trigger workflow via HTTP request': '通过HTTP请求触发工作流',
    });
  }
}

export default WorkflowExtensionPlugin;
```
```