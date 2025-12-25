# 01. Monorepo 基础概念

## 什么是 Monorepo

**Monorepo**（Monolithic Repository）是一种将多个相关项目存储在同一个代码仓库中的策略。

在我们的项目中，你可以看到：

```
fcd_web/
├── apps/                    # 多个应用
│   ├── finder/             # Finder 主应用
│   ├── hello-world/        # Hello World 示例
│   └── backend-mock/       # Mock 服务器
├── packages/               # 共享的包
│   ├── @core/             # 核心工具库
│   ├── shared-components/ # 共享组件
│   └── utils/             # 工具函数
```

**一个仓库，多个项目** = Monorepo

## Monorepo vs Polyrepo

### Polyrepo（多仓库）架构

```
company-projects/
├── repo-app-a.git/     # 独立仓库 A
├── repo-app-b.git/     # 独立仓库 B
└── repo-shared.git/    # 共享代码仓库
```

**特点**：
- 每个项目一个仓库
- 共享代码需要独立的仓库
- 需要发布到 npm 才能使用

**问题**：
- 修改共享代码需要同步多个仓库
- 版本管理复杂
- 依赖冲突难解决

### Monorepo（单仓库）架构

```
company-mono/
├── apps/
│   ├── app-a/
│   └── app-b/
└── packages/
    └── shared/
```

**优势**：
- ✅ 所有代码在一处，便于重构
- ✅ 原子化提交：一次提交包含所有改动
- ✅ 统一的依赖管理
- ✅ 代码共享更简单
- ✅ 跨项目的 CI/CD

## 为什么选择 Monorepo

### 1. 代码共享更简单

**Polyrepo 中**：
```bash
# 1. 修改共享库
cd repo-shared/
vim utils.ts

# 2. 发布新版本
npm version patch
npm publish

# 3. 更新所有依赖项目
cd ../repo-app-a/
npm update shared-lib

cd ../repo-app-b/
npm update shared-lib
```

**Monorepo 中**：
```bash
# 1. 修改共享库，直接生效
vim packages/shared/utils.ts

# 2. 所有应用自动使用最新代码
# 无需发布，无需更新
```

### 2. 统一的版本管理

```bash
# 只有一个 package.json
pnpm install  # 一次安装所有依赖

# 统一的依赖版本
{
  "vue": "catalog:"  # 所有项目使用相同版本
}
```

### 3. 原子化提交

```
commit abc123: "重构：统一按钮组件样式"

apps/finder/src/components/Button.vue    ✓
apps/hello-world/src/app.vue              ✓
packages/shared-components/Button.vue     ✓
```

**一次提交，完整功能**，不会出现不同应用使用不同版本的问题。

### 4. 跨项目重构

IDE 可以在整个仓库中重构：

```
重构前：
packages/shared/utils/formatDate()

全局重命名为：
packages/shared/utils/date/format()

自动更新所有引用：
apps/finder/src/store/user.ts         ✓
apps/hello-world/src/app.vue           ✓
packages/another-package/helper.ts     ✓
```

## 常见的 Monorepo 工具

### 1. pnpm workspace

我们项目使用的工具！特点：
- ✅ 节省磁盘空间（硬链接）
- ✅ 速度快
- ✅ 严格的依赖管理

### 2. Turborepo

用于构建加速：
```json
{
  "scripts": {
    "build": "turbo build"  // 并行构建所有项目
  }
}
```

### 3. 其他工具

- **Lerna**：老牌工具，功能全面
- **Nx**：企业级方案，功能强大
- **Yarn Workspaces**：Yarn 自带方案

## 什么时候应该用 Monorepo

### 适合 Monorepo 的场景

- ✅ 项目之间有共享代码
- ✅ 团队希望统一技术栈
- ✅ 需要频繁跨项目修改
- ✅ 希望简化 CI/CD

### 不适合 Monorepo 的场景

- ❌ 项目完全独立，无任何共享
- ❌ 团队规模极大，Git 性能成为瓶颈
- ❌ 项目使用完全不同的技术栈

## 本项目使用 Monorepo 的好处

在我们的 `fcd_web` 项目中：

1. **共享组件**：`@fcd/shared-components` 可在 finder、hello-world 中复用
2. **核心库**：`@vben-core/shared` 提供缓存、状态管理等功能
3. **类型安全**：TypeScript 类型在整个仓库中共享
4. **统一开发**：一次 `pnpm install` 安装所有依赖

## 下一步

理解了 Monorepo 的概念后，让我们学习：
- [pnpm Workspace 详解](./02-pnpm-workspace.md)
