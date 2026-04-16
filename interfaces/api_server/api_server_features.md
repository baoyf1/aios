# API服务器功能文档

## 功能概述
API服务器是Windows版本AIOS系统的重要组成部分，提供了RESTful API接口和WebSocket支持，用于与外部系统和前端应用集成。

## 核心功能

### 1. FastAPI实现

#### 1.1 基础API结构
```python
from fastapi import FastAPI

app = FastAPI(
    title="AIOS API",
    description="AIOS系统的RESTful API接口",
    version="1.0.0"
)

@app.get("/health")
def health_check():
    """
    健康检查接口
    """
    return {"status": "healthy"}
```

#### 1.2 路由管理
```python
from fastapi import APIRouter

api_router = APIRouter()

@api_router.get("/agents")
def list_agents():
    """
    列出所有Agent
    """
    pass

@api_router.post("/agents")
def create_agent(agent_data: dict):
    """
    创建新Agent
    """
    pass

app.include_router(api_router, prefix="/api/v1")
```

#### 1.3 请求验证
```python
from pydantic import BaseModel

class AgentCreate(BaseModel):
    name: str
    description: str
    type: str

@api_router.post("/agents")
def create_agent(agent: AgentCreate):
    """
    创建新Agent
    """
    pass
```

### 2. WebSocket支持

#### 2.1 WebSocket连接
```python
from fastapi import WebSocket

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    """
    WebSocket端点
    """
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"Message received: {data}")
    except Exception as e:
        print(f"WebSocket error: {e}")
    finally:
        await websocket.close()
```

#### 2.2 连接管理
```python
class ConnectionManager:
    def __init__(self):
        self.active_connections: List[WebSocket] = []
    
    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)
    
    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)
    
    async def broadcast(self, message: str):
        for connection in self.active_connections:
            await connection.send_text(message)

manager = ConnectionManager()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_text()
            await manager.broadcast(f"Client says: {data}")
    except Exception as e:
        print(f"WebSocket error: {e}")
    finally:
        manager.disconnect(websocket)
```

### 3. 认证授权

#### 3.1 JWT认证
```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/login")

SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

def verify_password(plain_password, hashed_password):
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password):
    return pwd_context.hash(password)

def create_access_token(data: dict):
    to_encode = data.copy()
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

async def get_current_user(token: str = Depends(oauth2_scheme)):
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception
    return username

@app.post("/api/v1/login")
def login(form_data: OAuth2PasswordRequestForm = Depends()):
    """
    用户登录
    """
    # 验证用户
    # 生成token
    pass
```

#### 3.2 权限控制
```python
from fastapi import Depends, HTTPException, status

async def get_current_active_user(current_user: str = Depends(get_current_user)):
    # 检查用户是否活跃
    return current_user

async def require_admin(current_user: str = Depends(get_current_active_user)):
    # 检查用户是否为管理员
    if not is_admin(current_user):
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Not enough permissions"
        )
    return current_user

@api_router.post("/agents", dependencies=[Depends(require_admin)])
def create_agent(agent: AgentCreate):
    """
    创建新Agent（需要管理员权限）
    """
    pass
```

### 4. API文档

#### 4.1 文档配置
```python
app = FastAPI(
    title="AIOS API",
    description="AIOS系统的RESTful API接口",
    version="1.0.0",
    docs_url="/docs",
    redoc_url="/redoc",
    openapi_url="/openapi.json"
)
```

#### 4.2 文档增强
```python
from fastapi.openapi.utils import get_openapi

def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema
    openapi_schema = get_openapi(
        title="AIOS API",
        version="1.0.0",
        description="AIOS系统的RESTful API接口",
        routes=app.routes,
    )
    # 自定义openapi_schema
    app.openapi_schema = openapi_schema
    return app.openapi_schema

app.openapi = custom_openapi
```

## 技术实现

### 1. 架构设计
- **分层设计**：路由层、业务逻辑层、数据访问层
- **模块化**：API端点作为独立模块
- **中间件**：认证、日志、错误处理

