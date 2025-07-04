# 自定义字段插件开发

## 概述

字段插件是 NocoBase 中最常见的插件类型，用于扩展数据字段类型和相应的前端组件。本文档基于官方字段插件的实现模式，提供完整的开发指南。

## 字段插件架构

### 核心组件

1. **字段接口 (Field Interface)** - 定义字段的配置和行为
2. **字段类型 (Field Type)** - 后端数据库字段类型
3. **前端组件** - 字段的显示和编辑组件
4. **配置组件** - 字段配置界面

### 官方字段插件分析

#### 序列号字段插件

```typescript
// packages/plugins/@nocobase/plugin-field-sequence/src/client/index.tsx
import { Plugin } from '@nocobase/client';
import { SequenceFieldProvider } from './SequenceFieldProvider';
import { SequenceFieldInterface } from './sequence';

export class PluginFieldSequenceClient extends Plugin {
  async load() {
    // 1. 添加字段提供者
    this.app.use(SequenceFieldProvider);
    
    // 2. 注册字段接口
    this.app.dataSourceManager.addFieldInterfaces([SequenceFieldInterface]);
    
    // 3. 与其他插件集成
    const calendarPlugin: any = this.app.pm.get('calendar');
    calendarPlugin?.registerTitleFieldInterface('sequence');
  }
}
```

#### 排序字段插件

```typescript
// packages/plugins/@nocobase/plugin-field-sort/src/client/index.tsx
import { Plugin } from '@nocobase/client';
import { SortFieldInterface } from './sort-interface';

export class PluginFieldSortClient extends Plugin {
  async load() {
    // 直接注册字段接口
    this.app.addFieldInterfaces([SortFieldInterface]);
  }
}
```

## 字段接口开发

### 基础字段接口

```typescript
// src/client/interfaces/MyFieldInterface.ts
import { IFieldInterface } from '@nocobase/client';

export const MyFieldInterface: IFieldInterface = {
  name: 'myField',
  type: 'object',
  group: 'basic',
  title: '{{t("My Field")}}',
  description: '{{t("My custom field description")}}',
  sortable: true,
  filterable: true,
  
  // 默认UI Schema
  default: {
    type: 'string',
    'x-component': 'MyFieldComponent',
    'x-component-props': {},
  },
  
  // 可用的组件
  availableTypes: ['MyFieldComponent', 'MyFieldReadPretty'],
  
  // 字段配置Schema
  schemaInitialize(schema: ISchema, { readPretty }) {
    if (readPretty) {
      schema['x-component'] = 'MyFieldReadPretty';
    } else {
      schema['x-component'] = 'MyFieldComponent';
    }
  },
  
  // 字段属性配置
  properties: {
    'uiSchema.x-component-props.placeholder': {
      type: 'string',
      title: '{{t("Placeholder")}}',
      'x-component': 'Input',
      'x-decorator': 'FormItem',
    },
    'uiSchema.x-component-props.maxLength': {
      type: 'number',
      title: '{{t("Max length")}}',
      'x-component': 'InputNumber',
      'x-decorator': 'FormItem',
    },
  },
  
  // 过滤操作符
  filterable: {
    operators: [
      { label: '{{t("contains")}}', value: '$includes', selected: true },
      { label: '{{t("does not contain")}}', value: '$notIncludes' },
      { label: '{{t("is")}}', value: '$eq' },
      { label: '{{t("is not")}}', value: '$ne' },
    ],
  },
};
```

### 复杂字段接口示例

