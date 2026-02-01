# CLAUDE.md - Python 后端开发系统规则

> **技术栈**: FastAPI + SQLAlchemy + Pydantic + Alembic + pytest + Ruff

---

## 🎯 常用命令

```bash
uvicorn src.main:app --reload          # 启动开发服务器（热重载）
ruff check src/                        # Ruff 检查
ruff check --fix src/                  # Ruff 自动修复
black src/                              # Black 格式化
pytest                                  # 运行测试
pytest --cov=src                        # 测试覆盖率
alembic revision --autogenerate -m "描述"  # 生成迁移
alembic upgrade head                    # 执行迁移
alembic downgrade -1                    # 回滚迁移
```

---

## 📁 项目结构

```
src/
├── main.py               # 应用入口
├── config/               # 配置（settings.py, database.py）
├── api/                  # API 路由
│   └── v1/
│       └── endpoints/    # 路由端点
├── models/               # SQLAlchemy 模型
├── schemas/              # Pydantic Schema
├── services/             # 业务逻辑层
├── repositories/         # 数据访问层
├── core/                 # 核心功能（security.py, deps.py）
├── db/
│   ├── base.py          # 数据库连接
│   └── migrations/      # Alembic 迁移
├── middleware/           # 中间件
├── utils/                # 工具函数
├── exceptions/           # 自定义异常
└── types/                # 类型定义
tests/
├── unit/                 # 单元测试
├── integration/          # 集成测试
└── conftest.py           # pytest 配置
```

---

## 🔥 FastAPI 核心模式

### 应用实例
```python
# src/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from src.api.v1.api import api_router
from src.core.config import settings
from src.exceptions.handlers import register_exception_handlers

app = FastAPI(
    title=settings.PROJECT_NAME,
    version=settings.VERSION,
    openapi_url=f"{settings.API_V1_STR}/openapi.json",
)

# CORS 中间件
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.BACKEND_CORS_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 注册异常处理器
register_exception_handlers(app)

# 注册路由
app.include_router(api_router, prefix=settings.API_V1_STR)

@app.get("/health")
async def health_check():
    return {"status": "ok"}
```

### 路由模块
```python
# src/api/v1/endpoints/users.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from typing import List
from src.api.deps import get_db, get_current_user
from src.schemas.user import UserCreate, UserUpdate, UserResponse
from src.services.user_service import UserService

router = APIRouter()

@router.get("/", response_model=List[UserResponse])
async def list_users(
    skip: int = 0,
    limit: int = 100,
    db: AsyncSession = Depends(get_db),
    current_user = Depends(get_current_user),
):
    return await UserService.list(db, skip=skip, limit=limit)

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: str,
    db: AsyncSession = Depends(get_db),
):
    user = await UserService.get_by_id(db, user_id)
    if not user:
        raise HTTPException(status_code=404, detail="用户不存在")
    return user

@router.post("/", response_model=UserResponse, status_code=201)
async def create_user(
    user_in: UserCreate,
    db: AsyncSession = Depends(get_db),
):
    return await UserService.create(db, user_in)

@router.patch("/{user_id}", response_model=UserResponse)
async def update_user(
    user_id: str,
    user_in: UserUpdate,
    db: AsyncSession = Depends(get_db),
):
    return await UserService.update(db, user_id, user_in)

@router.delete("/{user_id}", status_code=204)
async def delete_user(
    user_id: str,
    db: AsyncSession = Depends(get_db),
):
    await UserService.delete(db, user_id)
```

---

## 🗄️ SQLAlchemy 核心模式

### 模型定义
```python
# src/models/user.py
from sqlalchemy import Column, String, Boolean, DateTime
from sqlalchemy.sql import func
from src.db.base_class import Base
import uuid

class User(Base):
    __tablename__ = "users"

    id = Column(String, primary_key=True, default=lambda: str(uuid.uuid4()))
    email = Column(String, unique=True, nullable=False, index=True)
    name = Column(String, nullable=False)
    hashed_password = Column(String, nullable=False)
    is_active = Column(Boolean, default=True)
    is_superuser = Column(Boolean, default=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())

    def __repr__(self):
        return f"<User(id={self.id}, email={self.email})>"
```

