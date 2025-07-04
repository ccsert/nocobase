# 系统级复杂插件实现示例

## 概述

本文档提供系统级复杂插件的完整实现示例，包括部门管理、角色权限、工作流引擎、文件管理和通知系统等核心业务插件。这些示例展示了如何构建企业级的复杂功能模块。

## 1. 部门管理插件

### 数据模型设计

```typescript
// src/server/collections/departments.ts
export default {
  name: 'departments',
  title: '部门',
  tree: 'adjacency-list', // 邻接表树结构
  fields: [
    {
      name: 'id',
      type: 'bigInt',
      autoIncrement: true,
      primaryKey: true,
    },
    {
      name: 'name',
      type: 'string',
      allowNull: false,
      unique: true,
      validate: {
        len: [1, 100],
      },
    },
    {
      name: 'code',
      type: 'string',
      allowNull: false,
      unique: true,
      validate: {
        len: [1, 50],
      },
    },
    {
      name: 'description',
      type: 'text',
    },
    {
      name: 'parentId',
      type: 'bigInt',
      references: {
        model: 'departments',
        key: 'id',
      },
    },
    {
      name: 'parent',
      type: 'belongsTo',
      target: 'departments',
      foreignKey: 'parentId',
    },
    {
      name: 'children',
      type: 'hasMany',
      target: 'departments',
      foreignKey: 'parentId',
    },
    {
      name: 'manager',
      type: 'belongsTo',
      target: 'users',
      foreignKey: 'managerId',
    },
    {
      name: 'members',
      type: 'hasMany',
      target: 'users',
      foreignKey: 'departmentId',
    },
    {
      name: 'level',
      type: 'integer',
      defaultValue: 1,
    },
    {
      name: 'path',
      type: 'string', // 存储路径，如 "/1/2/3"
    },
    {
      name: 'sort',
      type: 'integer',
      defaultValue: 0,
    },
    {
      name: 'status',
      type: 'string',
      defaultValue: 'active',
      validate: {
        isIn: [['active', 'inactive']],
      },
    },
  ],
  indexes: [
    {
      fields: ['parentId'],
    },
    {
      fields: ['path'],
    },
    {
      fields: ['level'],
    },
  ],
};
```

### 后端服务实现

```typescript
// src/server/services/DepartmentService.ts
import { Service } from '@nocobase/server';

export class DepartmentService extends Service {
  async createDepartment(data: any) {
    const { parentId, ...departmentData } = data;
    
    // 计算层级和路径
    let level = 1;
    let path = '';
    
    if (parentId) {
      const parent = await this.db.getRepository('departments').findByPk(parentId);
      if (!parent) {
        throw new Error('Parent department not found');
      }
      
      level = parent.level + 1;
      path = `${parent.path}/${parentId}`;
    } else {
      path = '';
    }

    const department = await this.db.getRepository('departments').create({
      values: {
        ...departmentData,
        parentId,
        level,
        path,
      },
    });

    // 更新路径
    if (!parentId) {
      await department.update({ path: `/${department.id}` });
    }

    return department;
  }

  async moveDepartment(departmentId: number, newParentId?: number) {
    const department = await this.db.getRepository('departments').findByPk(departmentId, {
      appends: ['children'],
    });

    if (!department) {
      throw new Error('Department not found');
    }

    // 检查是否移动到自己的子部门
    if (newParentId) {
      const newParent = await this.db.getRepository('departments').findByPk(newParentId);
      if (!newParent) {
        throw new Error('New parent department not found');
      }

      if (newParent.path.includes(`/${departmentId}/`)) {
        throw new Error('Cannot move department to its own child');
      }
    }

    // 计算新的层级和路径
    let newLevel = 1;
    let newPath = '';

    if (newParentId) {
      const newParent = await this.db.getRepository('departments').findByPk(newParentId);
      newLevel = newParent.level + 1;
      newPath = `${newParent.path}/${newParentId}`;
    } else {
      newPath = `/${departmentId}`;
    }

    // 更新当前部门
    await department.update({
      parentId: newParentId,
      level: newLevel,
      path: newPath,
    });

    // 递归更新所有子部门
    await this.updateChildrenPaths(department);

    return department;
  }

  private async updateChildrenPaths(department: any) {
    const children = await this.db.getRepository('departments').find({
      filter: { parentId: department.id },
    });

    for (const child of children) {
      const newPath = `${department.path}/${child.id}`;
      const newLevel = department.level + 1;

      await child.update({
        path: newPath,
        level: newLevel,
      });

      // 递归更新子部门
      await this.updateChildrenPaths(child);
    }
  }

  async getDepartmentTree(rootId?: number) {
    const filter = rootId ? { parentId: rootId } : { parentId: null };
    
    const departments = await this.db.getRepository('departments').find({
      filter,
      sort: ['sort', 'createdAt'],
      appends: ['manager', 'members'],
    });

    // 递归构建树结构
    for (const dept of departments) {
      dept.children = await this.getDepartmentTree(dept.id);
    }

    return departments;
  }

  async assignManager(departmentId: number, managerId: number) {
    const department = await this.db.getRepository('departments').findByPk(departmentId);
    if (!department) {
      throw new Error('Department not found');
    }

    const user = await this.db.getRepository('users').findByPk(managerId);
    if (!user) {
      throw new Error('User not found');
    }

    await department.update({ managerId });

    // 添加管理员角色
    await this.addUserRole(managerId, 'department_manager', departmentId);

    return department;
  }

  async addMember(departmentId: number, userId: number) {
    const department = await this.db.getRepository('departments').findByPk(departmentId);
    if (!department) {
      throw new Error('Department not found');
    }

    const user = await this.db.getRepository('users').findByPk(userId);
    if (!user) {
      throw new Error('User not found');
    }

    await user.update({ departmentId });

    return user;
  }

  async removeMember(userId: number) {
    const user = await this.db.getRepository('users').findByPk(userId);
    if (!user) {
      throw new Error('User not found');
    }

    await user.update({ departmentId: null });

    return user;
  }

  private async addUserRole(userId: number, roleName: string, departmentId?: number) {
    // 实现用户角色分配逻辑
    const roleRepository = this.db.getRepository('roles');
    const userRoleRepository = this.db.getRepository('userRoles');

    const role = await roleRepository.findOne({
      filter: { name: roleName },
    });

    if (role) {
      await userRoleRepository.create({
        values: {
          userId,
          roleId: role.id,
          scope: departmentId ? { departmentId } : null,
        },
      });
    }
  }
}
```

