# 03. 创建共享包指南

本章将手把手教你创建一个可复用的共享组件包。

## 步骤概览

1. 创建包目录结构
2. 配置 package.json
3. 编写导出代码
4. 在其他项目中引用
5. 测试验证

## 完整示例：创建共享组件包

### 第 1 步：创建目录结构

```bash
# 在 packages 目录下创建新包
mkdir -p packages/my-shared-package/src/{components,utils,composables}
```

**推荐结构**：
```
packages/my-shared-package/
├── src/
│   ├── components/      # Vue 组件
│   ├── utils/          # 工具函数
│   ├── composables/    # Vue 组合式函数
│   ├── types/          # TypeScript 类型
│   └── index.ts        # 主入口文件
├── package.json
└── README.md
```

### 第 2 步：配置 package.json

#### 基础配置

```json
// packages/my-shared-package/package.json
{
  "name": "@fcd/my-shared",
  "version": "1.0.0",
  "type": "module",
  "private": true,
  "description": "共享组件和工具库"
}
```

**关键字段**：
- `name`: 包名，建议使用作用域 `@fcd/xxx`
- `private`: 设为 `true` 防止误发布到 npm
- `type`: `"module"` 使用 ES 模块

#### 添加依赖

```json
{
  "dependencies": {
    "vue": "catalog:",
    "vue-router": "catalog:"
  },
  "devDependencies": {
    "typescript": "catalog:",
    "vite": "catalog:"
  }
}
```

**依赖使用 `catalog:`**：
- 确保 workspace 中所有包使用相同版本
- 便于统一升级

### 第 3 步：编写代码

#### 创建工具函数

```typescript
// src/utils/format.ts
/**
 * 格式化日期
 */
export function formatDate(date: Date, format = 'YYYY-MM-DD'): string {
  const year = date.getFullYear();
  const month = String(date.getMonth() + 1).padStart(2, '0');
  const day = String(date.getDate()).padStart(2, '0');

  return format
    .replace('YYYY', String(year))
    .replace('MM', month)
    .replace('DD', day);
}

/**
 * 格式化数字
 */
export function formatNumber(num: number): string {
  return num.toLocaleString('zh-CN');
}
```

#### 创建 Vue 组件

```vue
<!-- src/components/Loading.vue -->
<template>
  <div v-if="loading" class="loading-overlay">
    <div class="spinner"></div>
    <p>{{ text }}</p>
  </div>
</template>

<script setup lang="ts">
interface Props {
  loading?: boolean;
  text?: string;
}

withDefaults(defineProps<Props>(), {
  loading: true,
  text: '加载中...',
});
</script>

<style scoped>
.loading-overlay {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 2rem;
}

.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid #f3f3f3;
  border-top: 4px solid #42b883;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
</style>
```

#### 创建 Composable

```typescript
// src/composables/useLoading.ts
import { ref } from 'vue';

export function useLoading(initialState = false) {
  const loading = ref(initialState);

  const start = () => {
    loading.value = true;
  };

  const stop = () => {
    loading.value = false;
  };

  return {
    loading,
    start,
    stop,
  };
}
```

#### 创建入口文件

```typescript
// src/index.ts
// 导出工具函数
export * from './utils/format';
export * from './utils/validate';

// 导出组件
export { default as Loading } from './components/Loading.vue';

// 导出 composables
export * from './composables/useLoading';

// 导出类型
export * from './types';
```

### 第 4 步：配置 exports

```json
// package.json
{
  "exports": {
    ".": {
      "types": "./src/index.ts",
      "development": "./src/index.ts",
      "default": "./src/index.ts"
    },
    "./components/*": "./src/components/*",
    "./utils": {
      "types": "./src/utils/index.ts",
      "development": "./src/utils/index.ts",
      "default": "./src/utils/index.ts"
    }
  }
}
```

### 第 5 步：在应用中引用

#### 添加依赖

```json
// apps/my-app/package.json
{
  "dependencies": {
    "@fcd/my-shared": "workspace:*"
  }
}
```

#### 使用导入

```typescript
// apps/my-app/src/main.ts
import { createApp } from 'vue';
import { formatDate, formatNumber } from '@fcd/my-shared/utils';
import Loading from '@fcd/my-shared/components/Loading.vue';

// 或使用统一入口
import { formatDate, Loading, useLoading } from '@fcd/my-shared';

const app = createApp(App);

// 注册全局组件
app.component('Loading', Loading);

app.mount('#app');
```

```vue
<!-- apps/my-app/src/App.vue -->
<script setup lang="ts">
import { ref } from 'vue';
import { useLoading } from '@fcd/my-shared';

const { loading, start, stop } = useLoading();

const fetchData = async () => {
  start();
  try {
    // 模拟 API 调用
    await new Promise(resolve => setTimeout(resolve, 2000));
  } finally {
    stop();
  }
};
</script>

<template>
  <div>
    <Loading v-if="loading" text="正在加载数据..." />
    <button @click="fetchData">获取数据</button>
  </div>
</template>
```

### 第 6 步：安装依赖并测试

```bash
# 安装依赖
pnpm install

# 启动开发服务器
pnpm dev:my-app

# 测试热更新
# 修改 packages/my-shared-package/src/components/Loading.vue
# 浏览器应自动刷新
```

## 不同类型的共享包

### 1. 组件库包

```
packages/ui-components/
├── src/
│   ├── Button.vue
│   ├── Input.vue
│   ├── Select.vue
│   └── index.ts
```

**使用场景**：跨应用的 UI 组件

### 2. 工具函数包

```
packages/utils/
├── src/
│   ├── format.ts
│   ├── validate.ts
│   ├── request.ts
│   └── index.ts
```

**使用场景**：纯函数，无框架依赖

### 3. 类型定义包

```
packages/types/
├── src/
│   ├── api.ts
│   ├── models.ts
│   └── index.ts
```

**使用场景**：TypeScript 类型共享

### 4. 业务逻辑包

```
packages/business/
├── src/
│   ├── user/
│   ├── auth/
│   └── products/
```

**使用场景**：跨应用的业务逻辑

## 开发模式 vs 生产模式

### 开发模式（推荐）

```json
{
  "exports": {
    ".": {
      "development": "./src/index.ts",  // 开发时直接使用源码
      "default": "./dist/index.js"      // 生产时使用编译后的代码
    }
  }
}
```

**优点**：
- 热更新即时生效
- TypeScript 类型提示完整
- 调试方便

### 生产模式

```bash
# 构建共享包
pnpm --filter @fcd/my-shared build

# 输出到 dist 目录
packages/my-shared-package/
├── dist/
│   ├── index.js
│   ├── components/
│   └── utils/
```

## 常见问题

### Q1: 组件样式丢失

**原因**：Vite 可能没有正确处理 `.vue` 文件的样式

**解决**：确保 Vite 配置中包含 `@vitejs/plugin-vue`

### Q2: 类型提示不生效

**检查**：
1. `tsconfig.json` 中是否包含包的路径
2. `package.json` 的 `exports` 中是否指定了 `types`

```json
{
  "exports": {
    ".": {
      "types": "./src/index.ts",  // 确保指定 types
      "default": "./src/index.ts"
    }
  }
}
```

### Q3: 热更新不工作

**解决**：重启开发服务器
```bash
# 停止当前服务，重新启动
pnpm dev:my-app
```

## 下一步

现在你已学会创建共享包，让我们深入了解：
- [exports 配置详解](./04-exports-configuration.md)