```typescript
// src/client/interfaces/RichTextInterface.ts
import { IFieldInterface } from '@nocobase/client';

export const RichTextInterface: IFieldInterface = {
  name: 'richText',
  type: 'object',
  group: 'media',
  title: '{{t("Rich Text")}}',
  description: '{{t("Rich text editor with formatting support")}}',
  sortable: false,
  filterable: {
    operators: [
      { label: '{{t("contains")}}', value: '$includes', selected: true },
      { label: '{{t("does not contain")}}', value: '$notIncludes' },
    ],
  },
  
  default: {
    type: 'string',
    'x-component': 'RichTextEditor',
    'x-component-props': {
      toolbar: 'full',
      height: 300,
    },
  },
  
  availableTypes: ['RichTextEditor', 'RichTextReadPretty'],
  
  schemaInitialize(schema: ISchema, { readPretty }) {
    if (readPretty) {
      schema['x-component'] = 'RichTextReadPretty';
    } else {
      schema['x-component'] = 'RichTextEditor';
    }
  },
  
  properties: {
    'uiSchema.x-component-props.toolbar': {
      type: 'string',
      title: '{{t("Toolbar")}}',
      'x-component': 'Select',
      'x-decorator': 'FormItem',
      enum: [
        { label: '{{t("Basic")}}', value: 'basic' },
        { label: '{{t("Standard")}}', value: 'standard' },
        { label: '{{t("Full")}}', value: 'full' },
      ],
      default: 'standard',
    },
    'uiSchema.x-component-props.height': {
      type: 'number',
      title: '{{t("Height")}}',
      'x-component': 'InputNumber',
      'x-decorator': 'FormItem',
      'x-component-props': {
        min: 100,
        max: 800,
        step: 50,
      },
      default: 300,
    },
    'uiSchema.x-component-props.placeholder': {
      type: 'string',
      title: '{{t("Placeholder")}}',
      'x-component': 'Input.TextArea',
      'x-decorator': 'FormItem',
    },
  },
};
```

## 前端组件开发

### 编辑组件

```typescript
// src/client/components/RichTextEditor.tsx
import React, { useRef, useEffect } from 'react';
import { observer, useField } from '@formily/react';
import { Input } from 'antd';
import { useTranslation } from 'react-i18next';

// 假设使用 TinyMCE 作为富文本编辑器
import { Editor } from '@tinymce/tinymce-react';

export const RichTextEditor = observer((props: any) => {
  const { 
    value, 
    onChange, 
    toolbar = 'standard',
    height = 300,
    placeholder,
    disabled,
    ...restProps 
  } = props;
  
  const field = useField();
  const { t } = useTranslation();
  const editorRef = useRef(null);

  // 工具栏配置
  const toolbarConfigs = {
    basic: 'bold italic underline | bullist numlist | link',
    standard: 'bold italic underline strikethrough | formatselect | bullist numlist | outdent indent | link image | undo redo',
    full: 'bold italic underline strikethrough | formatselect fontselect fontsizeselect | forecolor backcolor | bullist numlist | outdent indent | alignleft aligncenter alignright alignjustify | link image media table | code | undo redo | fullscreen',
  };

  const handleEditorChange = (content: string) => {
    onChange?.(content);
  };

  // 处理图片上传
  const handleImageUpload = (blobInfo, progress) => {
    return new Promise((resolve, reject) => {
      const formData = new FormData();
      formData.append('file', blobInfo.blob(), blobInfo.filename());

      // 使用 NocoBase 的文件上传 API
      fetch('/api/attachments:upload', {
        method: 'POST',
        body: formData,
        headers: {
          'Authorization': `Bearer ${localStorage.getItem('NOCOBASE_TOKEN')}`,
        },
      })
      .then(response => response.json())
      .then(result => {
        if (result.data) {
          resolve(result.data.url);
        } else {
          reject('Upload failed');
        }
      })
      .catch(error => {
        reject(error);
      });
    });
  };

  return (
    <div className="rich-text-editor">
      <Editor
        ref={editorRef}
        value={value || ''}
        onEditorChange={handleEditorChange}
        disabled={disabled}
        init={{
          height,
          menubar: false,
          toolbar: toolbarConfigs[toolbar] || toolbarConfigs.standard,
          placeholder,
          plugins: [
            'advlist', 'autolink', 'lists', 'link', 'image', 'charmap',
            'anchor', 'searchreplace', 'visualblocks', 'code', 'fullscreen',
            'insertdatetime', 'media', 'table', 'help', 'wordcount'
          ],
          images_upload_handler: handleImageUpload,
          content_style: `
            body { 
              font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; 
              font-size: 14px;
              line-height: 1.6;
            }
          `,
          branding: false,
          resize: false,
          statusbar: false,
        }}
        {...restProps}
      />
      
      {field?.errors?.length > 0 && (
        <div className="ant-form-item-explain ant-form-item-explain-error">
          {field.errors.map((error, index) => (
            <div key={index}>{error.message}</div>
          ))}
        </div>
      )}
    </div>
  );
});

RichTextEditor.displayName = 'RichTextEditor';
```

