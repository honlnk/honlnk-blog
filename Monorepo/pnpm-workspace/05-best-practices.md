# 05. 组件共享最佳实践

本章总结了在 Monorepo 中共享代码和组件的最佳实践。

## 目录组织规范

### 推荐的目录结构

```
packages/
├── @core/                    # 核心包（项目内部）
│   ├── base/                # 基础功能
│   │   └── shared/          # 共享基础
│   ├── composables/         # 组合式函数
│   ├── ui-kit/              # UI 工具包
│   └── preferences/         # 偏好设置
├── ui-components/           # 通用 UI 组件
├── utils/                   # 工具函数
├── types/                   # 类型定义
└── business/                # 业务逻辑
```

### 包的分类原则

#### 1. 核心包 (@core)

**用途**：项目基础设施，不对外发布

```
@core/
├── base/shared/       # 缓存、状态管理
├── composables/       # Vue composables
├── ui-kit/            # UI 工具
└── preferences/       # 应用配置
```

**特点**：
- 只在项目内部使用
- 与业务紧密耦合
- 不考虑向后兼容

#### 2. 通用组件包

**用途**：跨应用的 UI 组件

```
ui-components/
├── Button.vue
├── Input.vue
├── Card.vue
└── Modal.vue
```

**特点**：
- 高度可复用
- 与业务解耦
- 考虑通用性和可配置性

#### 3. 工具函数包

**用途**：纯函数，无副作用

```
utils/
├── format/          # 格式化
├── validate/        # 验证
├── request/         # 请求
└── storage/         # 存储
```

**特点**：
- 无框架依赖
- 易于测试
- 可独立使用

## 命名约定

### 包名规范

#### 使用作用域

```json
{
  // ✅ 好的命名
  "name": "@fcd/ui-components",
  "name": "@fcd/utils",

  // ❌ 不好的命名
  "name": "ui-components",
  "name": "utils"
}
```

**好处**：
- 避免命名冲突
- 清晰表明来源
- 防止误发布到 npm

### 导出路径规范

#### 按功能组织导出

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

**使用方式**：
```typescript
import Button from '@fcd/ui-components/components/Button';
import { format } from '@fcd/ui-components/utils';
import type { User } from '@fcd/ui-components/types';
```

### 组件命名

#### PascalCase for 组件

```vue
<!-- ✅ 好的命名 -->
<Button.vue />
<UserCard.vue />
<DataTable.vue />

<!-- ❌ 不好的命名 -->
<button.vue />
<userCard.vue />
<data-table.vue />
```

#### 文件名与组件名一致

```vue
<!-- Button.vue -->
<script setup lang="ts">
// ✅ 组件名与文件名一致
const __name = 'Button';  // 可选，Vue 会自动推断
</script>
```

## 组件设计原则

### 1. 单一职责

```vue
<!-- ✅ 好的组件：只做一件事 -->
<LoadingSpinner.vue>
<ErrorMessage.vue>
<SubmitButton.vue>

<!-- ❌ 不好的组件：职责过多 -->
<FormComponent.vue>  <!-- 既处理表单，又处理验证，还处理提交 -->
```

### 2. Props 设计

#### 使用 TypeScript 接口

```vue
<script setup lang="ts">
// ✅ 好的 Props 定义
interface Props {
  title: string;
  count?: number;
  disabled?: boolean;
  size?: 'small' | 'medium' | 'large';
}

const props = withDefaults(defineProps<Props>(), {
  count: 0,
  disabled: false,
  size: 'medium',
});
</script>
```

#### 提供合理的默认值

```vue
<script setup lang="ts">
// ✅ 好的默认值
const props = withDefaults(defineProps<Props>(), {
  count: 0,
  disabled: false,
  size: 'medium',  // 最常用的选项
});

// ❌ 不好的默认值
const props = withDefaults(defineProps<Props>(), {
  count: undefined,     // 应该有实际默认值
  size: 'large',        // 不应该是冷门选项
});
</script>
```

### 3. 事件设计

#### 使用 TypeScript 定义事件

```vue
<script setup lang="ts">
// ✅ 好的事件定义
interface Emits {
  (e: 'click', value: number): void;
  (e: 'submit', data: FormData): void;
  (e: 'cancel'): void;
}

const emit = defineEmits<Emits>();
</script>
```

#### 事件名语义化

```vue
<script setup lang="ts">
// ✅ 好的事件名
emit('submit', formData);
emit('cancel');
emit('update:modelValue', newValue);

// ❌ 不好的事件名
emit('ok');
emit('done');
emit('callback');
</script>
```

### 4. 插槽设计

#### 提供灵活的插槽

```vue
<template>
  <Card>
    <!-- 主内容插槽 -->
    <slot />

    <!-- 命名插槽 -->
    <template #header>
      <slot name="header">
        <h3>{{ title }}</h3>  <!-- 默认内容 -->
      </slot>
    </template>

    <template #footer>
      <slot name="footer" />
    </template>
  </Card>
</template>
```

## 类型安全

### 导出类型

```typescript
// packages/types/src/user.ts
export interface User {
  id: string;
  name: string;
  email: string;
  avatar?: string;
}

export type UserRole = 'admin' | 'user' | 'guest';

export interface CreateUserDto {
  name: string;
  email: string;
  password: string;
}
```

### 使用类型

