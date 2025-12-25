# Monorepo 与共享组件包完全指南

本指南将带你深入理解 Monorepo 架构和如何在多个项目间共享代码组件。

## 目录

1. [Monorepo 基础概念](./01-monorepo-basics.md)
   - 什么是 Monorepo
   - Monorepo vs Polyrepo
   - 为什么选择 Monorepo

2. [pnpm Workspace 详解](./02-pnpm-workspace.md)
   - pnpm workspace 配置
   - workspace 协议
   - catalog 依赖管理

3. [创建共享包指南](./03-creating-shared-packages.md)
   - 包结构设计
   - 配置文件详解
   - 发布与引用流程

4. [exports 配置详解](./04-exports-configuration.md)
   - exports 字段语法
   - 条件导出
   - 子路径导出

5. [组件共享最佳实践](./05-best-practices.md)
   - 目录组织规范
   - 命名约定
   - 类型安全
   - 性能优化

## 快速开始

如果你是第一次接触 Monorepo，建议按顺序阅读每一章。

如果你只想快速了解如何创建共享组件，可以直接跳转到 [创建共享包指南](./03-creating-shared-packages.md)。

## 项目结构参考

```
fcd_web/
├── apps/                    # 应用程序
│   ├── finder/             # 主应用
│   ├── hello-world/        # 示例应用
│   └── ...
├── packages/               # 共享包
│   ├── @core/             # 核心包
│   ├── shared-components/ # 共享组件
│   └── ...
├── internal/              # 内部工具
├── pnpm-workspace.yaml    # workspace 配置
└── package.json           # 根配置
```

## 学习目标

完成本指南后，你将能够：

- 理解 Monorepo 的核心概念和优势
- 熟练使用 pnpm workspace 管理多包项目
- 创建和维护可复用的共享组件包
- 正确配置 package.json 的 exports 字段
- 遵循最佳实践构建高质量的代码共享架构

---

> 💡 **提示**：本文档基于项目实际代码编写，所有示例都是可运行的实战代码。
