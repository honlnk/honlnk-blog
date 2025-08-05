# Vue 3 横向无缝滚动组件实现详解

在前端开发中，横向滚动效果（跑马灯、公告栏等）是非常常见的需求。分享一个我做的 Vue 3 横向无缝滚动组件，它支持多种配置选项，性能优秀，可以轻松集成到你的项目中。

## 🌟 组件效果预览

这个组件可以实现：
- 自动检测内容是否溢出容器
- 无缝循环滚动效果
- 支持左右两个方向滚动
- 鼠标悬停暂停功能
- 可配置的滚动速度和等待时间
- 响应式设计，自动适配窗口变化

## 🛠️ 核心功能特性

### 1. 智能溢出检测
组件会自动检测内容是否超出容器宽度，只有在内容溢出时才开始滚动，避免不必要的动画。

### 2. 无缝循环滚动
通过复制一份内容，实现视觉上的无缝循环效果，用户体验更佳。

### 3. 灵活的配置选项
```typescript
interface Props {
  width?: number;           // 容器最大宽度
  speed?: number;           // 滚动速度 (px/s)
  pauseOnHover?: boolean;   // 鼠标悬停是否暂停
  direction?: 'left' | 'right'; // 滚动方向
  waitTime?: number;        // 滚动完毕后等待时间（毫秒）
}
```

### 4. 高性能动画
使用 `requestAnimationFrame` 实现流畅的动画效果，避免卡顿。

## 📦 组件完整代码

```vue
<script lang="ts" setup>
import { computed, nextTick, onMounted, onUnmounted, ref, watch } from 'vue';

interface Props {
  /** 最大宽度 */
  width?: number;
  /** 滚动速度，单位：px/s */
  speed?: number;
  /** 鼠标悬停时是否暂停 */
  pauseOnHover?: boolean;
  /** 滚动方向 */
  direction?: 'left' | 'right';
  /** 滚动完毕后等待时间，单位：毫秒 */
  waitTime?: number;
}

const props = withDefaults(defineProps<Props>(), {
  width: 100,
  speed: 60,
  pauseOnHover: true,
  direction: 'left',
  waitTime: 2000
});

const containerRef = ref<HTMLElement>();
const contentRef = ref<HTMLElement>();
const wrapperRef = ref<HTMLElement>();

const isOverflow = ref(false);
const animationId = ref<number>();
const waitTimeoutId = ref<number>();
const currentTranslateX = ref(0);
const isHovered = ref(false);
const isWaiting = ref(false);
const contentWidth = ref(0);
const containerWidth = ref(0);

const containerWidthStyle = computed(() => {
  return props.width === -1 ? '100%' : `${props.width}px`;
});

// 检查内容是否溢出
async function checkOverflow() {
  await nextTick();

  if (!containerRef.value || !contentRef.value) return;

  containerWidth.value = containerRef.value.offsetWidth;
  contentWidth.value = contentRef.value.scrollWidth;

  isOverflow.value = contentWidth.value > containerWidth.value;

  if (isOverflow.value) {
    currentTranslateX.value = 0;
    startAnimation();
  } else {
    stopAnimation();
    currentTranslateX.value = 0;
    // 重置位置
    if (wrapperRef.value) {
      wrapperRef.value.style.transform = 'translateX(0px)';
    }
  }
}

// 等待指定时间后重新开始滚动
function startWaitTimer() {
  isWaiting.value = true;
  waitTimeoutId.value = window.setTimeout(() => {
    isWaiting.value = false;
    currentTranslateX.value = 0;
    if (wrapperRef.value) {
      wrapperRef.value.style.transform = 'translateX(0px)';
    }
    startAnimation();
  }, props.waitTime);
}

// 开始动画
function startAnimation() {
  if (animationId.value) {
    cancelAnimationFrame(animationId.value);
  }

  let lastTime = 0;
  function animate(currentTime: number) {
    if (lastTime === 0) lastTime = currentTime;

    const deltaTime = currentTime - lastTime;
    lastTime = currentTime;

    if (!isHovered.value && isOverflow.value && !isWaiting.value) {
      // 计算移动距离
      const moveDistance = (props.speed * deltaTime) / 1000;

      if (props.direction === 'left') {
        currentTranslateX.value -= moveDistance;

        // 当内容完全移出视口时，开始等待计时器
        if (Math.abs(currentTranslateX.value) >= contentWidth.value + 16) {
          stopAnimation();
          startWaitTimer();
          return;
        }
      } else {
        currentTranslateX.value += moveDistance;

        // 向右滚动时的重置逻辑
        if (currentTranslateX.value >= 0) {
          stopAnimation();
          startWaitTimer();
          return;
        }
      }

      // 移动整个包装器
      if (wrapperRef.value) {
        wrapperRef.value.style.transform = `translateX(${currentTranslateX.value}px)`;
      }
    }

    animationId.value = requestAnimationFrame(animate);
  };

  animationId.value = requestAnimationFrame(animate);
}

// 停止动画
function stopAnimation() {
  if (animationId.value) {
    cancelAnimationFrame(animationId.value);
    animationId.value = undefined;
  }
}

// 清除等待计时器
function clearWaitTimer() {
  if (waitTimeoutId.value) {
    clearTimeout(waitTimeoutId.value);
    waitTimeoutId.value = undefined;
    isWaiting.value = false;
  }
}

// 鼠标悬停事件
function handleMouseEnter() {
  if (props.pauseOnHover) {
    isHovered.value = true;
    // 如果正在等待，暂停等待计时器
    if (isWaiting.value) {
      clearWaitTimer();
    }
  }
}

function handleMouseLeave() {
  if (props.pauseOnHover) {
    isHovered.value = false;
    // 如果之前在等待状态且鼠标离开，重新开始等待
    if (isWaiting.value) {
      startWaitTimer();
    }
  }
}

// 监听插槽内容变化
const slotObserver = ref<MutationObserver>();

onMounted(() => {
  checkOverflow();

  // 监听窗口大小变化
  window.addEventListener('resize', checkOverflow);

  // 监听插槽内容变化
  if (containerRef.value) {
    slotObserver.value = new MutationObserver(() => {
      checkOverflow();
    });

    slotObserver.value.observe(containerRef.value, {
      childList: true,
      subtree: true,
      characterData: true
    });
  }
});

onUnmounted(() => {
  stopAnimation();
  clearWaitTimer();
  window.removeEventListener('resize', checkOverflow);
  slotObserver.value?.disconnect();
});

// 监听相关属性变化
watch([() => props.speed, () => props.direction, () => props.waitTime], () => {
  if (isOverflow.value) {
    stopAnimation();
    clearWaitTimer();
    currentTranslateX.value = 0;
    if (wrapperRef.value) {
      wrapperRef.value.style.transform = 'translateX(0px)';
    }
    startAnimation();
  }
});
</script>

<template>
  <div
    ref="containerRef"
    class="relative select-none overflow-hidden"
    :style="{ width: containerWidthStyle }"
    @mouseenter="handleMouseEnter"
    @mouseleave="handleMouseLeave"
  >
    <div ref="wrapperRef" class="flex transition-none will-change-transform">
      <!-- 原始内容 -->
      <div ref="contentRef" class="flex whitespace-nowrap">
        <slot />
      </div>

      <!-- 复制的内容用于无限循环 -->
      <div v-if="isOverflow" class="ml-4 flex whitespace-nowrap">
        <slot />
      </div>
    </div>
  </div>
</template>
```

