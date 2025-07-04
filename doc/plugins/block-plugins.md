# 自定义区块插件开发

## 概述

区块插件用于扩展 NocoBase 的页面展示组件，提供各种数据展示、图表、交互组件等功能。本文档基于官方区块插件的实现模式，详细介绍如何开发自定义区块。

## 区块插件架构

### 核心组件

1. **区块组件 (Block Component)** - 区块的主要显示组件
2. **区块初始化器 (Block Initializer)** - 用于添加区块的配置界面
3. **区块设置 (Block Settings)** - 区块的配置和设置界面
4. **区块模板 (Block Template)** - 预定义的区块配置模板

### 官方区块插件分析

#### 表格区块插件

```typescript
// packages/plugins/@nocobase/plugin-data-visualization/src/client/block/ChartBlock.tsx
import { SchemaInitializer } from '@nocobase/client';

export const ChartBlockInitializer = new SchemaInitializer({
  name: 'ChartBlockInitializer',
  title: '{{t("Chart")}}',
  icon: 'BarChartOutlined',
  wrap: (schema) => {
    return {
      type: 'void',
      'x-component': 'CardItem',
      'x-designer': 'ChartBlockDesigner',
      properties: {
        chart: schema,
      },
    };
  },
  items: [
    {
      name: 'chart',
      type: 'item',
      title: '{{t("Chart")}}',
      component: 'ChartBlockInitializerItem',
    },
  ],
});
```

#### 看板区块插件

```typescript
// packages/plugins/@nocobase/plugin-kanban/src/client/KanbanBlockInitializer.tsx
export const KanbanBlockInitializer = {
  name: 'kanban',
  title: '{{t("Kanban")}}',
  icon: 'TableOutlined',
  component: 'KanbanBlockInitializerItem',
};
```

## 图表区块插件示例

### 图表区块组件

```typescript
// src/client/components/ChartBlock.tsx
import React, { useMemo } from 'react';
import { Card, Spin, Empty, Alert } from 'antd';
import { observer, useField, useFieldSchema } from '@formily/react';
import { useRequest, useAPIClient, useDesignable } from '@nocobase/client';
import { Chart } from '@antv/g2';

export const ChartBlock = observer(() => {
  const field = useField();
  const schema = useFieldSchema();
  const api = useAPIClient();
  const { designable } = useDesignable();
  
  const {
    collection,
    chartType = 'column',
    xField,
    yField,
    colorField,
    title,
    height = 400,
    config = {},
  } = schema['x-component-props'] || {};

  // 获取数据
  const { data, loading, error } = useRequest(
    {
      resource: collection,
      action: 'list',
      params: {
        pageSize: 1000, // 图表通常需要更多数据
        appends: [colorField].filter(Boolean),
      },
    },
    {
      ready: !!collection && !!xField && !!yField,
      refreshDeps: [collection, xField, yField, colorField],
    }
  );

  // 处理图表数据
  const chartData = useMemo(() => {
    if (!data?.data) return [];
    
    return data.data.map(item => ({
      [xField]: item[xField],
      [yField]: Number(item[yField]) || 0,
      ...(colorField && { [colorField]: item[colorField] }),
    }));
  }, [data, xField, yField, colorField]);

  // 渲染图表
  const renderChart = () => {
    if (loading) {
      return (
        <div style={{ height, display: 'flex', alignItems: 'center', justifyContent: 'center' }}>
          <Spin size="large" />
        </div>
      );
    }

    if (error) {
      return (
        <Alert
          message="加载失败"
          description={error.message}
          type="error"
          showIcon
        />
      );
    }

    if (!chartData.length) {
      return (
        <Empty
          description="暂无数据"
          style={{ height, display: 'flex', flexDirection: 'column', justifyContent: 'center' }}
        />
      );
    }

    return (
      <ChartRenderer
        type={chartType}
        data={chartData}
        xField={xField}
        yField={yField}
        colorField={colorField}
        height={height}
        config={config}
      />
    );
  };

  return (
    <Card
      title={title}
      bordered={false}
      style={{ height: 'auto' }}
      bodyStyle={{ padding: designable ? 16 : 0 }}
    >
      {renderChart()}
    </Card>
  );
});

// 图表渲染器
const ChartRenderer = ({ type, data, xField, yField, colorField, height, config }) => {
  const containerRef = React.useRef<HTMLDivElement>(null);
  const chartRef = React.useRef<Chart>(null);

  React.useEffect(() => {
    if (!containerRef.current || !data.length) return;

    // 清理之前的图表
    if (chartRef.current) {
      chartRef.current.destroy();
    }

    // 创建新图表
    const chart = new Chart({
      container: containerRef.current,
      autoFit: true,
      height,
      padding: [20, 20, 50, 50],
    });

    chart.data(data);

    // 根据图表类型配置
    switch (type) {
      case 'column':
        chart
          .interval()
          .position(`${xField}*${yField}`)
          .color(colorField || undefined);
        break;
      
      case 'line':
        chart
          .line()
          .position(`${xField}*${yField}`)
          .color(colorField || undefined);
        chart
          .point()
          .position(`${xField}*${yField}`)
          .color(colorField || undefined);
        break;
      
      case 'pie':
        chart
          .coordinate('theta', {
            radius: 0.75,
          });
        chart
          .interval()
          .position(yField)
          .color(xField)
          .label(yField, {
            content: (data) => `${data[xField]}: ${data[yField]}`,
          })
          .adjust('stack');
        break;
      
      case 'area':
        chart
          .area()
          .position(`${xField}*${yField}`)
          .color(colorField || undefined);
        chart
          .line()
          .position(`${xField}*${yField}`)
          .color(colorField || undefined);
        break;
    }

    // 应用自定义配置
    if (config.legend !== undefined) {
      chart.legend(config.legend);
    }
    
    if (config.tooltip !== undefined) {
      chart.tooltip(config.tooltip);
    }

    chart.render();
    chartRef.current = chart;

    return () => {
      if (chartRef.current) {
        chartRef.current.destroy();
      }
    };
  }, [type, data, xField, yField, colorField, height, config]);

  return <div ref={containerRef} style={{ width: '100%', height }} />;
};
```

