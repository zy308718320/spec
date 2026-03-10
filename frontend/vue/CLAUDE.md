# CLAUDE.md - 前端开发系统规则 (Vue)

> **技术栈**: Bun + Vite + Vue 3 + TypeScript + Tailwind CSS + Pinia + VueUse + Ant Design Vue

---

## 🚨 强制要求

```
🔴 Vue 强制使用 v3 版本，禁止使用 v2 或以下版本
🔴 Tailwind CSS 强制使用 v4 版本，禁止使用 v3 或以下版本
🔴 Composition API + `<script setup>` 为默认组件写法
🔴 严禁使用 CommonJS 模块系统，必须使用 ESM
🔴 尽可能使用 TypeScript，仅在构建工具完全不支持时才用 JavaScript
🔴 数据结构必须定义为强类型，使用 any 或未结构化 JSON 前需征求用户同意
🔴 UI 组件库优先使用 Ant Design Vue，避免引入其他重量级组件库
🔴 禁止在代码中直接内联 SVG 图标，统一使用 Ant Design Vue Icons (@ant-design/icons-vue) 或开源图标库
```

---

## 🎯 常用命令

```bash
bun install          # 安装依赖
bun dev              # 启动开发服务器
bun run build        # 生产构建
bun run check        # Biome检查 + TypeScript检查 + 格式化
bun test             # 运行测试
```

---

## 📁 项目结构

```
src/
├── components/          # 通用组件
│   ├── common/         # 业务通用组件
│   └── layout/         # 布局组件
├── composables/         # 组合式函数（Composables）
├── directives/          # 自定义指令
├── features/            # 功能模块（按业务划分）
│   └── auth/
│       ├── components/
│       ├── composables/
│       ├── stores/
│       └── types/
├── pages/               # 页面组件
├── router/              # 路由配置
├── services/            # API 服务
├── stores/              # 全局 Pinia Store
├── types/               # 全局类型定义
├── utils/               # 工具函数
└── constants/           # 常量定义
```

---

## ⚡ Vue 3 组件规范

### 组件模板（使用 `<script setup>`）

```vue
<script setup lang="ts">
import { computed, ref } from 'vue';
import { useRouter } from 'vue-router';
import { cn } from '@/utils/cn';

interface IButtonProps {
  variant?: 'primary' | 'secondary';
  isLoading?: boolean;
}

const props = withDefaults(defineProps<IButtonProps>(), {
  variant: 'primary',
  isLoading: false,
});

const emit = defineEmits<{
  click: [];
}>();

const router = useRouter();
const classes = computed(() =>
  cn(
    'px-4 py-2 rounded-md font-medium transition-colors',
    props.variant === 'primary' && 'bg-blue-600 text-white hover:bg-blue-700',
    props.variant === 'secondary' && 'bg-gray-200 hover:bg-gray-300'
  )
);

const handleClick = () => {
  emit('click');
};
</script>

<template>
  <button
    :class="classes"
    :disabled="isLoading"
    @click="handleClick"
  >
    <span v-if="isLoading">加载中...</span>
    <slot v-else />
  </button>
</template>
```

### 核心规则

```
✅ 使用 `<script setup lang="ts">` 语法
✅ Props 使用 TypeScript interface + withDefaults
✅ Emits 使用 TypeScript 类型定义
✅ 优先使用 Composition API
✅ 合理使用 ref、reactive、computed
✅ 组件命名采用多单词（PascalCase）
✅ 事件处理函数命名：handleXxx
✅ 使用 v-bind 缩写语法（:class、:id 等）
✅ 使用 v-on 缩写语法（@click、@input 等）
```

---

## 🎨 Tailwind CSS 规范

### cn 工具函数

```typescript
// utils/cn.ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

### 类名顺序

```
布局 → 尺寸 → 间距 → 背景 → 边框 → 文字 → 效果 → 响应式 → 状态