### 数据库连接
```python
# src/db/base.py
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from src.core.config import settings

engine = create_async_engine(
    settings.DATABASE_URL,
    echo=settings.DEBUG,
    future=True,
)

AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
)

async def get_db() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()
```

---

## ✅ Pydantic 验证模式

```python
# src/schemas/user.py
from pydantic import BaseModel, EmailStr, Field, field_validator
from typing import Optional
from datetime import datetime

class UserBase(BaseModel):
    email: EmailStr
    name: str = Field(..., min_length=2, max_length=50)

class UserCreate(UserBase):
    password: str = Field(..., min_length=8, description="密码至少8位")

    @field_validator('password')
    @classmethod
    def password_must_contain_uppercase(cls, v: str) -> str:
        if not any(c.isupper() for c in v):
            raise ValueError('密码必须包含至少一个大写字母')
        return v

class UserUpdate(UserBase):
    email: Optional[EmailStr] = None
    name: Optional[str] = Field(None, min_length=2, max_length=50)

class UserResponse(UserBase):
    id: str
    is_active: bool
    created_at: datetime

    class Config:
        from_attributes = True  # Pydantic v2

class UserListResponse(BaseModel):
    data: list[UserResponse]
    total: int
    page: int
    limit: int
```

### 环境变量验证
```python
# src/core/config.py
from pydantic_settings import BaseSettings
from typing import List

class Settings(BaseSettings):
    # API 配置
    PROJECT_NAME: str = "My API"
    VERSION: str = "1.0.0"
    API_V1_STR: str = "/api/v1"

    # 安全配置
    SECRET_KEY: str
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 60 * 24 * 7  # 7 天

    # 数据库配置
    DATABASE_URL: str

    # CORS 配置
    BACKEND_CORS_ORIGINS: List[str] = ["http://localhost:3000"]

    # 环境
    DEBUG: bool = False
    ENVIRONMENT: str = "development"

    class Config:
        env_file = ".env"
        case_sensitive = True

settings = Settings()
```

---

## 🏗️ 分层架构

### Router → Service → Repository

```python
# Router: 处理请求/响应
@router.post("/", response_model=UserResponse)
async def create_user(
    user_in: UserCreate,
    db: AsyncSession = Depends(get_db),
):
    user = await UserService.create(db, user_in)
    return user

# Service: 业务逻辑
class UserService:
    @staticmethod
    async def create(db: AsyncSession, user_in: UserCreate) -> User:
        # 检查邮箱是否存在
        existing_user = await UserRepository.get_by_email(db, user_in.email)
        if existing_user:
            raise HTTPException(status_code=409, detail="邮箱已被注册")

        # 密码哈希
        hashed_password = get_password_hash(user_in.password)

        # 创建用户
        user_data = user_in.model_dump(exclude={"password"})
        user_data["hashed_password"] = hashed_password

        return await UserRepository.create(db, user_data)

# Repository: 数据访问
class UserRepository:
    @staticmethod
    async def create(db: AsyncSession, user_data: dict) -> User:
        db_user = User(**user_data)
        db.add(db_user)
        await db.commit()
        await db.refresh(db_user)
        return db_user
```

---

## 🛡️ 中间件与依赖注入

### 认证依赖
```python
# src/core/deps.py
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.ext.asyncio import AsyncSession
from src.db.base import get_db
from src.core.security import decode_access_token
from src.models.user import User
from src.repositories.user_repository import UserRepository

oauth2_scheme = OAuth2PasswordBearer(tokenUrl=f"/api/v1/auth/login")

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="无法验证凭证",
        headers={"WWW-Authenticate": "Bearer"},
    )

    try:
        payload = decode_access_token(token)
        user_id: str = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except Exception:
        raise credentials_exception

    user = await UserRepository.get_by_id(db, user_id)
    if user is None:
        raise credentials_exception

    return user

async def get_current_active_user(
    current_user: User = Depends(get_current_user),
) -> User:
    if not current_user.is_active:
        raise HTTPException(status_code=400, detail="用户未激活")
    return current_user

async def get_current_superuser(
    current_user: User = Depends(get_current_user),
) -> User:
    if not current_user.is_superuser:
        raise HTTPException(status_code=400, detail="权限不足")
    return current_user
```