### 只读组件

```typescript
// src/client/components/RichTextReadPretty.tsx
import React from 'react';
import { observer } from '@formily/react';
import DOMPurify from 'dompurify';

export const RichTextReadPretty = observer((props: any) => {
  const { value, className, style } = props;

  if (!value) {
    return <div className="text-gray-400">-</div>;
  }

  // 清理HTML内容，防止XSS攻击
  const sanitizedHTML = DOMPurify.sanitize(value, {
    ALLOWED_TAGS: [
      'p', 'br', 'strong', 'em', 'u', 's', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
      'ul', 'ol', 'li', 'a', 'img', 'table', 'thead', 'tbody', 'tr', 'th', 'td',
      'blockquote', 'code', 'pre', 'span', 'div'
    ],
    ALLOWED_ATTR: [
      'href', 'src', 'alt', 'title', 'width', 'height', 'style', 'class',
      'target', 'rel', 'colspan', 'rowspan'
    ],
    ALLOWED_URI_REGEXP: /^(?:(?:(?:f|ht)tps?|mailto|tel|callto|cid|xmpp|data):|[^a-z]|[a-z+.\-]+(?:[^a-z+.\-:]|$))/i,
  });

  return (
    <div 
      className={`rich-text-content ${className || ''}`}
      style={style}
      dangerouslySetInnerHTML={{ __html: sanitizedHTML }}
    />
  );
});

RichTextReadPretty.displayName = 'RichTextReadPretty';
```

## 后端字段类型开发

### 字段类型定义

```typescript
// src/server/fields/RichTextField.ts
import { DataTypes } from 'sequelize';
import { BaseColumnFieldOptions, Field } from '@nocobase/database';

export interface RichTextFieldOptions extends BaseColumnFieldOptions {
  type: 'richText';
  maxLength?: number;
  allowedTags?: string[];
  allowedAttributes?: string[];
}

export class RichTextField extends Field {
  get dataType() {
    return DataTypes.TEXT;
  }

  init() {
    const { maxLength } = this.options;
    
    // 添加验证规则
    this.addValidation('richTextValidation', (value) => {
      if (!value) return true;
      
      if (typeof value !== 'string') {
        throw new Error('Rich text field must be a string');
      }
      
      if (maxLength && value.length > maxLength) {
        throw new Error(`Rich text content exceeds maximum length of ${maxLength} characters`);
      }
      
      return true;
    });
  }

  // 数据处理钩子
  bind() {
    super.bind();
    
    // 保存前处理
    this.on('beforeSave', (instance, options) => {
      const value = instance.get(this.name);
      if (value) {
        // 清理HTML内容
        const cleanValue = this.sanitizeHTML(value);
        instance.set(this.name, cleanValue);
      }
    });
  }

  private sanitizeHTML(html: string): string {
    // 这里可以使用服务端的HTML清理库
    // 例如：jsdom + DOMPurify 或其他清理库
    return html; // 简化示例，实际应该进行清理
  }
}
```

### 字段注册

```typescript
// src/server/index.ts
import { Plugin } from '@nocobase/server';
import { RichTextField } from './fields/RichTextField';

export class RichTextFieldPlugin extends Plugin {
  beforeLoad() {
    // 注册字段类型
    this.db.registerFieldTypes({
      richText: RichTextField,
    });
  }

  async load() {
    // 注册字段接口
    this.db.interfaceManager.registerInterfaceType('richText', {
      type: 'richText',
      title: 'Rich Text',
      group: 'media',
      description: 'Rich text editor with formatting support',
      sortable: false,
      filterable: {
        operators: ['$includes', '$notIncludes'],
      },
      availableTypes: ['richText'],
      properties: {
        maxLength: {
          type: 'number',
          title: 'Max Length',
          default: 10000,
        },
      },
    });
  }
}

export default RichTextFieldPlugin;
```

## 地理位置字段插件示例

### 地理位置字段接口