示例：
"flex items-center w-full px-4 bg-white border rounded-lg text-sm shadow-sm md:w-auto hover:bg-gray-50"
```

### 在 Vue 中使用

```vue
<template>
  <div :class="cn('base-class', isActive && 'active-class', props.className)">
    Content
  </div>
</template>
```

---

## 🐻 Pinia 状态管理

### Store 模板

```typescript
// stores/useUserStore.ts
import { defineStore } from 'pinia';
import { ref, computed } from 'vue';

export interface IUser {
  id: string;
  name: string;
  email: string;
}

export const useUserStore = defineStore('user', () => {
  // State
  const user = ref<IUser | null>(null);
  const token = ref<string>('');

  // Getters
  const isAuthenticated = computed(() => !!user.value);
  const userName = computed(() => user.value?.name ?? '');

  // Actions
  const setUser = (newUser: IUser) => {
    user.value = newUser;
  };

  const logout = () => {
    user.value = null;
    token.value = '';
  };

  return {
    user,
    token,
    isAuthenticated,
    userName,
    setUser,
    logout,
  };
});
```

### 使用规则

```vue
<script setup lang="ts">
import { useUserStore } from '@/stores/useUserStore';
import { storeToRefs } from 'pinia';

const userStore = useUserStore();

// ✅ 使用 storeToRefs 保持响应式
const { user, isAuthenticated } = storeToRefs(userStore);

// ✅ Actions 可以直接解构
const { logout } = userStore;

// ❌ 避免直接解构整个 store（失去响应式）
// const { user } = useUserStore();
</script>
```

---

## 🌐 API 服务层

### HTTP 客户端

```typescript
// services/http.ts
import { useUserStore } from '@/stores/useUserStore';

class HttpClient {
  private baseUrl = import.meta.env.VITE_API_BASE_URL;

  private async request<T>(
    endpoint: string,
    config: RequestInit = {}
  ): Promise<T> {
    const userStore = useUserStore();
    const token = userStore.token;

    const response = await fetch(`${this.baseUrl}${endpoint}`, {
      ...config,
      headers: {
        'Content-Type': 'application/json',
        ...(token && { Authorization: `Bearer ${token}` }),
        ...config.headers,
      },
    });

    if (!response.ok) throw new Error('Request failed');
    return response.json();
  }

  get<T>(endpoint: string) {
    return this.request<T>(endpoint, { method: 'GET' });
  }

  post<T>(endpoint: string, data?: unknown) {
    return this.request<T>(endpoint, {
      method: 'POST',
      body: JSON.stringify(data),
    });
  }

  put<T>(endpoint: string, data?: unknown) {
    return this.request<T>(endpoint, {
      method: 'PUT',
      body: JSON.stringify(data),
    });
  }

  delete<T>(endpoint: string) {
    return this.request<T>(endpoint, { method: 'DELETE' });
  }
}

export const http = new HttpClient();
```

### API 服务模块

```typescript
// services/api/user.ts
import type { IUser, IUpdateUserDTO } from '@/types/user';
import { http } from '../http';

export const userService = {
  getProfile: () => http.get<IUser>('/user/profile'),
  updateProfile: (data: IUpdateUserDTO) =>
    http.put<IUser>('/user/profile', data),
};
```

---

## 🎣 Composables（组合式函数）

### 常用 Composables

```typescript
// composables/useDebounce.ts
import { ref, watch } from 'vue';

export function useDebounce<T>(value: Ref<T>, delay: number) {
  const debouncedValue = ref(value.value) as Ref<T>;

  let timeout: ReturnType<typeof setTimeout> | null = null;

  watch(value, (newValue) => {
    if (timeout) clearTimeout(timeout);
    timeout = setTimeout(() => {
      debouncedValue.value = newValue;
    }, delay);
  });

  return debouncedValue;
}

// composables/useClickOutside.ts
import { onMounted, onUnmounted, ref } from 'vue';