### 前端组件实现

```typescript
// src/client/components/DepartmentTree.tsx
import React, { useState } from 'react';
import { Tree, Card, Button, Modal, Form, Input, Select, Space, Popconfirm } from 'antd';
import { PlusOutlined, EditOutlined, DeleteOutlined, UserOutlined } from '@ant-design/icons';
import { useRequest, useAPIClient } from '@nocobase/client';

const { TreeNode } = Tree;
const { Option } = Select;

export const DepartmentTree = () => {
  const api = useAPIClient();
  const [selectedDept, setSelectedDept] = useState(null);
  const [modalVisible, setModalVisible] = useState(false);
  const [modalType, setModalType] = useState('create'); // create, edit, assign
  const [form] = Form.useForm();

  // 获取部门树数据
  const { data: departments, loading, refresh } = useRequest({
    resource: 'departments',
    action: 'tree',
  });

  // 获取用户列表
  const { data: users } = useRequest({
    resource: 'users',
    action: 'list',
    params: { pageSize: 1000 },
  });

  // 创建部门
  const handleCreate = (parentId?: number) => {
    setModalType('create');
    setSelectedDept({ parentId });
    form.resetFields();
    setModalVisible(true);
  };

  // 编辑部门
  const handleEdit = (dept: any) => {
    setModalType('edit');
    setSelectedDept(dept);
    form.setFieldsValue(dept);
    setModalVisible(true);
  };

  // 分配管理员
  const handleAssignManager = (dept: any) => {
    setModalType('assign');
    setSelectedDept(dept);
    form.setFieldsValue({ managerId: dept.managerId });
    setModalVisible(true);
  };

  // 删除部门
  const handleDelete = async (deptId: number) => {
    try {
      await api.resource('departments').destroy({ filterByTk: deptId });
      refresh();
    } catch (error) {
      console.error('Failed to delete department:', error);
    }
  };

  // 提交表单
  const handleSubmit = async (values: any) => {
    try {
      if (modalType === 'create') {
        await api.resource('departments').create({
          values: { ...values, parentId: selectedDept?.parentId },
        });
      } else if (modalType === 'edit') {
        await api.resource('departments').update({
          filterByTk: selectedDept.id,
          values,
        });
      } else if (modalType === 'assign') {
        await api.resource('departments').assignManager({
          filterByTk: selectedDept.id,
          managerId: values.managerId,
        });
      }

      setModalVisible(false);
      refresh();
    } catch (error) {
      console.error('Failed to submit:', error);
    }
  };

  // 渲染树节点
  const renderTreeNodes = (data: any[]) => {
    return data?.map(dept => (
      <TreeNode
        key={dept.id}
        title={
          <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center' }}>
            <span>
              {dept.name}
              {dept.manager && (
                <span style={{ marginLeft: 8, color: '#666', fontSize: 12 }}>
                  (管理员: {dept.manager.name})
                </span>
              )}
            </span>
            <Space size="small">
              <Button
                type="text"
                size="small"
                icon={<PlusOutlined />}
                onClick={(e) => {
                  e.stopPropagation();
                  handleCreate(dept.id);
                }}
              />
              <Button
                type="text"
                size="small"
                icon={<EditOutlined />}
                onClick={(e) => {
                  e.stopPropagation();
                  handleEdit(dept);
                }}
              />
              <Button
                type="text"
                size="small"
                icon={<UserOutlined />}
                onClick={(e) => {
                  e.stopPropagation();
                  handleAssignManager(dept);
                }}
              />
              <Popconfirm
                title="确定删除这个部门吗？"
                onConfirm={(e) => {
                  e?.stopPropagation();
                  handleDelete(dept.id);
                }}
                onCancel={(e) => e?.stopPropagation()}
              >
                <Button
                  type="text"
                  size="small"
                  danger
                  icon={<DeleteOutlined />}
                  onClick={(e) => e.stopPropagation()}
                />
              </Popconfirm>
            </Space>
          </div>
        }
      >
        {dept.children && renderTreeNodes(dept.children)}
      </TreeNode>
    ));
  };

  return (
    <Card
      title="部门管理"
      extra={
        <Button type="primary" icon={<PlusOutlined />} onClick={() => handleCreate()}>
          添加根部门
        </Button>
      }
    >
      <Tree
        showLine
        defaultExpandAll
        loading={loading}
      >
        {renderTreeNodes(departments?.data || [])}
      </Tree>

      <Modal
        title={
          modalType === 'create' ? '创建部门' :
          modalType === 'edit' ? '编辑部门' : '分配管理员'
        }
        open={modalVisible}
        onCancel={() => setModalVisible(false)}
        onOk={() => form.submit()}
      >
        <Form form={form} layout="vertical" onFinish={handleSubmit}>
          {modalType !== 'assign' && (
            <>
              <Form.Item
                name="name"
                label="部门名称"
                rules={[{ required: true, message: '请输入部门名称' }]}
              >
                <Input />
              </Form.Item>
              
              <Form.Item
                name="code"
                label="部门编码"
                rules={[{ required: true, message: '请输入部门编码' }]}
              >
                <Input />
              </Form.Item>
              
              <Form.Item name="description" label="部门描述">
                <Input.TextArea rows={3} />
              </Form.Item>
            </>
          )}

          {modalType === 'assign' && (
            <Form.Item
              name="managerId"
              label="管理员"
              rules={[{ required: true, message: '请选择管理员' }]}
            >
              <Select placeholder="选择管理员" showSearch>
                {users?.data?.map(user => (
                  <Option key={user.id} value={user.id}>
                    {user.name} ({user.email})
                  </Option>
                ))}
              </Select>
            </Form.Item>
          )}
        </Form>
      </Modal>
    </Card>
  );
};
```

