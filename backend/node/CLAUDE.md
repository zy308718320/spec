# CLAUDE.md - 后端开发系统规则

> **技术栈**: Bun + Hono + Drizzle ORM + Zod + Biome

---

## 🎯 常用命令

```bash
bun dev                  # 启动开发服务器（热重载）
bun run check            # Biome 检查 + 格式化
bun test                 # 运行测试
bun run db:generate      # 生成数据库迁移
bun run db:migrate       # 执行迁移
bun run db:studio        # Drizzle Studio
```

---

## 📁 项目结构

```
src/
├── index.ts              # 入口
├── app.ts                # Hono 应用实例
├── config/               # 配置（env.ts, database.ts）
├── routes/               # 路由模块
├── controllers/          # 控制器（请求处理）
├── services/             # 业务逻辑层
├── repositories/         # 数据访问层
├── db/
│   ├── index.ts         # 数据库连接
│   ├── schema/          # Drizzle Schema
│   └── migrations/
├── middlewares/          # 中间件
├── validators/           # Zod Schema
├── types/                # 类型定义
├── utils/                # 工具函数
└── errors/               # 自定义错误
```

---

## 🔥 Hono 核心模式

### 应用实例
```typescript
// src/app.ts
import { Hono } from 'hono'
import { cors } from 'hono/cors'
import { logger } from 'hono/logger'
import { secureHeaders } from 'hono/secure-headers'

const app = new Hono<AppEnv>()

app.use('*', logger())
app.use('*', secureHeaders())
app.use('*', cors({ origin: ['http://localhost:3000'], credentials: true }))

app.route('/api', routes)
app.get('/health', (c) => c.json({ status: 'ok' }))
app.onError(errorHandler)
app.notFound((c) => c.json({ error: 'Not Found' }, 404))

export { app }
```

### 路由模块
```typescript
// src/routes/users.ts
import { Hono } from 'hono'
import { zValidator } from '@hono/zod-validator'

const users = new Hono<AppEnv>()
  .use('*', authMiddleware)
  .get('/', zValidator('query', querySchema), UserController.list)
  .get('/:id', UserController.getById)
  .post('/', zValidator('json', createUserSchema), UserController.create)
  .patch('/:id', zValidator('json', updateUserSchema), UserController.update)
  .delete('/:id', UserController.delete)

export { users }
```

---

## 🗄️ Drizzle ORM 核心模式

### Schema 定义
```typescript
// src/db/schema/users.ts
import { sqliteTable, text, integer } from 'drizzle-orm/sqlite-core'
import { createId } from '@paralleldrive/cuid2'

export const users = sqliteTable('users', {
  id: text('id').primaryKey().$defaultFn(() => createId()),
  email: text('email').notNull().unique(),
  name: text('name').notNull(),
  password: text('password').notNull(),
  role: text('role', { enum: ['admin', 'user'] }).default('user').notNull(),
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
  updatedAt: integer('updated_at', { mode: 'timestamp' }).$onUpdate(() => new Date()),
})

export type User = typeof users.$inferSelect
export type NewUser = typeof users.$inferInsert
```

### 数据库连接
```typescript
// src/db/index.ts
import { drizzle } from 'drizzle-orm/bun-sqlite'
import { Database } from 'bun:sqlite'
import * as schema from './schema'

const sqlite = new Database(process.env.DATABASE_URL || 'sqlite.db')
export const db = drizzle(sqlite, { schema })
```

---

## ✅ Zod 验证模式

```typescript
// src/validators/user.ts
import { z } from 'zod'

export const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(2).max(50),
  password: z.string().min(8).regex(/[A-Z]/, '需包含大写字母'),
})

export const updateUserSchema = createUserSchema.partial().omit({ password: true })

export const querySchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  search: z.string().optional(),
})

export type CreateUserInput = z.infer<typeof createUserSchema>
```

### 环境变量验证
```typescript
// src/config/env.ts
import { z } from 'zod'

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().min(1),
  JWT_SECRET: z.string().min(32),
})

export const env = envSchema.parse(process.env)
```

