# NocoBase 开发文档

## 项目概述

NocoBase 是一个可扩展性优先的开源无代码开发平台。通过几分钟的部署，您就可以拥有一个私有、可控、极易扩展的无代码开发平台！

### 核心特性

- **数据模型驱动**: 采用数据结构与使用界面分离的设计思路
- **所见即所得**: 可视化配置界面，支持实时预览
- **插件化架构**: 所有功能都通过插件实现，扩展性极强
- **多数据库支持**: 支持 MySQL、PostgreSQL、SQLite 等主流数据库
- **RESTful API**: 基于资源和操作的 API 设计模式

### 技术栈

#### 前端技术栈
- **React 18**: 现代化的前端框架
- **TypeScript**: 类型安全的 JavaScript 超集
- **Ant Design 5.24.2**: 企业级 UI 设计语言和组件库
- **Formily**: 阿里巴巴统一前端表单解决方案
- **Axios**: HTTP 客户端库
- **React Router Dom**: 前端路由管理

#### 后端技术栈
- **Node.js (>=18)**: JavaScript 运行时环境
- **Koa.js**: 轻量级 Web 框架
- **TypeScript**: 类型安全的开发体验
- **Sequelize**: ORM 数据库操作库
- **WebSocket**: 实时通信支持

### 项目结构

```
nocobase/
├── packages/
│   ├── core/                 # 核心包
│   │   ├── client/           # 前端核心
│   │   ├── server/           # 后端核心
│   │   ├── database/         # 数据库层
│   │   ├── resourcer/        # 资源管理
│   │   ├── actions/          # 操作处理
│   │   ├── auth/             # 认证系统
│   │   └── acl/              # 访问控制
│   ├── plugins/              # 插件包
│   │   └── @nocobase/        # 官方插件
│   └── presets/              # 预设包
├── examples/                 # 示例代码
├── docker/                   # Docker 配置
└── docs/                     # 文档目录
```

## 快速开始

### 环境要求

- Node.js >= 18
- Yarn 1.22.19
- 数据库 (MySQL/PostgreSQL/SQLite)

### 安装依赖

```bash
# 安装依赖
yarn install

# 安装后处理
yarn postinstall
```

### 开发环境启动

```bash
# 启动开发服务器
yarn dev

# 仅启动后端服务器
yarn dev-server
```

### 生产环境构建

```bash
# 构建项目
yarn build

# 启动生产服务器
yarn start
```

## 文档导航

### 📚 开发指南
- [架构设计](./architecture.md) - 了解 NocoBase 的整体架构
- [前端开发指南](./frontend/guide.md) - 前端开发最佳实践
- [后端开发指南](./backend/guide.md) - 后端开发最佳实践
- [插件开发指南](./plugins/development.md) - 如何开发自定义插件

### 🔧 API 参考
- [前端 API](./api/frontend.md) - 前端 API 接口文档
- [后端 API](./api/backend.md) - 后端 API 接口文档
- [数据库 API](./backend/database.md) - 数据库操作 API

### 🚀 部署运维
- [部署指南](./deployment.md) - 生产环境部署指南
- [配置说明](./configuration.md) - 系统配置参数说明

### 💡 示例代码
- [插件示例](./plugins/examples.md) - 常见插件开发示例
- [API 使用示例](./api/examples.md) - API 调用示例

## 开发规范

### 代码规范
- 使用 TypeScript 进行类型安全开发
- 遵循 ESLint 代码规范
- 使用 Prettier 进行代码格式化
- 编写单元测试和集成测试

### Git 提交规范
- 使用 Conventional Commits 规范
- 提交前自动运行 lint 检查
- 每个 PR 需要通过 CI/CD 检查

### 版本管理
- 使用 Lerna 进行多包管理
- 遵循语义化版本控制 (SemVer)
- 定期发布 Alpha 和正式版本

## 社区资源

- **官方网站**: https://www.nocobase.com/
- **在线演示**: https://demo.nocobase.com/new
- **官方文档**: https://docs.nocobase.com/
- **GitHub**: https://github.com/nocobase/nocobase
- **论坛**: https://forum.nocobase.com/

## 许可证

本项目采用 AGPL-3.0 开源许可证。详情请参阅 [LICENSE](../LICENSE-AGPL.txt) 文件。

## 贡献指南

欢迎贡献代码！请先阅读我们的贡献指南，了解如何参与项目开发。

1. Fork 项目仓库
2. 创建功能分支
3. 提交代码变更
4. 创建 Pull Request
5. 等待代码审查

---

*最后更新时间: 2025-01-04*