### 异常处理
```python
# src/exceptions/handlers.py
from fastapi import FastAPI, Request, status
from fastapi.responses import JSONResponse
from src.exceptions.custom import AppException

async def app_exception_handler(request: Request, exc: AppException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": exc.code, "message": exc.message},
    )

async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": "HTTP_ERROR", "message": exc.detail},
    )

async def validation_exception_handler(request: Request, exc: ValidationError):
    return JSONResponse(
        status_code=status.HTTP_400_BAD_REQUEST,
        content={"error": "VALIDATION_ERROR", "details": exc.errors()},
    )

def register_exception_handlers(app: FastAPI):
    app.add_exception_handler(AppException, app_exception_handler)
    app.add_exception_handler(HTTPException, http_exception_handler)
    app.add_exception_handler(ValidationError, validation_exception_handler)

# src/exceptions/custom.py
class AppException(Exception):
    def __init__(self, code: str, message: str, status_code: int = 400):
        self.code = code
        self.message = message
        self.status_code = status_code
        super().__init__(message)
```

---

## 🔒 安全规范

### 密码哈希与 JWT
```python
# src/core/security.py
from passlib.context import CryptContext
from jose import jwt
from datetime import datetime, timedelta
from src.core.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    return pwd_context.hash(password)

def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)

    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, settings.SECRET_KEY, algorithm="HS256")
    return encoded_jwt

def decode_access_token(token: str) -> dict:
    try:
        payload = jwt.decode(token, settings.SECRET_KEY, algorithms=["HS256"])
        return payload
    except jwt.JWTError:
        return None
```

---

## 📡 API 响应格式

```python
# 成功响应
{ "data": { ... } }
{ "data": [...], "meta": { "total": 100, "page": 1, "limit": 20, "totalPages": 5 } }

# 错误响应
{ "error": "ERROR_CODE", "message": "错误描述" }
{ "error": "VALIDATION_ERROR", "details": [...] }

# 状态码
200 OK / 201 Created / 204 No Content
400 Bad Request / 401 Unauthorized / 403 Forbidden / 404 Not Found / 409 Conflict
500 Internal Server Error
```

---

## 🎯 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| 变量/函数 | snake_case | `user_name`, `get_user()` |
| 类 | PascalCase | `UserService`, `UserRepository` |
| 常量 | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| 私有成员 | 前缀下划线 | `_private_method` |
| 模块名 | 小写 | `user_service.py` |
| 数据库表名 | 复数小写 | `users`, `user_profiles` |
| Pydantic Schema | PascalCase + 后缀 | `UserCreate`, `UserResponse` |

---

## 📋 提交前检查

```
□ ruff check --fix src/ 通过
□ pytest --cov 通过
□ 所有接口有 Pydantic 验证
□ 异常处理完整
□ 敏感操作有权限验证
□ 无 print() 遗留（使用 logger）
□ 类型注解完整
```

---

## 🧪 测试规范

### 单元测试
```python
# tests/unit/test_user_service.py
import pytest
from unittest.mock import AsyncMock, Mock
from src.services.user_service import UserService
from src.schemas.user import UserCreate
from httpx import HTTPStatusError

@pytest.mark.asyncio
async def test_create_user_success(db_session):
    # Arrange
    user_in = UserCreate(
        email="test@example.com",
        name="Test User",
        password="Password123"
    )
    mock_repo = AsyncMock(return_value=Mock(id="123", email="test@example.com"))

    # Act
    with pytest.mock.patch.object(UserRepository, 'create', mock_repo):
        user = await UserService.create(db_session, user_in)

    # Assert
    assert user.email == "test@example.com"
    mock_repo.assert_called_once()

@pytest.mark.asyncio
async def test_create_user_duplicate_email(db_session):
    user_in = UserCreate(
        email="exists@example.com",
        name="Test User",
        password="Password123"
    )

    with pytest.mock.patch.object(
        UserRepository, 'get_by_email',
        AsyncMock(return_value=Mock(id="existing"))
    ):
        with pytest.raises(HTTPException) as exc_info:
            await UserService.create(db_session, user_in)

    assert exc_info.value.status_code == 409
```

