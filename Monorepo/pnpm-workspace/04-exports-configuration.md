# 04. exports 配置详解

`exports` 字段是 package.json 中最重要的配置之一，它控制着包的哪些部分可以被外部访问。

## 什么是 exports

`exports` 定义了包的"公共接口"，决定了其他项目如何导入你的包。

### 基本语法

```json
{
  "exports": {
    ".": "./index.js"
  }
}
```

**含义**：当你导入 `@fcd/my-shared` 时，实际导入的是 `./index.js`

## exports 语法详解

### 1. 字符串语法（简单导出）

```json
{
  "exports": {
    ".": "./src/index.ts",
    "./utils": "./src/utils/index.ts",
    "./components/*": "./src/components/*"
  }
}
```

**使用**：
```typescript
import { something } from '@fcd/my-shared';           // -> ./src/index.ts
import { format } from '@fcd/my-shared/utils';        // -> ./src/utils/index.ts
import Button from '@fcd/my-shared/components/Button'; // -> ./src/components/Button
```

### 2. 对象语法（条件导出）

```json
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",      // TypeScript 类型
      "development": "./src/index.ts",   // 开发环境
      "production": "./dist/index.js",   // 生产环境
      "default": "./dist/index.js"       // 默认（兜底）
    }
  }
}
```

**条件优先级**：
1. `types` - TypeScript 类型检查
2. `development` - 开发模式（`NODE_ENV=development`）
3. `production` - 生产模式（`NODE_ENV=production`）
4. `default` - 其他情况的默认值

### 3. 通配符语法

```json
{
  "exports": {
    "./components/*": "./src/components/*.vue",
    "./utils/*": "./src/utils/*.ts",
    "./*": "./src/*.ts"
  }
}
```

**使用**：
```typescript
import Button from '@fcd/my-shared/components/Button';
import Input from '@fcd/my-shared/components/Input';
import { format } from '@fcd/my-shared/utils/date';
```

## 实战配置示例

### 示例 1：Vue 组件库

```json
{
  "name": "@fcd/ui-components",
  "exports": {
    ".": {
      "types": "./src/index.ts",
      "development": "./src/index.ts",
      "default": "./dist/index.js"
    },
    "./components/*": "./src/components/*.vue",
    "./styles": "./src/styles/index.css"
  }
}
```

**使用方式**：
```typescript
// 导入主入口
import { Button, Card } from '@fcd/ui-components';

// 导入单个组件
import Button from '@fcd/ui-components/components/Button';

// 导入样式
import '@fcd/ui-components/styles';
```

### 示例 2：工具函数库

```json
{
  "name": "@fcd/utils",
  "exports": {
    ".": "./src/index.ts",
    "./format": {
      "types": "./src/format/index.ts",
      "development": "./src/format/index.ts",
      "default": "./dist/format/index.js"
    },
    "./validate": "./src/validate/index.ts",
    "./request": "./src/request/index.ts"
  }
}
```

**使用方式**：
```typescript
import { format } from '@fcd/utils/format';
import { validateEmail } from '@fcd/utils/validate';
import { request } from '@fcd/utils/request';
```

### 示例 3：类型定义包

```json
{
  "name": "@fcd/types",
  "exports": {
    ".": "./src/index.ts",
    "./api": "./src/api/types.ts",
    "./models": "./src/models/index.ts",
    "./store": "./src/store/types.ts"
  },
  "typesVersions": {
    "*": {
      "api": ["src/api/types.ts"],
      "models": ["src/models/index.ts"]
    }
  }
}
```

**使用方式**：
```typescript
import type { User } from '@fcd/types/models';
import type { ApiResponse } from '@fcd/types/api';
```

## 开发模式 vs 生产模式

### 开发模式配置

```json
{
  "exports": {
    ".": {
      "types": "./src/index.ts",
      "development": "./src/index.ts",  // 使用源码
      "default": "./dist/index.js"
    }
  }
}
```

**优势**：
- ✅ 热更新即时生效
- ✅ 源码映射方便调试
- ✅ TypeScript 类型提示更准确