export function useClickOutside<T extends HTMLElement>(
  callback: () => void
) {
  const target = ref<T | null>(null);

  const handleClick = (event: MouseEvent) => {
    if (!target.value?.contains(event.target as Node)) {
      callback();
    }
  };

  onMounted(() => {
    document.addEventListener('mousedown', handleClick);
  });

  onUnmounted(() => {
    document.removeEventListener('mousedown', handleClick);
  });

  return { target };
}
```

### VueUse 推荐使用

```typescript
import {
  useLocalStorage,
  useMouse,
  usePreferredDark,
  useWindowSize,
} from '@vueuse/core';

// 本地存储
const stored = useLocalStorage('key', defaultValue);

// 响应式窗口大小
const { width, height } = useWindowSize();

// 首选暗黑模式
const isDark = usePreferredDark();
```

---

## 📝 TypeScript 规范

### 强类型要求

```
✅ 所有数据结构必须定义为强类型
✅ 优先使用 interface 或 type 明确定义结构
✅ API 响应数据必须有对应的类型定义
⚠️ 使用 any 或未结构化 JSON 前，必须先征求用户同意
❌ 禁止随意使用 any 绕过类型检查
```

### 类型定义

```typescript
// types/user.ts

// 实体类型（interface 以 I 开头）
export interface IUser {
  id: string;
  name: string;
  email: string;
  role: TUserRole;
}

// 枚举用 const 对象
export const UserRole = {
  Admin: 'admin',
  User: 'user',
} as const;

export type TUserRole = (typeof UserRole)[keyof typeof UserRole];

// DTO 类型（interface 以 I 开头）
export interface ICreateUserDTO {
  name: string;
  email: string;
}

// 组件 Props（interface 以 I 开头）
export interface IUserCardProps {
  user: IUser;
  onEdit?: (user: IUser) => void;
  class?: string;
}
```

### Vue 组件类型

```typescript
// Props 类型
interface IProps {
  title: string;
  count?: number;
}

// Emits 类型
interface IEmits {
  change: [value: string];
  delete: [id: string];
}

// 在组件中使用
const props = defineProps<IProps>();
const emit = defineEmits<IEmits>();
```

---

## 🚀 Vue Router 配置

### 路由配置

```typescript
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router';
import type { RouteRecordRaw } from 'vue-router';

const routes: RouteRecordRaw[] = [
  {
    path: '/',
    name: 'Home',
    component: () => import('@/pages/HomePage.vue'),
  },
  {
    path: '/about',
    name: 'About',
    component: () => import('@/pages/AboutPage.vue'),
    meta: { requiresAuth: true },
  },
];

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
});

// 路由守卫
router.beforeEach((to, from, next) => {
  const userStore = useUserStore();
  if (to.meta.requiresAuth && !userStore.isAuthenticated) {
    next('/login');
  } else {
    next();
  }
});

export default router;
```

### 在组件中使用路由

```vue
<script setup lang="ts">
import { useRouter, useRoute } from 'vue-router';

const router = useRouter();
const route = useRoute();

// 编程式导航
const navigateToAbout = () => {
  router.push({ name: 'About', params: { id: 123 } });
};