### API 集成测试
```python
# tests/integration/test_api_users.py
import pytest
from httpx import AsyncClient
from src.main import app

@pytest.mark.asyncio
async def test_create_user(client: AsyncClient):
    response = await client.post(
        "/api/v1/users/",
        json={
            "email": "test@example.com",
            "name": "Test User",
            "password": "Password123"
        }
    )

    assert response.status_code == 201
    data = response.json()
    assert data["email"] == "test@example.com"
    assert "id" in data
    assert "hashed_password" not in data

@pytest.mark.asyncio
async def test_list_users_unauthorized(client: AsyncClient):
    response = await client.get("/api/v1/users/")

    assert response.status_code == 401
```

### pytest 配置
```python
# tests/conftest.py
import pytest
import asyncio
from typing import AsyncGenerator
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from src.main import app
from src.db.base import get_db
from src.models.base import Base

TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"

engine = create_async_engine(TEST_DATABASE_URL, echo=True)
TestingSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

@pytest.fixture(scope="function")
async def db_session() -> AsyncGenerator[AsyncSession, None]:
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    async with TestingSessionLocal() as session:
        yield session
        await session.rollback()

@pytest.fixture(scope="function")
async def client(db_session: AsyncSession) -> AsyncGenerator[AsyncClient, None]:
    async def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac
    app.dependency_overrides.clear()
```

---

## 📊 SQLAlchemy 查询模式

### 常用查询
```python
from sqlalchemy import select, and_, or_, like
from sqlalchemy.orm import selectinload

# 分页查询
async def list(db: AsyncSession, skip: int = 0, limit: int = 100):
    stmt = select(User).offset(skip).limit(limit).order_by(User.created_at.desc())
    result = await db.execute(stmt)
    return result.scalars().all()

# 条件查询
async def search(db: AsyncSession, keyword: str):
    stmt = select(User).where(
        or_(
            like(User.name, f"%{keyword}%"),
            like(User.email, f"%{keyword}%")
        )
    )
    result = await db.execute(stmt)
    return result.scalars().all()

# 关联查询
async def get_with_posts(db: AsyncSession, user_id: str):
    stmt = select(User).where(User.id == user_id).options(selectinload(User.posts))
    result = await db.execute(stmt)
    return result.scalar_one_or_none()

# 计数
async def count(db: AsyncSession) -> int:
    from sqlalchemy import func
    stmt = select(func.count(User.id))
    result = await db.execute(stmt)
    return result.scalar()

# 事务
async def transfer_user_data(db: AsyncSession, from_id: str, to_id: str):
    async with db.begin():
        # 执行多个操作
        await db.execute(...)

        # 如果发生异常，自动回滚
        # 如果成功，自动提交
```

---

## 🛠️ 工具函数

### 分页助手
```python
# src/utils/pagination.py
from typing import Generic, TypeVar, List
from pydantic import BaseModel

T = TypeVar('T')

class PaginatedResponse(BaseModel, Generic[T]):
    data: List[T]
    total: int
    page: int
    limit: int
    total_pages: int
    has_next: bool
    has_prev: bool

    @classmethod
    def create(cls, data: List[T], total: int, page: int, limit: int):
        total_pages = (total + limit - 1) // limit
        return cls(
            data=data,
            total=total,
            page=page,
            limit=limit,
            total_pages=total_pages,
            has_next=page < total_pages,
            has_prev=page > 1,
        )
```

### 响应助手
```python
# src/utils/response.py
from typing import Any, Generic, TypeVar, Optional
from pydantic import BaseModel

T = TypeVar('T')

class ApiResponse(BaseModel, Generic[T]):
    data: T
    message: Optional[str] = None

class ApiError(BaseModel):
    error: str
    message: str
    details: Optional[Any] = None
```

---

## 📦 配置文件

