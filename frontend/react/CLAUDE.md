# CLAUDE.md - 前端开发系统规则

> **技术栈**: Bun + Vite + React + Tailwind CSS + Zustand + Biome + shadcn/ui

---

## 🚨 强制要求

```
🔴 React 强制使用 v19 版本，禁止使用 v18 或以下版本
🔴 Tailwind CSS 强制使用 v4 版本，禁止使用 v3 或以下版本
🔴 Biome 强制使用 v2 及以上版本
🔴 严禁使用 CommonJS 模块系统，必须使用 ESM
🔴 尽可能使用 TypeScript，仅在构建工具完全不支持时才用 JavaScript
🔴 数据结构必须定义为强类型，使用 any 或未结构化 JSON 前需征求用户同意
🔴 UI 组件库优先使用 shadcn/ui，避免引入其他重量级组件库
🔴 禁止在代码中直接内联 SVG 图标，统一使用开源 Font 图标库（如 Lucide、Phosphor Icons）
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
│   ├── ui/             # 基础 UI 组件（Button/Input/Modal）
│   └── common/         # 业务通用组件
├── features/           # 功能模块（按业务划分）
│   └── auth/
│       ├── components/
│       ├── hooks/
│       ├── stores/
│       └── types/
├── hooks/              # 全局自定义 Hooks
├── layouts/            # 布局组件
├── pages/              # 页面组件
├── services/           # API 服务
├── stores/             # 全局 Zustand Store
├── types/              # 全局类型定义
├── utils/              # 工具函数
└── constants/          # 常量定义
```

---

## ⚛️ React 组件规范

### 组件模板
```tsx
import { forwardRef } from 'react';
import { cn } from '@/utils/cn';

interface IButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary';
  isLoading?: boolean;
}

export const Button = forwardRef<HTMLButtonElement, IButtonProps>(
  ({ className, variant = 'primary', isLoading, children, ...props }, ref) => {
    return (
      <button
        ref={ref}
        disabled={isLoading}
        className={cn(
          'px-4 py-2 rounded-md font-medium transition-colors',
          variant === 'primary' && 'bg-blue-600 text-white hover:bg-blue-700',
          variant === 'secondary' && 'bg-gray-200 hover:bg-gray-300',
          className
        )}
        {...props}
      >
        {isLoading ? <Spinner /> : children}
      </button>
    );
  }
);
```

### 核心规则
```
✅ 使用函数组件 + TypeScript
✅ Props 接口继承原生 HTML 属性
✅ 使用 forwardRef 暴露 ref
✅ 使用 cn() 合并类名，支持外部覆盖
✅ Early return 处理 loading/error/empty 状态
✅ 事件处理函数命名：handleXxx
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

### 常用模式
```tsx
// 居中容器
"mx-auto max-w-7xl px-4 sm:px-6 lg:px-8"

// 卡片
"rounded-lg border bg-white p-6 shadow-sm"

// 响应式隐藏
"hidden md:block"  // 移动端隐藏
"block md:hidden"  // 桌面端隐藏
```

---

## 🐻 Zustand 状态管理

### Store 模板
```typescript
// stores/useUserStore.ts
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

interface IUserState {
  user: IUser | null;
  isAuthenticated: boolean;
  setUser: (user: IUser) => void;
  logout: () => void;
}

export const useUserStore = create<IUserState>()(
  devtools(
    persist(
      immer((set) => ({
        user: null,
        isAuthenticated: false,

        setUser: (user) =>
          set((state) => {
            state.user = user;
            state.isAuthenticated = true;
          }),

        logout: () =>
          set((state) => {
            state.user = null;
            state.isAuthenticated = false;
          }),
      })),
      { name: 'user-storage' }
    )
  )
);
```

### 使用规则
```tsx
// ✅ 使用选择器，避免不必要的重渲染
const userName = useUserStore((state) => state.user?.name);

// ✅ 多个状态使用 shallow
import { shallow } from 'zustand/shallow';
const { user, isLoading } = useUserStore(
  (state) => ({ user: state.user, isLoading: state.isLoading }),
  shallow
);