### 2. 核心组件
- **FastAPI**：API框架
- **Uvicorn**：ASGI服务器
- **Pydantic**：数据验证
- **JWT**：认证
- **WebSocket**：实时通信

### 3. 数据流
1. **请求接收**：FastAPI接收HTTP请求或WebSocket连接
2. **认证授权**：验证用户身份和权限
3. **请求处理**：执行业务逻辑
4. **响应生成**：返回HTTP响应或WebSocket消息
5. **日志记录**：记录请求和响应信息

## 配置选项

### 1. 服务器配置
```yaml
api_server:
  host: "0.0.0.0"
  port: 8000
  debug: false
  reload: false
  workers: 4
```

### 2. 认证配置
```yaml
auth:
  secret_key: "your-secret-key"
  algorithm: "HS256"
  access_token_expire_minutes: 30
  api_key_header: "X-API-Key"
  api_keys:
    - "api-key-1"
    - "api-key-2"
```

### 3. CORS配置
```yaml
cors:
  enabled: true
  origins:
    - "*"
  methods:
    - "GET"
    - "POST"
    - "PUT"
    - "DELETE"
  headers:
    - "Content-Type"
    - "Authorization"
```

## 错误处理

### 1. 异常处理
```python
from fastapi import HTTPException, Request
from fastapi.responses import JSONResponse

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"detail": exc.detail},
    )

@app.exception_handler(Exception)
async def general_exception_handler(request: Request, exc: Exception):
    return JSONResponse(
        status_code=500,
        content={"detail": "Internal server error"},
    )
```

### 2. 错误响应模型
```python
from pydantic import BaseModel

class ErrorResponse(BaseModel):
    detail: str
    error_code: str
    timestamp: str

@api_router.get("/agents/{agent_id}")
def get_agent(agent_id: str):
    if not agent_exists(agent_id):
        raise HTTPException(
            status_code=404,
            detail="Agent not found"
        )
    return get_agent_by_id(agent_id)
```

## 性能优化

### 1. 异步处理
```python
@app.get("/agents")
async def list_agents():
    """
    列出所有Agent
    """
    agents = await get_agents()
    return agents
```

### 2. 缓存
```python
from fastapi import Depends
from fastapi_cache import FastAPICache
from fastapi_cache.backends.memory import MemoryBackend
from fastapi_cache.decorator import cache

FastAPICache.init(MemoryBackend())

@api_router.get("/agents")
@cache(expire=60)
async def list_agents():
    """
    列出所有Agent（缓存60秒）
    """
    agents = await get_agents()
    return agents
```

### 3. 连接池
```python
import aiohttp

async def get_session():
    async with aiohttp.ClientSession() as session:
        yield session

@api_router.get("/external-api")
async def call_external_api(session: aiohttp.ClientSession = Depends(get_session)):
    """
    调用外部API
    """
    async with session.get("https://api.example.com") as response:
        return await response.json()
```

## 安全考虑

### 1. HTTPS
```python
# 生产环境配置
# uvicorn main:app --host 0.0.0.0 --port 8000 --ssl-keyfile=key.pem --ssl-certfile=cert.pem
```

### 2. 输入验证
```python
from pydantic import BaseModel, Field

class AgentCreate(BaseModel):
    name: str = Field(..., min_length=1, max_length=50)
    description: str = Field(..., max_length=200)
    type: str = Field(..., regex="^[a-zA-Z0-9_]+$")
```

### 3. 速率限制
```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@api_router.get("/agents")
@limiter.limit("10/minute")
async def list_agents():
    """
    列出所有Agent（每分钟最多10次请求）
    """
    pass
```

## 集成与扩展

### 1. 与其他模块的集成
- 与Agent系统集成，提供Agent管理API
- 与工具系统集成，提供工具调用API
- 与配置系统集成，提供配置管理API

### 2. 扩展点
- 自定义路由
- 中间件扩展
- 认证方式扩展
- 文档自定义