### pyproject.toml
```toml
[project]
name = "my-api"
version = "1.0.0"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.104.0",
    "uvicorn[standard]>=0.24.0",
    "sqlalchemy[asyncio]>=2.0.0",
    "asyncpg>=0.29.0",  # PostgreSQL
    "alembic>=1.12.0",
    "pydantic>=2.4.0",
    "pydantic-settings>=2.1.0",
    "python-jose[cryptography]>=3.3.0",
    "passlib[bcrypt]>=1.7.4",
    "python-multipart>=0.0.6",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-asyncio>=0.21.0",
    "pytest-cov>=4.1.0",
    "httpx>=0.25.0",
    "black>=23.10.0",
    "ruff>=0.1.0",
    "mypy>=1.6.0",
]

[tool.black]
line-length = 100
target-version = ['py311']

[tool.ruff]
line-length = 100
target-version = "py311"
select = ["E", "F", "I", "N", "W", "UP"]
ignore = ["E501"]

[tool.ruff.per-file-ignores]
"__init__.py" = ["F401"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
pythonpath = ["."]
```

### Alembic 配置
```python
# alembic.ini
[alembic]
script_location = src/db/migrations
file_template = %%(year)d%%(month).2d%%(day).2d_%%(hour).2d%%(minute).2d%%(second).2d_%%(rev)s_%%(slug)s
sqlalchemy.url = postgresql+asyncpg://user:pass@localhost/dbname

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic
```

---

## 🌐 环境变量

```bash
# .env.example
ENVIRONMENT=development
DEBUG=true

# API
PROJECT_NAME=My API
VERSION=1.0.0
API_V1_STR=/api/v1

# 安全
SECRET_KEY=your-super-secret-key-at-least-32-characters-long
ACCESS_TOKEN_EXPIRE_MINUTES=10080

# 数据库
DATABASE_URL=postgresql+asyncpg://user:password@localhost:5432/dbname

# CORS
BACKEND_CORS_ORIGINS=["http://localhost:3000"]
```

---

## 📂 文件模板

### 新建路由模块
```python
# src/api/v1/endpoints/resource.py
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from typing import List
from src.api.deps import get_db, get_current_user
from src.schemas.resource import ResourceCreate, ResourceUpdate, ResourceResponse
from src.services.resource_service import ResourceService

router = APIRouter()

@router.get("/", response_model=List[ResourceResponse])
async def list_resources(
    skip: int = 0,
    limit: int = 100,
    db: AsyncSession = Depends(get_db),
    current_user = Depends(get_current_user),
):
    return await ResourceService.list(db, skip=skip, limit=limit)

@router.get("/{resource_id}", response_model=ResourceResponse)
async def get_resource(
    resource_id: str,
    db: AsyncSession = Depends(get_db),
):
    return await ResourceService.get_by_id(db, resource_id)

@router.post("/", response_model=ResourceResponse, status_code=201)
async def create_resource(
    resource_in: ResourceCreate,
    db: AsyncSession = Depends(get_db),
):
    return await ResourceService.create(db, resource_in)

@router.patch("/{resource_id}", response_model=ResourceResponse)
async def update_resource(
    resource_id: str,
    resource_in: ResourceUpdate,
    db: AsyncSession = Depends(get_db),
):
    return await ResourceService.update(db, resource_id, resource_in)

@router.delete("/{resource_id}", status_code=204)
async def delete_resource(
    resource_id: str,
    db: AsyncSession = Depends(get_db),
):
    await ResourceService.delete(db, resource_id)
```

### 新建模型
```python
# src/models/resource.py
from sqlalchemy import Column, String, DateTime, Text
from sqlalchemy.sql import func
from src.db.base_class import Base
import uuid

class Resource(Base):
    __tablename__ = "resources"

    id = Column(String, primary_key=True, default=lambda: str(uuid.uuid4()))
    name = Column(String, nullable=False)
    description = Column(Text)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())

    def __repr__(self):
        return f"<Resource(id={self.id}, name={self.name})>"
```

---

## ⚡ 性能优化

```
✅ 使用数据库索引优化查询
✅ 分页查询避免全表扫描
✅ 使用 asyncio.gather 并行查询
✅ 合理使用 defer/load_only 只查询需要的字段
✅ 批量写入使用 bulk_save_objects
✅ 复杂查询考虑缓存（Redis）
✅ 连接池配置合理
```