---

## 🏗️ 分层架构

### Controller → Service → Repository

```typescript
// Controller: 处理请求/响应
static async create(c: Context<AppEnv>) {
  const input = c.req.valid('json') as CreateUserInput
  const user = await UserService.create(input)
  return c.json({ data: user }, 201)
}

// Service: 业务逻辑
static async create(input: CreateUserInput) {
  const exists = await UserRepository.findByEmail(input.email)
  if (exists) throw new AppError('EMAIL_EXISTS', '邮箱已被注册', 409)
  const hashedPassword = await Bun.password.hash(input.password)
  return UserRepository.create({ ...input, password: hashedPassword })
}

// Repository: 数据访问
static async create(data: NewUser) {
  const [user] = await db.insert(users).values(data).returning()
  return user
}
```

---

## 🛡️ 中间件

### 认证中间件
```typescript
// src/middlewares/auth.ts
import { createMiddleware } from 'hono/factory'
import { verify } from 'hono/jwt'

export const authMiddleware = createMiddleware<AppEnv>(async (c, next) => {
  const token = c.req.header('Authorization')?.slice(7)
  if (!token) throw new AppError('UNAUTHORIZED', '未提供认证令牌', 401)

  try {
    const payload = await verify(token, env.JWT_SECRET)
    c.set('user', payload)
    await next()
  } catch {
    throw new AppError('INVALID_TOKEN', '无效的认证令牌', 401)
  }
})
```

### 错误处理
```typescript
// src/middlewares/errorHandler.ts
export const errorHandler: ErrorHandler = (err, c) => {
  if (err instanceof ZodError) {
    return c.json({ error: 'VALIDATION_ERROR', details: err.errors }, 400)
  }
  if (err instanceof AppError) {
    return c.json({ error: err.code, message: err.message }, err.status)
  }
  return c.json({ error: 'INTERNAL_ERROR', message: '服务器内部错误' }, 500)
}

// src/errors/AppError.ts
export class AppError extends Error {
  constructor(public code: string, message: string, public status = 400) {
    super(message)
  }
}
```

---

## 📝 类型定义

```typescript
// src/types/env.ts
export type AppEnv = {
  Variables: {
    user: { id: string; role: string }
  }
}

// src/types/response.ts
export interface ApiResponse<T> {
  data: T
  meta?: { total: number; page: number; limit: number; totalPages: number }
}
```

---

## ⚙️ 配置文件

### biome.json
```json
{
  "linter": {
    "rules": {
      "recommended": true,
      "correctness": { "noUnusedVariables": "error", "noUnusedImports": "error" },
      "suspicious": { "noExplicitAny": "warn", "noConsoleLog": "warn" }
    }
  },
  "formatter": { "indentStyle": "space", "indentWidth": 2 },
  "javascript": { "formatter": { "quoteStyle": "single", "semicolons": "asNeeded" } }
}
```

### drizzle.config.ts
```typescript
import { defineConfig } from 'drizzle-kit'

export default defineConfig({
  schema: './src/db/schema/index.ts',
  out: './src/db/migrations',
  dialect: 'sqlite',
  dbCredentials: { url: process.env.DATABASE_URL || 'sqlite.db' },
})
```

---

## 🔒 安全规范

```
🔴 密码使用 Bun.password.hash() 加密
🔴 JWT 密钥至少 32 位，存环境变量
🔴 所有输入必须 Zod 验证
🔴 使用 Drizzle ORM 防 SQL 注入
🔴 敏感信息不记录日志
🔴 配置 CORS 白名单
```

---

## 📡 API 响应格式

```typescript
// 成功
{ "data": { ... } }
{ "data": [...], "meta": { "total": 100, "page": 1, "limit": 20, "totalPages": 5 } }

// 错误
{ "error": "ERROR_CODE", "message": "错误描述" }

// 状态码
200 OK / 201 Created / 204 No Content
400 Bad Request / 401 Unauthorized / 403 Forbidden / 404 Not Found / 409 Conflict
500 Internal Error
```

---