## 🎯 使用示例

```vue
<template>
  <div class="demo">
    <h3>基础用法</h3>
    <HorizontalScroll :speed="80" :wait-time="3000">
      <span class="text-red-500">这是很长很长的文本内容，会自动滚动显示...</span>
    </HorizontalScroll>

    <h3>向右滚动</h3>
    <HorizontalScroll direction="right" :speed="50">
      <span class="text-blue-500">向右滚动的内容展示效果</span>
    </HorizontalScroll>

    <h3>禁用悬停暂停</h3>
    <HorizontalScroll :pause-on-hover="false" :speed="100">
      <span class="text-green-500">即使鼠标悬停也不会暂停的滚动</span>
    </HorizontalScroll>
  </div>
</template>

<script setup>
import HorizontalScroll from './components/HorizontalScroll.vue'
</script>
```

## 🔧 核心实现原理

### 1. 动画控制
使用 `requestAnimationFrame` 实现高性能动画，通过计算帧时间差来控制移动距离：

```typescript
const moveDistance = (props.speed * deltaTime) / 1000;
```

### 2. 无缝循环
通过复制一份相同的内容，当第一份内容移出视口时，第二份内容正好接上，实现无缝效果。

### 3. 响应式更新
监听窗口大小变化和内容变化，自动重新计算是否需要滚动：

```javascript
window.addEventListener('resize', checkOverflow);
slotObserver.value = new MutationObserver(() => {
  checkOverflow();
});
```

## 🚀 性能优化点

1. **使用 `will-change: transform`** 提升动画性能
2. **`requestAnimationFrame` 替代 `setInterval`** 实现更流畅的动画
3. **智能暂停机制** 避免不必要的计算
4. **MutationObserver 监听配置变化** 实时响应动态配置

## 💡 实际应用场景

- 网站公告栏滚动
- 商品标签展示
- 新闻资讯轮播
- 用户评价滚动展示
- 股票行情实时滚动

## 📝 总结

这个横向滚动组件具有以下优势：

- ✅ **功能完整**：支持多种配置选项
- ✅ **性能优秀**：使用现代 Web API 实现
- ✅ **易于使用**：简单的 Props 配置
- ✅ **响应式设计**：自动适配各种场景
- ✅ **代码清晰**：TypeScript + Vue 3 Composition API

希望这个组件能帮助到需要实现横向滚动效果的朋友们。如果你有更好的想法或建议，欢迎在评论区交流讨论！

---

*[相关标签]* #Vue3 #TypeScript #前端组件 #滚动效果 #跑马灯 #无缝滚动