```typescript
// apps/finder/src/stores/user.ts
import type { User, UserRole } from '@fcd/types';

const user = ref<User | null>(null);
const role = ref<UserRole>('guest');
```

### 类型导出配置

```json
// packages/types/package.json
{
  "exports": {
    ".": "./src/index.ts",
    "./user": {
      "types": "./src/user/index.ts",  // 明确指定类型文件
      "default": "./src/user/index.ts"
    }
  }
}
```

## 性能优化

### 1. 按需加载

```typescript
// ❌ 不好的方式：导入整个库
import * as Utils from '@fcd/utils';

// ✅ 好的方式：按需导入
import { formatDate, formatNumber } from '@fcd/utils/format';
```

### 2. Tree-shaking 友好

```typescript
// ✅ 好的导出：支持 tree-shaking
export const formatDate = () => { };
export const formatNumber = () => { };

// ❌ 不好的导出：影响 tree-shaking
const utils = {
  formatDate: () => { },
  formatNumber: () => { },
};
export default utils;
```

### 3. 避免循环依赖

```
// ❌ 循环依赖
packages/a/index.ts -> packages/b/index.ts -> packages/a/index.ts

// ✅ 解决方案：提取公共部分
packages/a/index.ts -> packages/common/index.ts
packages/b/index.ts -> packages/common/index.ts
```

## 文档和注释

### 组件文档

```vue
<!--
  Button 组件

  @description 通用的按钮组件，支持多种类型和尺寸

  @props
  - type: 按钮类型 ('primary' | 'success' | 'warning' | 'danger')
  - size: 按钮尺寸 ('small' | 'medium' | 'large')
  - disabled: 是否禁用

  @events
  - click: 点击事件，返回 MouseEvent

  @example
  <Button type="primary" size="medium" @click="handleClick">
    点击我
  </Button>
-->
<script setup lang="ts">
// ...
</script>
```

### 函数注释

```typescript
/**
 * 格式化日期为指定格式
 * @param date - 要格式化的日期
 * @param format - 格式字符串，默认 'YYYY-MM-DD'
 * @returns 格式化后的日期字符串
 *
 * @example
 * formatDate(new Date(), 'YYYY-MM-DD') // '2024-12-25'
 */
export function formatDate(date: Date, format = 'YYYY-MM-DD'): string {
  // ...
}
```

## 测试

### 单元测试

```typescript
// packages/utils/__tests__/format.test.ts
import { describe, it, expect } from 'vitest';
import { formatDate, formatNumber } from '../src/format';

describe('formatDate', () => {
  it('should format date correctly', () => {
    const date = new Date('2024-12-25');
    expect(formatDate(date)).toBe('2024-12-25');
  });

  it('should handle custom format', () => {
    const date = new Date('2024-12-25');
    expect(formatDate(date, 'MM/DD/YYYY')).toBe('12/25/2024');
  });
});
```

## 版本管理

### 使用 Changesets

```bash
# 1. 添加 changeset
pnpm changeset

# 2. 生成版本号
pnpm changeset version

# 3. 发布
pnpm changeset publish
```

### 语义化版本

```
1.0.0  ->  1.0.1  (patch: 修复 bug)
1.0.1  ->  1.1.0  (minor: 新增功能，向后兼容)
1.1.0  ->  2.0.0  (major: 破坏性变更)
```

## 常见陷阱

### 1. 隐式依赖

```typescript
// ❌ 组件隐式依赖全局样式
// packages/shared-components/Button.vue
<template>
  <button class="btn">Click</button>  <!-- 依赖全局 .btn 样式 -->
</template>

// ✅ 组件自带样式
<template>
  <button class="button">Click</button>  <!-- 使用 scoped 样式 -->
</template>

<style scoped>
.button { /* ... */ }
</style>
```

### 2. 过度抽象

```typescript
// ❌ 过度抽象
class AbstractButtonFactory {
  createButton(config: ButtonConfig): Button { }
  // 100+ 行的抽象代码
}

// ✅ 简单直接
const Button = (props: ButtonProps) => {
  return <button {...props} />;
};
```

### 3. 忽视 TypeScript

```typescript
// ❌ 类型不安全
export const processData = (data: any) => { };

// ✅ 类型安全
interface Data {
  id: string;
  value: number;
}
export const processData = (data: Data) => { };
```

## 检查清单

发布共享包前，确保：

- [ ] 包名符合命名规范
- [ ] exports 配置正确
- [ ] 类型定义完整
- [ ] 所有导出都有类型
- [ ] 文档注释齐全
- [ ] 单元测试覆盖
- [ ] 无循环依赖
- [ ] 无隐式依赖
- [ ] Tree-shaking 友好
- [ ] 版本号正确更新

## 总结

优秀的共享代码应该：

1. **清晰**：命名明确，结构清晰
2. **类型安全**：完整的 TypeScript 支持
3. **独立**：最小化外部依赖
4. **可测试**：易于编写和运行测试
5. **文档化**：良好的注释和文档
6. **可维护**：代码简洁，易于理解

---

🎉 恭喜！你已经完成了 Monorepo 与共享组件包完全指南的学习。

现在你可以：
- 创建和维护 Monorepo 项目
- 开发可复用的共享组件
- 正确配置 package.json 的 exports
- 遵循最佳实践构建高质量的代码架构