```python
# 批量插入
from sqlalchemy.orm import bulk_save_objects

async def bulk_create(db: AsyncSession, items_data: list[dict]):
    items = [Item(**data) for data in items_data]
    db.add_all(items)
    await db.commit()
    return items

# 只查询需要的字段
stmt = select(User.id, User.name).where(User.is_active == True)

# 并行查询
import asyncio

async def get_dashboard_data(db: AsyncSession):
    users, posts, stats = await asyncio.gather(
        UserRepository.list(db, limit=10),
        PostRepository.list(db, limit=10),
        StatsRepository.get(db),
    )
    return {"users": users, "posts": posts, "stats": stats}
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

## 🔐 认证模式

### 登录与 Token 生成
```python
# src/services/auth_service.py
from datetime import timedelta
from fastapi import HTTPException, status
from src.core.security import verify_password, create_access_token
from src.core.config import settings
from src.repositories.user_repository import UserRepository

class AuthService:
    @staticmethod
    async def login(db: AsyncSession, email: str, password: str) -> dict:
        user = await UserRepository.get_by_email(db, email)

        if not user or not verify_password(password, user.hashed_password):
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="邮箱或密码错误",
            )

        if not user.is_active:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="用户账号已被禁用",
            )

        access_token = create_access_token(
            data={"sub": str(user.id)},
            expires_delta=timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES),
        )

        return {
            "access_token": access_token,
            "token_type": "bearer",
            "user": {
                "id": user.id,
                "email": user.email,
                "name": user.name,
            }
        }
```

---

## 📝 日志规范

```python
# src/utils/logger.py
import logging
from src.core.config import settings

logging.basicConfig(
    level=logging.INFO if settings.DEBUG else logging.WARNING,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    handlers=[
        logging.FileHandler("logs/app.log"),
        logging.StreamHandler(),
    ],
)

logger = logging.getLogger(__name__)

# 使用
logger.info("用户创建成功", extra={"user_id": user.id})
logger.error("数据库连接失败", exc_info=True)
```

---

## 🎯 类型注解

```python
# 强制使用类型注解
from typing import List, Optional, Dict, Any, AsyncGenerator

async def get_users(
    db: AsyncSession,
    skip: int = 0,
    limit: int = 100,
) -> List[User]:
    ...

def calculate_total(
    items: List[float],
    discount: Optional[float] = None,
) -> float:
    ...
```

---

## 🔧 开发工具配置

### VSCode 配置 (.vscode/settings.json)
```json
{
  "python.defaultInterpreterPath": ".venv/bin/python",
  "python.formatting.provider": "black",
  "python.linting.enabled": true,
  "python.linting.ruffEnabled": true,
  "python.linting.mypyEnabled": true,
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.organizeImports": true
  }
}
```

### Git Ignore (.gitignore)
```
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
.venv/
venv/
ENV/

# 测试
.pytest_cache/
.coverage
htmlcov/
*.cover

# 数据库
*.db
*.sqlite3

# 环境
.env
.env.local

# IDE
.vscode/
.idea/
*.swp

# 日志
logs/
*.log
```

---

## 📚 最佳实践

### 1. 类型提示优先
```python
# ✅ 推荐
def get_user(user_id: str) -> Optional[User]:
    ...

# ❌ 避免
def get_user(user_id):
    ...
```

### 2. 使用异步 I/O
```python
# ✅ 推荐
async def create_user(user_in: UserCreate):
    # 数据库操作
    await db.execute(...)
    # HTTP 请求
    async with httpx.AsyncClient() as client:
        await client.get(...)

# ❌ 避免
def create_user(user_in: UserCreate):
    # 同步操作阻塞事件循环
    db.execute(...)
```

### 3. 资源清理
```python
# ✅ 推荐
async with AsyncSessionLocal() as session:
    yield session
    # 自动清理

# ❌ 避免
session = AsyncSessionLocal()
# 可能忘记关闭
```

---

## 🔗 相关资源

- [FastAPI 官方文档](https://fastapi.tiangolo.com/)
- [SQLAlchemy 2.0 文档](https://docs.sqlalchemy.org/en/20/)
- [Pydantic 文档](https://docs.pydantic.dev/)
- [Alembic 文档](https://alembic.sqlalchemy.org/)
- [pytest 文档](https://docs.pytest.org/)