// 获取路由参数
const userId = route.params.id as string;
</script>
```

---

## ⚙️ 配置文件

### biome.json

```json
{
  "vcs": {
    "enabled": true,
    "clientKind": "git",
    "useIgnoreFile": true
  },
  "files": {
    "ignoreUnknown": false
  },
  "formatter": {
    "enabled": true,
    "formatWithErrors": false,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineEnding": "lf",
    "lineWidth": 140
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "correctness": {
        "noUnusedVariables": "error",
        "noUnusedImports": "error",
        "useExhaustiveDependencies": "warn",
        "noUnknownFunction": "off",
        "noUndeclaredVariables": "error",
        "noInvalidUseBeforeDeclaration": "error"
      },
      "suspicious": {
        "noExplicitAny": "warn",
        "noConsole": "off",
        "noArrayIndexKey": "off",
        "noConfusingVoidType": "warn",
        "noEmptyBlock": "warn",
        "noMisleadingCharacterClass": "error"
      },
      "style": {
        "noNonNullAssertion": "off",
        "useImportType": "warn",
        "noParameterAssign": "warn"
      },
      "a11y": {
        "noSvgWithoutTitle": "off",
        "useButtonType": "off",
        "useKeyWithClickEvents": "off",
        "noStaticElementInteractions": "off",
        "useValidAnchor": "warn"
      },
      "complexity": {
        "noForEach": "off"
      },
      "nursery": {
        "useSortedClasses": "error"
      }
    }
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "single",
      "jsxQuoteStyle": "double",
      "quoteProperties": "asNeeded",
      "trailingCommas": "all",
      "semicolons": "always",
      "arrowParentheses": "asNeeded",
      "bracketSpacing": true,
      "bracketSameLine": false
    }
  },
  "css": {
    "parser": {
      "cssModules": false,
      "tailwindDirectives": true
    }
  },
  "json": {
    "formatter": {
      "enabled": true,
      "indentStyle": "space",
      "indentWidth": 2
    }
  },
  "vue": {
    "formatter": {
      "quoteStyle": "double"
    }
  }
}
```

### vite.config.ts

```typescript
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import path from 'node:path';

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});
```

### tsconfig.json (paths)

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    },
    "types": ["vite/client"]
  }
}
```

---

## 🧪 测试规范

### 组件测试（使用 Vitest + Vue Test Utils）

```typescript
import { describe, it, expect, vi } from 'vitest';
import { mount } from '@vue/test-utils';
import Button from '@/components/Button.vue';

describe('Button', () => {
  it('should emit click event when clicked', async () => {
    const wrapper = mount(Button, {
      props: {
        variant: 'primary',
      },
      slots: {
        default: 'Click me',
      },
    });

    await wrapper.find('button').trigger('click');
    expect(wrapper.emitted('click')).toHaveLength(1);
  });

  it('should be disabled when loading', () => {
    const wrapper = mount(Button, {
      props: {
        isLoading: true,
      },
    });

    const button = wrapper.find('button');
    expect(button.attributes('disabled')).toBeDefined();
  });
});
```

### Store 测试

```typescript
import { setActivePinia, createPinia } from 'pinia';
import { describe, beforeEach, it, expect } from 'vitest';
import { useUserStore } from '@/stores/useUserStore';

describe('useUserStore', () => {
  beforeEach(() => {
    setActivePinia(createPinia());
  });

  it('should set user', () => {
    const store = useUserStore();
    store.setUser({ id: '1', name: 'John', email: 'john@example.com' });
    expect(store.isAuthenticated).toBe(true);
  });

  it('should logout', () => {
    const store = useUserStore();
    store.setUser({ id: '1', name: 'John', email: 'john@example.com' });
    store.logout();
    expect(store.user).toBeNull();
    expect(store.isAuthenticated).toBe(false);
  });
});
```

---

## 🚀 性能优化

```vue
<script setup lang="ts">
import { computed, shallowRef, defineAsyncComponent } from 'vue';

// ✅ 使用 computed 缓存计算
const fullName = computed(() => `${firstName.value} ${lastName.value}`);

// ✅ 路由懒加载
const HeavyComponent = defineAsyncComponent(() =>
  import('@/components/HeavyComponent.vue')
);

// ✅ 使用 shallowRef 优化大型对象
const largeData = shallowRef<LargeObject>({ ... });

// ✅ v-once 渲染静态内容
// <div v-once>静态内容</div>

// ✅ v-memo 优化列表渲染
// <div v-for="item in list" :key="item.id" v-memo="[item.id]">
</script>

<!-- ✅ 使用 key 优化列表 -->
<template>
  <div v-for="item in items" :key="item.id">
    {{ item.name }}
  </div>
</template>
```

---

## 🔒 安全规范

```
🔴 不硬编码敏感信息，使用环境变量 VITE_XXX
🔴 用户输入必须验证（推荐 Zod）
🔴 v-html 必须用 DOMPurify 消毒
🔴 外部链接添加 rel="noopener noreferrer"
🔴 敏感数据不存 localStorage，用 httpOnly cookie
```