```typescript
// src/client/interfaces/LocationInterface.ts
import { IFieldInterface } from '@nocobase/client';

export const LocationInterface: IFieldInterface = {
  name: 'location',
  type: 'object',
  group: 'basic',
  title: '{{t("Location")}}',
  description: '{{t("Geographic location with latitude and longitude")}}',
  sortable: false,
  filterable: {
    operators: [
      { label: '{{t("within distance")}}', value: '$near' },
      { label: '{{t("within bounds")}}', value: '$within' },
    ],
  },
  
  default: {
    type: 'object',
    'x-component': 'LocationPicker',
    'x-component-props': {
      mapType: 'google',
      zoom: 15,
    },
  },
  
  availableTypes: ['LocationPicker', 'LocationDisplay'],
  
  schemaInitialize(schema: ISchema, { readPretty }) {
    if (readPretty) {
      schema['x-component'] = 'LocationDisplay';
    } else {
      schema['x-component'] = 'LocationPicker';
    }
  },
  
  properties: {
    'uiSchema.x-component-props.mapType': {
      type: 'string',
      title: '{{t("Map Type")}}',
      'x-component': 'Select',
      'x-decorator': 'FormItem',
      enum: [
        { label: 'Google Maps', value: 'google' },
        { label: 'OpenStreetMap', value: 'osm' },
        { label: 'Baidu Maps', value: 'baidu' },
      ],
      default: 'google',
    },
    'uiSchema.x-component-props.zoom': {
      type: 'number',
      title: '{{t("Default Zoom")}}',
      'x-component': 'InputNumber',
      'x-decorator': 'FormItem',
      'x-component-props': {
        min: 1,
        max: 20,
      },
      default: 15,
    },
  },
};
```

### 地理位置选择组件

```typescript
// src/client/components/LocationPicker.tsx
import React, { useState, useEffect } from 'react';
import { observer, useField } from '@formily/react';
import { Button, Input, Space, Modal } from 'antd';
import { EnvironmentOutlined } from '@ant-design/icons';

interface LocationValue {
  latitude: number;
  longitude: number;
  address?: string;
}

export const LocationPicker = observer((props: any) => {
  const { value, onChange, mapType = 'google', zoom = 15, disabled } = props;
  const field = useField();
  const [modalVisible, setModalVisible] = useState(false);
  const [currentLocation, setCurrentLocation] = useState<LocationValue | null>(value);

  const handleLocationSelect = (location: LocationValue) => {
    setCurrentLocation(location);
    onChange?.(location);
    setModalVisible(false);
  };

  const getCurrentPosition = () => {
    if (navigator.geolocation) {
      navigator.geolocation.getCurrentPosition(
        (position) => {
          const location = {
            latitude: position.coords.latitude,
            longitude: position.coords.longitude,
          };
          
          // 反向地理编码获取地址
          reverseGeocode(location).then(address => {
            const fullLocation = { ...location, address };
            handleLocationSelect(fullLocation);
          });
        },
        (error) => {
          console.error('Error getting location:', error);
        }
      );
    }
  };

  const reverseGeocode = async (location: { latitude: number; longitude: number }) => {
    // 这里应该调用地理编码服务
    // 简化示例，实际应该调用真实的地理编码API
    return `${location.latitude.toFixed(6)}, ${location.longitude.toFixed(6)}`;
  };

  return (
    <div className="location-picker">
      <Space.Compact style={{ width: '100%' }}>
        <Input
          value={currentLocation?.address || (currentLocation ? `${currentLocation.latitude}, ${currentLocation.longitude}` : '')}
          placeholder="点击选择位置"
          readOnly
          disabled={disabled}
        />
        <Button
          icon={<EnvironmentOutlined />}
          onClick={() => setModalVisible(true)}
          disabled={disabled}
        >
          选择
        </Button>
        <Button onClick={getCurrentPosition} disabled={disabled}>
          当前位置
        </Button>
      </Space.Compact>

      <Modal
        title="选择位置"
        open={modalVisible}
        onCancel={() => setModalVisible(false)}
        width={800}
        footer={null}
      >
        <MapComponent
          mapType={mapType}
          zoom={zoom}
          initialLocation={currentLocation}
          onLocationSelect={handleLocationSelect}
        />
      </Modal>
    </div>
  );
});

// 地图组件
const MapComponent = ({ mapType, zoom, initialLocation, onLocationSelect }) => {
  // 这里应该根据 mapType 渲染不同的地图组件
  // 简化示例
  return (
    <div style={{ height: 400, background: '#f0f0f0', display: 'flex', alignItems: 'center', justifyContent: 'center' }}>
      <div>地图组件 ({mapType})</div>
    </div>
  );
};
```