### 图表区块初始化器

```typescript
// src/client/initializers/ChartBlockInitializer.tsx
import React from 'react';
import { FormOutlined } from '@ant-design/icons';
import { SchemaInitializerItem, useSchemaInitializer } from '@nocobase/client';

export const ChartBlockInitializerItem = () => {
  const { insert } = useSchemaInitializer();

  const handleClick = () => {
    insert({
      type: 'void',
      'x-component': 'CardItem',
      'x-designer': 'ChartBlockDesigner',
      'x-component-props': {
        title: '图表',
      },
      properties: {
        chart: {
          type: 'void',
          'x-component': 'ChartBlock',
          'x-component-props': {
            chartType: 'column',
            height: 400,
          },
        },
      },
    });
  };

  return (
    <SchemaInitializerItem
      icon={<FormOutlined />}
      title="图表"
      onClick={handleClick}
    />
  );
};
```

### 图表区块设计器

```typescript
// src/client/designers/ChartBlockDesigner.tsx
import React from 'react';
import { GeneralSchemaDesigner, useCollection, useCollectionManager } from '@nocobase/client';
import { ISchema } from '@formily/react';

export const ChartBlockDesigner = () => {
  const { name, title } = useCollection();
  const { getCollectionFields } = useCollectionManager();
  
  const fields = getCollectionFields(name);
  const numberFields = fields.filter(field => 
    ['integer', 'float', 'decimal'].includes(field.type)
  );
  const categoryFields = fields.filter(field => 
    ['string', 'select', 'radioGroup'].includes(field.interface)
  );

  return (
    <GeneralSchemaDesigner
      schemaSettings="chartBlockSettings"
      template={{
        type: 'void',
        'x-component': 'CardItem',
        'x-designer': 'ChartBlockDesigner',
        properties: {
          chart: {
            type: 'void',
            'x-component': 'ChartBlock',
            'x-component-props': {
              collection: name,
              chartType: 'column',
              height: 400,
            },
          },
        },
      }}
    />
  );
};

// 图表区块设置
export const chartBlockSettings = new SchemaSettings({
  name: 'chartBlockSettings',
  items: [
    {
      name: 'collection',
      type: 'select',
      title: '数据表',
      componentProps: {
        title: '选择数据表',
      },
      useComponentProps() {
        const { collections } = useCollectionManager();
        return {
          options: collections.map(collection => ({
            label: collection.title || collection.name,
            value: collection.name,
          })),
        };
      },
    },
    {
      name: 'chartType',
      type: 'select',
      title: '图表类型',
      componentProps: {
        options: [
          { label: '柱状图', value: 'column' },
          { label: '折线图', value: 'line' },
          { label: '饼图', value: 'pie' },
          { label: '面积图', value: 'area' },
        ],
      },
    },
    {
      name: 'xField',
      type: 'select',
      title: 'X轴字段',
      useComponentProps() {
        const { name } = useCollection();
        const { getCollectionFields } = useCollectionManager();
        const fields = getCollectionFields(name);
        
        return {
          options: fields.map(field => ({
            label: field.uiSchema?.title || field.name,
            value: field.name,
          })),
        };
      },
    },
    {
      name: 'yField',
      type: 'select',
      title: 'Y轴字段',
      useComponentProps() {
        const { name } = useCollection();
        const { getCollectionFields } = useCollectionManager();
        const fields = getCollectionFields(name).filter(field => 
          ['integer', 'float', 'decimal'].includes(field.type)
        );
        
        return {
          options: fields.map(field => ({
            label: field.uiSchema?.title || field.name,
            value: field.name,
          })),
        };
      },
    },
    {
      name: 'colorField',
      type: 'select',
      title: '分组字段',
      componentProps: {
        allowClear: true,
      },
      useComponentProps() {
        const { name } = useCollection();
        const { getCollectionFields } = useCollectionManager();
        const fields = getCollectionFields(name);
        
        return {
          options: fields.map(field => ({
            label: field.uiSchema?.title || field.name,
            value: field.name,
          })),
        };
      },
    },
    {
      name: 'height',
      type: 'number',
      title: '图表高度',
      componentProps: {
        min: 200,
        max: 800,
        step: 50,
      },
    },
    {
      name: 'title',
      type: 'input',
      title: '图表标题',
    },
    {
      name: 'divider',
      type: 'divider',
    },
    {
      name: 'remove',
      type: 'remove',
      componentProps: {
        removeParentsIfNoChildren: true,
        breakRemoveOn: {
          'x-component': 'Grid',
        },
      },
    },
  ],
});
```