---

## 📋 提交前检查

```
□ bun run check 通过
□ bun run build 成功
□ bun test 通过
□ 无 console.log 遗留
□ 组件有完整 TypeScript 类型
□ 新功能有对应测试
```

---

## 🎯 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| 组件文件 | PascalCase（多单词） | `UserCard.vue` |
| Composable 文件 | camelCase + use 前缀 | `useAuth.ts` |
| 工具函数 | camelCase | `formatDate.ts` |
| 类型文件 | camelCase | `types.ts` |
| 常量 | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| CSS 类 | kebab-case | `user-card` |

### Vue 特殊命名规则

```
✅ 组件名必须是多单词（避免与 HTML 标签冲突）
✅ 组件名使用 PascalCase
✅ Props 使用 camelCase
✅ Events 使用 kebab-case
✅ v-model 修饰符命名清晰
```

---

## 🎨 当用户提供设计稿时

### 1. 分析设计
- 识别布局结构（Grid/Flex）
- 提取颜色值（转换为 CSS 变量或 Tailwind 主题配置）
- 识别字体和间距规律（映射到 Tailwind 类）
- 标注响应式断点（sm/md/lg/xl/2xl）

### 2. 生成代码
- 使用语义化 HTML
- CSS 采用 Tailwind（遵循本文档 Tailwind CSS 规范）
- 按本文档组件规范进行拆分
- 添加必要注释

### 3. 质量检查
- 像素级对比设计稿
- 响应式测试（各断点）
- 可访问性检查（a11y）
- 使用 chrome-devtools-mcp 进行视觉回归测试，必要时再次对比设计稿

---

## 🎯 Ant Design Vue 使用规范

### 组件引入

```typescript
// main.ts - 全局引入（大型项目）
import Antd from 'ant-design-vue';
import 'ant-design-vue/dist/reset.css';
app.use(Antd);

// 按需引入（推荐）
import { Button, Input } from 'ant-design-vue';
```

### 类型使用

```vue
<script setup lang="ts">
import type { Rule } from 'ant-design-vue/es/form';
import { message } from 'ant-design-vue';

const rules: Record<string, Rule[]> = {
  name: [
    { required: true, message: '请输入名称', trigger: 'blur' },
    { min: 3, max: 5, message: '长度在 3 到 5 个字符', trigger: 'blur' },
  ],
};

const submitForm = async () => {
  try {
    await formRef.value.validate();
    message.success('提交成功！');
  } catch (error) {
    message.error('表单验证失败');
  }
};
</script>
```

---

## 🎯 Vue 3 最佳实践

### 响应式数据选择

```typescript
// ✅ 基本类型使用 ref
const count = ref(0);
const message = ref('hello');

// ✅ 对象使用 reactive
const state = reactive({
  count: 0,
  message: 'hello',
});

// ✅ 解构 reactive 使用 toRefs
const { count, message } = toRefs(state);

// ⚠️ 避免 ref 嵌套
// const obj = ref({ count: 0 }) // ❌
// const obj = reactive({ count: 0 }) // ✅
```

### 生命周期

```vue
<script setup lang="ts">
import { onMounted, onUnmounted, onUpdated } from 'vue';

// 组件挂载后
onMounted(() => {
  console.log('mounted');
});

// 组件卸载前
onUnmounted(() => {
  console.log('unmounted');
});

// 组件更新后
onUpdated(() => {
  console.log('updated');
});
</script>
```

### Watch 和 WatchEffect

```typescript
// watch - 需要明确监听源
watch(source, (newValue, oldValue) => {
  console.log(`changed from ${oldValue} to ${newValue}`);
});

// watchEffect - 自动追踪依赖
watchEffect(() => {
  console.log(`count is: ${count.value}`);
});

// watch 监听多个源
watch([count, message], ([newCount, newMessage]) => {
  console.log(`count: ${newCount}, message: ${newMessage}`);
});
```

---