### 地理位置后端字段

```typescript
// src/server/fields/LocationField.ts
import { DataTypes } from 'sequelize';
import { BaseColumnFieldOptions, Field } from '@nocobase/database';

export interface LocationFieldOptions extends BaseColumnFieldOptions {
  type: 'location';
}

export class LocationField extends Field {
  get dataType() {
    // 使用 JSON 类型存储地理位置数据
    return DataTypes.JSON;
  }

  init() {
    // 添加验证规则
    this.addValidation('locationValidation', (value) => {
      if (!value) return true;
      
      if (typeof value !== 'object') {
        throw new Error('Location field must be an object');
      }
      
      const { latitude, longitude } = value;
      
      if (typeof latitude !== 'number' || typeof longitude !== 'number') {
        throw new Error('Location must have valid latitude and longitude');
      }
      
      if (latitude < -90 || latitude > 90) {
        throw new Error('Latitude must be between -90 and 90');
      }
      
      if (longitude < -180 || longitude > 180) {
        throw new Error('Longitude must be between -180 and 180');
      }
      
      return true;
    });
  }

  // 添加地理位置查询操作符
  bind() {
    super.bind();
    
    // 注册自定义操作符
    this.database.operators.set('$near', (value, { field }) => {
      // 实现附近查询逻辑
      const { latitude, longitude, distance } = value;
      return {
        [field]: {
          // 这里应该使用数据库的地理位置查询功能
          // 例如 PostGIS 的 ST_DWithin 函数
        }
      };
    });
  }
}
```

## 插件集成和测试

### 插件主文件

```typescript
// src/client/index.tsx
import { Plugin } from '@nocobase/client';
import { RichTextEditor, RichTextReadPretty } from './components/RichTextEditor';
import { LocationPicker, LocationDisplay } from './components/LocationPicker';
import { RichTextInterface } from './interfaces/RichTextInterface';
import { LocationInterface } from './interfaces/LocationInterface';

export class MyFieldPlugin extends Plugin {
  async load() {
    // 注册组件
    this.app.addComponents({
      RichTextEditor,
      RichTextReadPretty,
      LocationPicker,
      LocationDisplay,
    });

    // 注册字段接口
    this.app.dataSourceManager.addFieldInterfaces([
      RichTextInterface,
      LocationInterface,
    ]);

    // 添加国际化
    this.app.i18n.addResources('zh-CN', 'my-field-plugin', {
      'Rich Text': '富文本',
      'Location': '地理位置',
      'Toolbar': '工具栏',
      'Height': '高度',
      'Map Type': '地图类型',
      'Default Zoom': '默认缩放',
    });
  }
}

export default MyFieldPlugin;
```

### 测试用例

```typescript
// tests/client/RichTextEditor.test.tsx
import React from 'react';
import { render, fireEvent, waitFor } from '@testing-library/react';
import { RichTextEditor } from '../../src/client/components/RichTextEditor';

describe('RichTextEditor', () => {
  it('should render correctly', () => {
    const { container } = render(
      <RichTextEditor value="" onChange={() => {}} />
    );
    
    expect(container.querySelector('.rich-text-editor')).toBeInTheDocument();
  });

  it('should handle value changes', async () => {
    const onChange = jest.fn();
    const { container } = render(
      <RichTextEditor value="" onChange={onChange} />
    );
    
    // 模拟编辑器内容变化
    // 注意：实际测试中需要模拟 TinyMCE 的行为
    
    await waitFor(() => {
      expect(onChange).toHaveBeenCalled();
    });
  });

  it('should sanitize HTML content', () => {
    const maliciousHTML = '<script>alert("xss")</script><p>Safe content</p>';
    const { container } = render(
      <RichTextEditor value={maliciousHTML} onChange={() => {}} />
    );
    
    // 验证脚本标签被移除
    expect(container.innerHTML).not.toContain('<script>');
    expect(container.innerHTML).toContain('Safe content');
  });
});
```

---

*本文档详细介绍了字段插件的开发流程，从简单的文本字段到复杂的富文本和地理位置字段，提供了完整的实现示例。*