## 看板区块插件示例

### 看板区块组件

```typescript
// src/client/components/KanbanBlock.tsx
import React, { useMemo } from 'react';
import { Card, Avatar, Tag, Empty, Spin } from 'antd';
import { DragDropContext, Droppable, Draggable } from 'react-beautiful-dnd';
import { observer, useField, useFieldSchema } from '@formily/react';
import { useRequest, useAPIClient } from '@nocobase/client';

export const KanbanBlock = observer(() => {
  const field = useField();
  const schema = useFieldSchema();
  const api = useAPIClient();

  const {
    collection,
    groupField,
    titleField,
    descriptionField,
    avatarField,
    tagFields = [],
    height = 600,
  } = schema['x-component-props'] || {};

  // 获取数据
  const { data, loading, refresh } = useRequest(
    {
      resource: collection,
      action: 'list',
      params: {
        pageSize: 1000,
        appends: [avatarField, ...tagFields].filter(Boolean),
        sort: ['createdAt'],
      },
    },
    {
      ready: !!collection && !!groupField && !!titleField,
      refreshDeps: [collection, groupField, titleField],
    }
  );

  // 处理看板数据
  const kanbanData = useMemo(() => {
    if (!data?.data) return {};

    const groups = {};

    // 初始化分组
    const groupValues = ['待处理', '进行中', '已完成'];
    groupValues.forEach(value => {
      groups[value] = {
        id: value,
        title: value,
        items: [],
      };
    });

    // 分组数据
    data.data.forEach(item => {
      const groupValue = item[groupField];
      if (groups[groupValue]) {
        groups[groupValue].items.push(item);
      }
    });

    return groups;
  }, [data, groupField]);

  // 拖拽处理
  const handleDragEnd = async (result) => {
    const { destination, source, draggableId } = result;

    if (!destination) return;

    if (
      destination.droppableId === source.droppableId &&
      destination.index === source.index
    ) {
      return;
    }

    try {
      // 更新数据库中的分组字段
      await api.resource(collection).update({
        filterByTk: draggableId,
        values: {
          [groupField]: destination.droppableId,
        },
      });

      // 刷新数据
      refresh();
    } catch (error) {
      console.error('Failed to update item:', error);
    }
  };

  if (loading) {
    return (
      <div style={{ height, display: 'flex', alignItems: 'center', justifyContent: 'center' }}>
        <Spin size="large" />
      </div>
    );
  }

  if (!Object.keys(kanbanData).length) {
    return <Empty description="暂无数据" />;
  }

  return (
    <div style={{ height, overflow: 'auto' }}>
      <DragDropContext onDragEnd={handleDragEnd}>
        <div style={{ display: 'flex', gap: 16, padding: 16, minWidth: 'max-content' }}>
          {Object.values(kanbanData).map((group: any) => (
            <KanbanColumn
              key={group.id}
              group={group}
              titleField={titleField}
              descriptionField={descriptionField}
              avatarField={avatarField}
              tagFields={tagFields}
            />
          ))}
        </div>
      </DragDropContext>
    </div>
  );
});

// 看板列组件
const KanbanColumn = ({ group, titleField, descriptionField, avatarField, tagFields }) => {
  return (
    <div style={{ minWidth: 280, maxWidth: 320 }}>
      <Card
        title={
          <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center' }}>
            <span>{group.title}</span>
            <Tag>{group.items.length}</Tag>
          </div>
        }
        size="small"
        style={{ height: '100%' }}
        bodyStyle={{ padding: 8, maxHeight: 500, overflow: 'auto' }}
      >
        <Droppable droppableId={group.id}>
          {(provided, snapshot) => (
            <div
              ref={provided.innerRef}
              {...provided.droppableProps}
              style={{
                backgroundColor: snapshot.isDraggingOver ? '#f0f0f0' : 'transparent',
                minHeight: 100,
                borderRadius: 4,
                padding: 4,
              }}
            >
              {group.items.map((item, index) => (
                <KanbanCard
                  key={item.id}
                  item={item}
                  index={index}
                  titleField={titleField}
                  descriptionField={descriptionField}
                  avatarField={avatarField}
                  tagFields={tagFields}
                />
              ))}
              {provided.placeholder}
            </div>
          )}
        </Droppable>
      </Card>
    </div>
  );
};

// 看板卡片组件
const KanbanCard = ({ item, index, titleField, descriptionField, avatarField, tagFields }) => {
  return (
    <Draggable draggableId={String(item.id)} index={index}>
      {(provided, snapshot) => (
        <div
          ref={provided.innerRef}
          {...provided.draggableProps}
          {...provided.dragHandleProps}
          style={{
            ...provided.draggableProps.style,
            marginBottom: 8,
          }}
        >
          <Card
            size="small"
            style={{
              cursor: 'grab',
              backgroundColor: snapshot.isDragging ? '#e6f7ff' : 'white',
              boxShadow: snapshot.isDragging ? '0 4px 8px rgba(0,0,0,0.1)' : undefined,
            }}
            bodyStyle={{ padding: 12 }}
          >
            <div style={{ marginBottom: 8 }}>
              <div style={{ fontWeight: 500, marginBottom: 4 }}>
                {item[titleField]}
              </div>
              {descriptionField && item[descriptionField] && (
                <div style={{ fontSize: 12, color: '#666', lineHeight: 1.4 }}>
                  {item[descriptionField]}
                </div>
              )}
            </div>

            {tagFields.length > 0 && (
              <div style={{ marginBottom: 8 }}>
                {tagFields.map(tagField => {
                  const tagValue = item[tagField];
                  if (!tagValue) return null;

                  return (
                    <Tag key={tagField} size="small" style={{ marginRight: 4, marginBottom: 4 }}>
                      {tagValue}
                    </Tag>
                  );
                })}
              </div>
            )}

            {avatarField && item[avatarField] && (
              <div style={{ display: 'flex', alignItems: 'center', justifyContent: 'space-between' }}>
                <Avatar size="small" src={item[avatarField]} />
                <span style={{ fontSize: 12, color: '#999' }}>
                  {new Date(item.createdAt).toLocaleDateString()}
                </span>
              </div>
            )}
          </Card>
        </div>
      )}
    </Draggable>
  );
};
```