// ❌ 避免选择整个 store
const store = useUserStore();
```

---

## 🌐 API 服务层

### HTTP 客户端
```typescript
// services/http.ts
class HttpClient {
  private baseUrl = import.meta.env.VITE_API_BASE_URL;

  private async request<T>(endpoint: string, config: RequestInit = {}): Promise<T> {
    const token = useUserStore.getState().token;

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
    return this.request<T>(endpoint, { method: 'POST', body: JSON.stringify(data) });
  }
}

export const http = new HttpClient();
```

### API 服务模块
```typescript
// services/api/user.ts
export const userService = {
  getProfile: () => http.get<IUser>('/user/profile'),
  updateProfile: (data: IUpdateUserDTO) => http.patch<IUser>('/user/profile', data),
};
```

---

## 🔧 自定义 Hooks

### 常用 Hooks
```typescript
// hooks/useDebounce.ts
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// hooks/useClickOutside.ts
export function useClickOutside<T extends HTMLElement>(handler: () => void) {
  const ref = useRef<T>(null);

  useEffect(() => {
    const listener = (e: MouseEvent) => {
      if (!ref.current?.contains(e.target as Node)) handler();
    };
    document.addEventListener('mousedown', listener);
    return () => document.removeEventListener('mousedown', listener);
  }, [handler]);

  return ref;
}
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
interface IUser {
  id: string;
  name: string;
  email: string;
  role: TUserRole;
}

// 枚举用 const 对象
const UserRole = { Admin: 'admin', User: 'user' } as const;
type TUserRole = (typeof UserRole)[keyof typeof UserRole];

// DTO 类型（interface 以 I 开头）
interface ICreateUserDTO {
  name: string;
  email: string;
}

// 组件 Props（interface 以 I 开头）
interface IUserCardProps {
  user: IUser;
  onEdit?: (user: IUser) => void;
  className?: string;
}
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
  }
}
```

### vite.config.ts
```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'node:path';
export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: { '@': path.resolve(__dirname, './src') },
  },
});
```

### tsconfig.json (paths)
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  }
}
```

---

## 🧪 测试规范

### 组件测试
```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

describe('Button', () => {
  it('should call onClick when clicked', async () => {
    const handleClick = vi.fn();
    render(<Button onClick={handleClick}>Click</Button>);
    
    await userEvent.click(screen.getByRole('button'));
    expect(handleClick).toHaveBeenCalledTimes(1);
  });
});
```

### Store 测试
```typescript
describe('useUserStore', () => {
  beforeEach(() => {
    useUserStore.setState({ user: null, isAuthenticated: false });
  });

  it('should set user', () => {
    useUserStore.getState().setUser({ id: '1', name: 'John' });
    expect(useUserStore.getState().isAuthenticated).toBe(true);
  });
});
```

---

## 🚀 性能优化

```tsx
// ✅ React.memo 避免重渲染
export const UserCard = memo(function UserCard({ user }: IProps) {});

// ✅ useMemo 缓存计算
const sorted = useMemo(() => items.sort(...), [items]);

// ✅ useCallback 缓存函数
const handleSubmit = useCallback((data) => {}, []);

// ✅ 路由懒加载
const Dashboard = lazy(() => import('@/pages/Dashboard'));

// ✅ 列表使用稳定 key
{items.map((item) => <Item key={item.id} />)}
```

---

## 🔒 安全规范

```
🔴 不硬编码敏感信息，使用环境变量 VITE_XXX
🔴 用户输入必须验证（推荐 Zod）
🔴 dangerouslySetInnerHTML 必须用 DOMPurify 消毒
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
| 组件文件 | PascalCase | `UserCard.tsx` |
| Hook 文件 | camelCase + use 前缀 | `useAuth.ts` |
| 工具函数 | camelCase | `formatDate.ts` |
| 类型文件 | camelCase | `types.ts` |
| 常量 | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| CSS 类 | kebab-case | `user-card` |

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