## 2. 角色权限插件 (RBAC)

### 数据模型设计

```typescript
// src/server/collections/roles.ts
export default {
  name: 'roles',
  title: '角色',
  fields: [
    {
      name: 'id',
      type: 'bigInt',
      autoIncrement: true,
      primaryKey: true,
    },
    {
      name: 'name',
      type: 'string',
      allowNull: false,
      unique: true,
    },
    {
      name: 'title',
      type: 'string',
      allowNull: false,
    },
    {
      name: 'description',
      type: 'text',
    },
    {
      name: 'type',
      type: 'string',
      defaultValue: 'custom',
      validate: {
        isIn: [['system', 'custom', 'department']],
      },
    },
    {
      name: 'permissions',
      type: 'hasMany',
      target: 'rolePermissions',
      foreignKey: 'roleId',
    },
    {
      name: 'users',
      type: 'belongsToMany',
      target: 'users',
      through: 'userRoles',
    },
    {
      name: 'status',
      type: 'string',
      defaultValue: 'active',
      validate: {
        isIn: [['active', 'inactive']],
      },
    },
  ],
};

// src/server/collections/permissions.ts
export default {
  name: 'permissions',
  title: '权限',
  fields: [
    {
      name: 'id',
      type: 'bigInt',
      autoIncrement: true,
      primaryKey: true,
    },
    {
      name: 'name',
      type: 'string',
      allowNull: false,
      unique: true,
    },
    {
      name: 'title',
      type: 'string',
      allowNull: false,
    },
    {
      name: 'resource',
      type: 'string',
      allowNull: false,
    },
    {
      name: 'action',
      type: 'string',
      allowNull: false,
    },
    {
      name: 'scope',
      type: 'json', // 权限范围配置
    },
    {
      name: 'category',
      type: 'string',
      defaultValue: 'general',
    },
  ],
  indexes: [
    {
      fields: ['resource', 'action'],
      unique: true,
    },
  ],
};
```