### 生产模式配置

```json
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"  // 使用编译后的代码
    }
  }
}
```

**优势**：
- ✅ 文件体积更小
- ✅ 性能更好
- ✅ 代码已转译，兼容性更好

## 条件导出详解

### 环境条件

```json
{
  "exports": {
    ".": {
      "node": "./dist/node.js",
      "browser": "./dist/browser.js",
      "default": "./dist/index.js"
    }
  }
}
```

**运行时判断**：
```typescript
// Node.js 环境
import { something } from '@fcd/my-shared';  // -> ./dist/node.js

// 浏览器环境
import { something } from '@fcd/my-shared';  // -> ./dist/browser.js
```

### Import 条件

```json
{
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    }
  }
}
```

**区分 ESM 和 CommonJS**：
```typescript
// ESM import
import { something } from '@fcd/my-shared';  // -> ./dist/index.js

// CommonJS require
const { something } = require('@fcd/my-shared');  // -> ./dist/index.cjs
```

## 子路径导出

### 定义子路径

```json
{
  "exports": {
    ".": "./src/index.ts",
    "./Button": "./src/components/Button.vue",
    "./Input": "./src/components/Input.vue",
    "./Card": "./src/components/Card.vue"
  }
}
```

**使用方式**：
```typescript
import Button from '@fcd/my-shared/Button';
import Input from '@fcd/my-shared/Input';
```

### 子路径 exports vs 直接导出

**子路径 exports**（推荐）：
```json
{
  "exports": {
    "./Button": "./src/Button.vue"
  }
}
```

**直接导出**（不推荐）：
```json
// ❌ 这样不行！
{
  "exports": {
    "./Button.vue": "./src/Button.vue"
  }
}
```

**原因**：Vite 可能无法正确解析 `.vue` 扩展名

## 最佳实践

### 1. 始终提供 types 字段

```json
{
  "exports": {
    ".": {
      "types": "./src/index.ts",  // ✅ 好的
      "default": "./src/index.ts"
    }
  }
}
```

### 2. 开发时使用源码

```json
{
  "exports": {
    ".": {
      "development": "./src/index.ts",  // ✅ 开发时用源码
      "default": "./dist/index.js"       // 生产时用编译产物
    }
  }
}
```

### 3. 使用通配符减少配置

```json
{
  "exports": {
    // ✅ 简洁
    "./components/*": "./src/components/*.vue",

    // ❌ 繁琐
    "./components/Button": "./src/components/Button.vue",
    "./components/Input": "./src/components/Input.vue",
    "./components/Card": "./src/components/Card.vue"
  }
}
```

### 4. 提供清晰的导出结构

```json
{
  "exports": {
    ".": "./src/index.ts",
    "./components/*": "./src/components/*.vue",
    "./utils": "./src/utils/index.ts",
    "./types": "./src/types/index.ts"
  }
}
```

**使用方清晰**：
```typescript
// 组件
import Button from '@fcd/shared/components/Button';

// 工具
import { format } from '@fcd/shared/utils';

// 类型
import type { User } from '@fcd/shared/types';
```

## 调试 exports 问题

### 问题 1：找不到模块

```
Error: Cannot find module '@fcd/my-shared/Button'
```

**检查**：
1. `package.json` 的 `exports` 是否包含 `./Button`
2. 文件是否真实存在

### 问题 2：类型提示不工作

**解决**：确保有 `types` 字段

```json
{
  "exports": {
    ".": {
      "types": "./src/index.ts",  // 确保有这行
      "default": "./src/index.ts"
    }
  }
}
```

### 问题 3：.vue 文件无法导入

**正确方式**：
```json
{
  "exports": {
    "./components/*": "./src/components/*"
  }
}
```

```typescript
// ✅ 正确
import Button from '@fcd/my-shared/components/Button.vue';

// ❌ 错误
import Button from '@fcd/my-shared/Button';
```

## 下一步

理解了 exports 配置后，让我们学习：
- [组件共享最佳实践](./05-best-practices.md)