## 🎯 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| 路由文件 | 复数小写 | `users.ts` |
| Controller/Service/Repository | PascalCase + 后缀 | `UserService` |
| Schema 表名 | 复数小写 | `users` |
| 验证 Schema | camelCase + Schema | `createUserSchema` |
| 中间件 | camelCase + Middleware | `authMiddleware` |
| 错误码 | UPPER_SNAKE_CASE | `USER_NOT_FOUND` |

---

## 📋 提交前检查

```
□ bun run check 通过
□ bun test 通过
□ 所有接口有 Zod 验证
□ 错误处理完整
□ 敏感操作有权限验证
□ 无 console.log 遗留
```

## 🧪 测试规范

### 单元测试
```typescript
// src/services/user.test.ts
import { describe, it, expect, mock, beforeEach } from 'bun:test'

describe('UserService', () => {
  beforeEach(() => {
    // 重置 mock
  })

  it('should create user with hashed password', async () => {
    const user = await UserService.create({ email: 'test@example.com', name: 'Test', password: 'Password123' })
    expect(user.email).toBe('test@example.com')
  })

  it('should throw error when email exists', async () => {
    expect(UserService.create({ email: 'exists@example.com', ... })).rejects.toThrow('邮箱已被注册')
  })
})
```

### API 集成测试
```typescript
// src/routes/users.test.ts
import { describe, it, expect } from 'bun:test'
import { app } from '@/app'

describe('GET /api/users', () => {
  it('should return 401 without token', async () => {
    const res = await app.request('/api/users')
    expect(res.status).toBe(401)
  })

  it('should return users list', async () => {
    const res = await app.request('/api/users', {
      headers: { Authorization: `Bearer ${validToken}` },
    })
    expect(res.status).toBe(200)
    const json = await res.json()
    expect(json.data).toBeArray()
  })
})
```

---

## 🔐 认证模式

### JWT 生成与验证
```typescript
// src/services/auth.ts
import { sign, verify } from 'hono/jwt'
import { env } from '@/config/env'

export class AuthService {
  static async login(email: string, password: string) {
    const user = await UserRepository.findByEmail(email)
    if (!user) throw new AppError('INVALID_CREDENTIALS', '邮箱或密码错误', 401)

    const valid = await Bun.password.verify(password, user.password)
    if (!valid) throw new AppError('INVALID_CREDENTIALS', '邮箱或密码错误', 401)

    const token = await sign(
      { id: user.id, role: user.role, exp: Math.floor(Date.now() / 1000) + 60 * 60 * 24 * 7 },
      env.JWT_SECRET
    )
    return { token, user: { id: user.id, email: user.email, name: user.name } }
  }
}
```

### 角色权限中间件
```typescript
// src/middlewares/auth.ts
export const requireRole = (...roles: string[]) =>
  createMiddleware<AppEnv>(async (c, next) => {
    const user = c.get('user')
    if (!roles.includes(user.role)) {
      throw new AppError('FORBIDDEN', '权限不足', 403)
    }
    await next()
  })

// 使用
users.delete('/:id', requireRole('admin'), UserController.delete)
```

---

## 📊 Drizzle 查询模式

### 常用查询
```typescript
// 分页查询
const data = await db.select().from(users)
  .where(like(users.name, `%${search}%`))
  .orderBy(desc(users.createdAt))
  .limit(limit)
  .offset((page - 1) * limit)

// 关联查询
const user = await db.query.users.findFirst({
  where: eq(users.id, id),
  with: { posts: true },
})

// 聚合查询
const [{ total }] = await db.select({ total: count() }).from(users)

// 事务
await db.transaction(async (tx) => {
  await tx.insert(users).values(userData)
  await tx.insert(profiles).values(profileData)
})
```

### 关系定义
```typescript
// src/db/schema/relations.ts
import { relations } from 'drizzle-orm'

export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}))

export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, { fields: [posts.authorId], references: [users.id] }),
}))
```

---

## 🛠️ 工具函数

