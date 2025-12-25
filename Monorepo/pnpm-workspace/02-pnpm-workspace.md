# 02. pnpm Workspace 详解

## 什么是 pnpm Workspace

pnpm Workspace 是 pnpm 提供的 Monorepo 管理方案，它允许你在一个仓库中管理多个包，并自动处理它们之间的依赖关系。

## workspace 配置文件

### pnpm-workspace.yaml

项目根目录的 `pnpm-workspace.yaml` 定义了哪些目录属于 workspace：

```yaml
packages:
  - internal/*          # internal 目录下的所有包
  - packages/*          # packages 目录下的所有包
  - packages/@core/*    # packages/@core 目录下的所有包
  - apps/*              # apps 目录下的所有包
  - scripts/*           # scripts 目录下的所有包
```

**通配符 `*` 的含义**：
- `apps/*` 匹配 `apps/finder`、`apps/hello-world` 等
- `packages/@core/*` 匹配 `packages/@core/base`、`packages/@core/ui-kit` 等

## workspace: 协议

这是 pnpm workspace 的核心特性！

### 基本用法

在 `package.json` 中引用 workspace 中的包：

```json
// apps/finder/package.json
{
  "dependencies": {
    "@fcd/shared-components": "workspace:*"
  }
}
```

**`workspace:*` 的含义**：
- 告诉 pnpm："这是一个 workspace 中的包"
- 自动链接到本地的包，而不是从 npm 下载
- 修改共享包的代码会立即生效（热更新）

### workspace 版本范围

```json
{
  "dependencies": {
    // 精确匹配
    "shared": "workspace:1.0.0",

    // 匹配小版本
    "shared": "workspace:^1.0.0",

    // 匹配任意版本（最常用）
    "shared": "workspace:*"
  }
}
```

### 实际效果对比

**使用 `workspace:*`**：
```bash
$ pnpm install
# pnpm 创建符号链接，指向本地包
node_modules/@fcd/shared-components -> packages/shared-components
```

**不使用 `workspace:*`**（错误做法）：
```bash
$ pnpm install
# pnpm 会尝试从 npm 下载，可能找不到！
```

## catalog: 依赖管理

### 什么是 catalog

`catalog:` 是 pnpm 的一种依赖版本管理方式，在 `pnpm-workspace.yaml` 中统一定义依赖版本：

```yaml
# pnpm-workspace.yaml
catalog:
  vue: ^3.5.24
  vite: ^7.2.2
  typescript: ^5.9.3
  dayjs: ^1.11.13
```

### catalog 的好处

**不使用 catalog**（可能版本不一致）：
```json
// apps/finder/package.json
{ "dependencies": { "vue": "^3.5.20" } }

// apps/hello-world/package.json
{ "dependencies": { "vue": "^3.5.24" } }

// packages/shared/package.json
{ "dependencies": { "vue": "^3.5.22" } }
```

**使用 catalog**（统一版本）：
```json
// 所有 package.json
{
  "dependencies": {
    "vue": "catalog:"  // 统一使用 catalog 中定义的版本
  }
}
```

### catalog 配置示例

```yaml
# pnpm-workspace.yaml
catalog:
  # 框架核心
  'vue': ^3.5.24
  'vue-router': ^4.5.1
  'pinia': ^3.0.3

  # 构建工具
  'vite': ^7.2.2
  'typescript': ^5.9.3

  # UI 库
  'element-plus': ^2.11.5
  'naive-ui': ^2.42.0

  # 工具库
  'dayjs': ^1.11.13
  '@vueuse/core': ^13.4.0
```

## 常用命令

### 安装依赖

```bash
# 安装所有项目的依赖
pnpm install

# 只安装特定项目的依赖
pnpm --filter @vben/hello-world install

# 给特定项目添加依赖
pnpm --filter @vben/hello-world add lodash
```

### 运行脚本

```bash
# 运行根目录脚本
pnpm dev

# 运行特定项目的脚本
pnpm --filter @vben/hello-world dev

# 使用快捷方式（需要在根 package.json 配置）
pnpm dev:hello  # 等同于 pnpm --filter @vben/hello-world dev
```

### 构建

```bash
# 构建所有项目
pnpm build

# 构建特定项目
pnpm --filter @vben/hello-world build
```

## workspace 内部原理

### 符号链接

当你运行 `pnpm install` 时：

```
node_modules/
├── @fcd/
│   └── shared-components -> ../../../packages/shared-components
├── @vben-core/
│   ├── shared -> ../../../packages/@core/base/shared
│   └── ui-kit -> ../../../packages/@core/ui-kit
└── vue -> [store content]
```

**特点**：
- `workspace:*` 包是符号链接，指向实际包目录
- 外部包（如 `vue`）存储在全局 store，通过硬链接引用

### 节省空间

```
传统 npm:
apps/finder/node_modules/vue/        = 50 MB
apps/hello/node_modules/vue/         = 50 MB
packages/shared/node_modules/vue/    = 50 MB
总计: 150 MB

pnpm workspace:
~/.pnpm-store/vue/                   = 50 MB
所有项目通过硬链接引用
总计: 50 MB
```

## 过滤器 (filter)

`--filter` 是 pnpm workspace 的强大功能：

```bash
# 按包名过滤
pnpm --filter @vben/hello-world dev

# 按目录过滤
pnpm --filter ./apps/hello-world dev

# 按模式匹配
pnpm --filter "@vben/*" build

# 包含依赖（构建依赖的所有包）
pnpm --filter @vben/hello-world... build

# 包含被依赖（构建所有依赖此包的项目）
pnpm --filter ...@vben/hello-world build
```

## 实战案例

### 案例 1：添加新应用

```bash
# 1. 创建应用目录
mkdir -p apps/my-new-app

# 2. 初始化 package.json
cd apps/my-new-app
pnpm init

# 3. 添加 workspace 依赖
pnpm add @fcd/shared-components --workspace-root
```

### 案例 2：共享代码修改立即生效

```bash
# 终端 1：启动应用
pnpm dev:hello

# 终端 2：修改共享组件
vim packages/shared-components/Button.vue

# 结果：浏览器自动刷新，看到修改！
```

### 案例 3：查看依赖关系

```bash
# 查看包的依赖图
pnpm list --depth 0

# 查看哪些包使用了特定依赖
pnpm why vue
```

## 常见问题

### Q1: workspace:* 和 npm 版本有什么区别？

```json
{
  "dependencies": {
    // 从 npm 下载
    "lodash": "^4.17.21",

    // 链接到本地包
    "@fcd/shared": "workspace:*"
  }
}
```

**区别**：
- `workspace:*` 只用于本地包，会创建符号链接
- npm 版本用于外部包，从 registry 下载

### Q2: 如何发布 workspace 包？

```bash
# 发布前，pnpm 会将 workspace:* 替换为实际版本
pnpm publish --filter @fcd/shared-components
```

### Q3: 为什么依赖没有更新？

```bash
# 清除缓存重新安装
pnpm install --force

# 或者
rm -rf node_modules pnpm-lock.yaml
pnpm install
```

## 下一步

现在你理解了 pnpm workspace，让我们学习：
- [创建共享包指南](./03-creating-shared-packages.md)