### 分页响应
```typescript
// src/utils/pagination.ts
export function paginate<T>(data: T[], total: number, page: number, limit: number) {
  return {
    data,
    meta: {
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit),
      hasNext: page * limit < total,
      hasPrev: page > 1,
    },
  }
}
```

### 响应助手
```typescript
// src/utils/response.ts
export const success = <T>(c: Context, data: T, status = 200) => c.json({ data }, status)
export const created = <T>(c: Context, data: T) => c.json({ data }, 201)
export const noContent = (c: Context) => c.body(null, 204)
```

---

## 📦 package.json

```json
{
  "scripts": {
    "dev": "bun run --hot src/index.ts",
    "start": "NODE_ENV=production bun src/index.ts",
    "check": "biome check --apply ./src",
    "test": "bun test",
    "db:generate": "drizzle-kit generate",
    "db:migrate": "drizzle-kit migrate",
    "db:studio": "drizzle-kit studio"
  },
  "dependencies": {
    "hono": "^4.0.0",
    "@hono/zod-validator": "^0.2.0",
    "drizzle-orm": "^0.30.0",
    "zod": "^3.22.0",
    "@paralleldrive/cuid2": "^2.2.0"
  },
  "devDependencies": {
    "@biomejs/biome": "^1.5.0",
    "drizzle-kit": "^0.20.0",
    "@types/bun": "latest"
  }
}
```

---

## 🌐 环境变量

```bash
# .env.example
NODE_ENV=development
PORT=3000
DATABASE_URL=sqlite.db
JWT_SECRET=your-super-secret-key-at-least-32-chars
```

---

## 📂 文件模板

### 新建路由模块
```typescript
// src/routes/[resource].ts
import { Hono } from 'hono'
import { zValidator } from '@hono/zod-validator'
import { createSchema, updateSchema, querySchema } from '@/validators/[resource]'
import { Controller } from '@/controllers/[resource]'
import { authMiddleware } from '@/middlewares/auth'

const resource = new Hono<AppEnv>()
  .use('*', authMiddleware)
  .get('/', zValidator('query', querySchema), Controller.list)
  .get('/:id', Controller.getById)
  .post('/', zValidator('json', createSchema), Controller.create)
  .patch('/:id', zValidator('json', updateSchema), Controller.update)
  .delete('/:id', Controller.delete)

export { resource }
```

### 新建 Schema
```typescript
// src/db/schema/[resource].ts
import { sqliteTable, text, integer } from 'drizzle-orm/sqlite-core'
import { createId } from '@paralleldrive/cuid2'

export const resources = sqliteTable('resources', {
  id: text('id').primaryKey().$defaultFn(() => createId()),
  // ... 字段定义
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
  updatedAt: integer('updated_at', { mode: 'timestamp' }).$onUpdate(() => new Date()),
})

export type Resource = typeof resources.$inferSelect
export type NewResource = typeof resources.$inferInsert
```

---

## ⚡ 性能优化

```
✅ 使用数据库索引优化查询
✅ 分页查询避免全表扫描
✅ 使用 Promise.all 并行查询
✅ 合理使用 select 只查询需要的字段
✅ 大量写入使用批量插入
✅ 复杂查询考虑缓存
```

```typescript
// 批量插入
await db.insert(users).values(usersArray)

// 只查询需要的字段
await db.select({ id: users.id, name: users.name }).from(users)

// 并行查询
const [data, total] = await Promise.all([
  db.select().from(users).limit(20),
  db.select({ count: count() }).from(users),
])
```

---

## 🚨 常见错误处理

| 场景 | 错误码 | 状态码 |
|------|--------|--------|
| 资源不存在 | `NOT_FOUND` | 404 |
| 邮箱已注册 | `EMAIL_EXISTS` | 409 |
| 验证失败 | `VALIDATION_ERROR` | 400 |
| 未认证 | `UNAUTHORIZED` | 401 |
| 无权限 | `FORBIDDEN` | 403 |
| 密码错误 | `INVALID_CREDENTIALS` | 401 |
| 令牌过期 | `TOKEN_EXPIRED` | 401 |

---

