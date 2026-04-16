# Layer 1 硬件层设计需求文档

**版本**: v1.1  
**章节**: 第八章  
**源文档**: AIOS-5层架构设计.md  
**创建时间**: 2026-04-10  
**更新**: 2026-04-10 18:10  
**状态**: 需求中  
**优先级**: P0

---

## 一、设计原则

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Layer 1 设计原则                            │
│                                                                      │
│  LLM推理 ──────────▶ 外部API (OpenAI/Claude/MiniMax)              │
│       ▲                                                           │
│       │ 原因:                                                     │
│       │ - 无需本地GPU，成本低                                      │
│       │ - 模型更新灵活                                            │
│       │ - 可使用最新模型                                          │
│                                                                      │
│  向量嵌入 ─────────▶ 本地Ollama (nomic-embed-text)                │
│       ▲                                                           │
│       │ 原因:                                                     │
│       │ - 记忆系统需要频繁检索                                     │
│       │ - 本地延迟低                                              │
│       │ - 离线可用                                                │
│                                                                      │
│  消息通道 ─────────▶ 飞书开放平台API                               │
│       ▲                                                           │
│       │ 原因:                                                     │
│       │ - 统一入口                                                │
│       │ - 便于扩展到其他IM平台                                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、模块架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Layer 1: Hardware Layer                        │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                  External API Gateway                            │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │ │
│  │  │ OpenAI API  │  │ Claude API   │  │ MiniMax API  │        │ │
│  │  │              │  │              │  │              │        │ │
│  │  │ • GPT-4o    │  │ • Claude 3.5 │  │ • MoE        │        │ │
│  │  │ • GPT-3.5   │  │ • Opus/Sonnet│  │ • ABI        │        │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘        │ │
│  │                                                                  │ │
│  │  统一接口: LLMGateway                                           │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              │                                        │
│  ┌───────────────────────────┼───────────────────────────────────┐ │
│  │                    Local Embedding                              │ │
│  │                                                                  │ │
│  │   Ollama + nomic-embed-text (本地向量嵌入)                     │ │
│  │                                                                  │ │
│  │   用途:                                                         │ │
│  │   - 记忆系统向量存储与检索                                      │ │
│  │   - 文档相似度匹配                                             │ │
│  │   - 语义搜索                                                   │ │
│  │                                                                  │ │
│  │   配置:                                                        │ │
│  │   - endpoint: http://localhost:11434                          │ │
│  │   - model: nomic-embed-text                                    │ │
│  │   - dimension: 768                                            │ │
│  │                                                                  │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              │                                        │
│  ┌───────────────────────────┼───────────────────────────────────┐ │
│  │                    Message Gateway                              │ │
│  │                                                                  │ │
│  │   飞书开放平台 API                                               │ │
│  │   - Webhook接收消息                                             │ │
│  │   - 消息发送                                                    │ │
│  │                                                                  │ │
│  │   可扩展:                                                       │ │
│  │   - 企业微信                                                    │ │
│  │   - Discord                                                    │ │
│  │   - Telegram                                                   │ │
│  │   - Slack                                                      │ │
│  │                                                                  │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              │                                        │
│  ┌───────────────────────────┼───────────────────────────────────┐ │
│  │                    File System                                  │ │
│  │                                                                  │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │ │
│  │  │   本地磁盘    │  │   网络存储    │  │  临时文件    │        │ │
│  │  │              │  │  (可选)       │  │              │        │ │
│  │  │ /root/aios  │  │              │  │ /tmp/aios    │        │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘        │ │
│  │                                                                  │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 三、模块职责表

```
┌─────────────────────────────────────────────────────────────────────┐
│  模块              │  职责                      │  实现方式           │
├────────────────────┼────────────────────────────┼────────────────────┤
│  External LLM API  │  大语言模型对话             │  HTTP REST调用     │
│                    │  - chat                    │  - OpenAI SDK      │
│                    │  - stream_chat             │  - Anthropic SDK   │
│                    │  - embed                   │  - 自定义HTTP      │
├────────────────────┼────────────────────────────┼────────────────────┤
│  Local Embedding   │  向量嵌入生成与检索         │  Ollama + LanceDB  │
│                    │  - embed_text             │                    │
│                    │  - vector_search           │                    │
├────────────────────┼────────────────────────────┼────────────────────┤
│  Message Gateway   │  飞书消息收发               │  飞书开放平台SDK   │
│                    │  - receive_webhook         │                    │
│                    │  - send_message            │                    │
├────────────────────┼────────────────────────────┼────────────────────┤
│  File System       │  Agent持久化存储            │  标准文件系统API   │
│                    │  - read/write              │                    │
│                    │  - list/stat               │                    │
├────────────────────┼────────────────────────────┼────────────────────┤
│  HTTP Client       │  外部API调用                │  libcurl/reqwest   │
│                    │  - get/post                │                    │
└────────────────────┴────────────────────────────┴────────────────────┘
```

---

## 四、LLM Gateway 详细设计

### 4.1 接口定义

```python
# ==================== llm_gateway.py ====================
from abc import ABC, abstractmethod
from typing import List, Dict, Any, AsyncGenerator, Optional
from dataclasses import dataclass
from enum import Enum

class LLMProvider(Enum):
    OPENAI = "openai"
    ANTHROPIC = "anthropic"
    MINIMAX = "minimax"
    OLLAMA = "ollama"
    AZURE = "azure"

@dataclass
class ChatMessage:
    """聊天消息"""
    role: str  # "user", "assistant", "system"
    content: str

@dataclass
class ChatRequest:
    """聊天请求"""
    messages: List[ChatMessage]
    model: str
    temperature: float = 0.7
    max_tokens: int = 2000
    stream: bool = False
    top_p: float = 1.0
    stop: List[str] = None

@dataclass
class ChatResponse:
    """聊天响应"""
    content: str
    model: str
    usage: Dict[str, int]  # prompt_tokens, completion_tokens, total_tokens
    finish_reason: str

@dataclass
class EmbedRequest:
    """嵌入请求"""
    text: str
    model: str = "nomic-embed-text"

@dataclass
class EmbedResponse:
    """嵌入响应"""
    embedding: List[float]
    model: str
    tokens: int

class ILLMGateway(ABC):
    """LLM网关接口"""
    
    @property
    @abstractmethod
    def provider(self) -> LLMProvider:
        """提供商类型"""
        pass
    
    @abstractmethod
    async def chat(self, request: ChatRequest) -> ChatResponse:
        """
        同步聊天
        
        Args:
            request: 聊天请求
            
        Returns:
            聊天响应
        """
        pass
    
    @abstractmethod
    async def stream_chat(self, request: ChatRequest) -> AsyncGenerator[str, None]:
        """
        流式聊天
        
        Args:
            request: 聊天请求
            
        Yields:
            生成的文本片段
        """
        pass
    
    @abstractmethod
    async def embed(self, request: EmbedRequest) -> EmbedResponse:
        """
        文本嵌入
        
        Args:
            request: 嵌入请求
            
        Returns:
            嵌入响应
        """
        pass
```

### 4.2 OpenAI 实现

```python
# ==================== openai_gateway.py ====================
import openai
from typing import AsyncGenerator

class OpenAIGateway(ILLMGateway):
    """OpenAI LLM网关"""
    
    def __init__(self, api_key: str, base_url: str = None, **kwargs):
        self.client = openai.AsyncOpenAI(
            api_key=api_key,
            base_url=base_url,
            **kwargs
        )
        self.default_model = kwargs.get("model", "gpt-4o")
    
    @property
    def provider(self) -> LLMProvider:
        return LLMProvider.OPENAI
    
    async def chat(self, request: ChatRequest) -> ChatResponse:
        """同步聊天"""
        response = await self.client.chat.completions.create(
            model=request.model or self.default_model,
            messages=[{"role": m.role, "content": m.content} for m in request.messages],
            temperature=request.temperature,
            max_tokens=request.max_tokens,
            top_p=request.top_p,
            stop=request.stop,
            stream=False
        )
        
        choice = response.choices[0]
        return ChatResponse(
            content=choice.message.content,
            model=response.model,
            usage={
                "prompt_tokens": response.usage.prompt_tokens,
                "completion_tokens": response.usage.completion_tokens,
                "total_tokens": response.usage.total_tokens
            },
            finish_reason=choice.finish_reason
        )
    
    async def stream_chat(self, request: ChatRequest) -> AsyncGenerator[str, None]:
        """流式聊天"""
        stream = await self.client.chat.completions.create(
            model=request.model or self.default_model,
            messages=[{"role": m.role, "content": m.content} for m in request.messages],
            temperature=request.temperature,
            max_tokens=request.max_tokens,
            top_p=request.top_p,
            stop=request.stop,
            stream=True
        )
        
        async for chunk in stream:
            if chunk.choices and chunk.choices[0].delta.content:
                yield chunk.choices[0].delta.content
    
    async def embed(self, request: EmbedRequest) -> EmbedResponse:
        """文本嵌入"""
        response = await self.client.embeddings.create(
            model=request.model,
            input=request.text
        )
        
        return EmbedResponse(
            embedding=response.data[0].embedding,
            model=response.model,
            tokens=response.usage.total_tokens
        )
```

### 4.3 Anthropic 实现

```python
# ==================== anthropic_gateway.py ====================
import anthropic
from typing import AsyncGenerator

class AnthropicGateway(ILLMGateway):
    """Anthropic LLM网关 (Claude)"""
    
    def __init__(self, api_key: str, **kwargs):
        self.client = anthropic.AsyncAnthropic(api_key=api_key)
        self.default_model = kwargs.get("model", "claude-3-5-sonnet-20241022")
    
    @property
    def provider(self) -> LLMProvider:
        return LLMProvider.ANTHROPIC
    
    async def chat(self, request: ChatRequest) -> ChatResponse:
        """同步聊天"""
        response = await self.client.messages.create(
            model=request.model or self.default_model,
            messages=[{"role": m.role, "content": m.content} for m in request.messages],
            temperature=request.temperature,
            max_tokens=request.max_tokens,
            stop_sequence=request.stop[0] if request.stop else None
        )
        
        return ChatResponse(
            content=response.content[0].text,
            model=response.model,
            usage={
                "prompt_tokens": response.usage.input_tokens,
                "completion_tokens": response.usage.output_tokens,
                "total_tokens": response.usage.input_tokens + response.usage.output_tokens
            },
            finish_reason=response.stop_reason
        )
    
    async def stream_chat(self, request: ChatRequest) -> AsyncGenerator[str, None]:
        """流式聊天"""
        async with self.client.messages.stream(
            model=request.model or self.default_model,
            messages=[{"role": m.role, "content": m.content} for m in request.messages],
            temperature=request.temperature,
            max_tokens=request.max_tokens,
            stop_sequence=request.stop[0] if request.stop else None
        ) as stream:
            async for text in stream.text_stream:
                yield text
    
    async def embed(self, request: EmbedRequest) -> EmbedResponse:
        """Claude不原生支持embedding，使用OpenAI兼容接口"""
        # 可选: 调用OpenAI或其他embedding服务
        raise NotImplementedError("Anthropic does not support embeddings API")
```

### 4.4 LLM Factory

```python
# ==================== llm_factory.py ====================
class LLMFactory:
    """LLM工厂类"""
    
    _gateways: Dict[LLMProvider, Type[ILLMGateway]] = {
        LLMProvider.OPENAI: OpenAIGateway,
        LLMProvider.ANTHROPIC: AnthropicGateway,
        # 可以添加更多provider
    }
    
    @classmethod
    def create(cls, provider: str, api_key: str, **kwargs) -> ILLMGateway:
        """
        创建LLM网关实例
        
        Args:
            provider: 提供商名称 ("openai", "anthropic", "minimax")
            api_key: API密钥
            **kwargs: 其他配置参数
            
        Returns:
            LLM网关实例
        """
        provider_enum = LLMProvider(provider.lower())
        
        if provider_enum not in cls._gateways:
            raise ValueError(f"Unsupported provider: {provider}")
        
        gateway_class = cls._gateways[provider_enum]
        return gateway_class(api_key=api_key, **kwargs)
    
    @classmethod
    def register(cls, provider: LLMProvider, gateway_class: Type[ILLMGateway]):
        """注册新的provider"""
        cls._gateways[provider] = gateway_class
```

---

## 五、向量存储详细设计

### 5.1 向量存储接口

```python
# ==================== vector_store.py ====================
from abc import ABC, abstractmethod
from typing import List, Tuple, Optional, Any
from dataclasses import dataclass

@dataclass
class VectorEntry:
    """向量条目"""
    id: str
    vector: List[float]
    text: str
    metadata: dict
    created_at: float
    updated_at: float

@dataclass
class SearchResult:
    """搜索结果"""
    id: str
    score: float
    text: str
    metadata: dict

class IVectorStore(ABC):
    """向量存储接口"""
    
    @abstractmethod
    async def add(self, entries: List[VectorEntry]) -> List[str]:
        """
        添加向量
        
        Args:
            entries: 向量条目列表
            
        Returns:
            添加的ID列表
        """
        pass
    
    @abstractmethod
    async def search(self, query_vector: List[float], 
                     top_k: int = 5, 
                     filter_func: callable = None) -> List[SearchResult]:
        """
        搜索相似向量
        
        Args:
            query_vector: 查询向量
            top_k: 返回数量
            filter_func: 过滤函数 (可选)
            
        Returns:
            搜索结果列表
        """
        pass
    
    @abstractmethod
    async def delete(self, ids: List[str]) -> bool:
        """删除向量"""
        pass
    
    @abstractmethod
    async def get(self, id: str) -> Optional[VectorEntry]:
        """获取单个向量"""
        pass
    
    @abstractmethod
    async def update(self, id: str, entry: VectorEntry) -> bool:
        """更新向量"""
        pass
```

### 5.2 LanceDB 实现

```python
# ==================== lancedb_store.py ====================
import lancedb
from typing import List, Callable, Optional
import numpy as np

class LanceDBStore(IVectorStore):
    """LanceDB向量存储"""
    
    def __init__(self, db_path: str = "/root/aios/data/vector_db"):
        self.db_path = db_path
        self.db = lancedb.connect(db_path)
        self.table = None
        self._init_table()
    
    def _init_table(self):
        """初始化表"""
        schema = {
            "vector": "vector<float>(768)",  # nomic-embed-text dimension
            "text": "text",
            "id": "text",
            "metadata": "json",
            "created_at": "float",
            "updated_at": "float"
        }
        
        if "embeddings" not in self.table_names():
            self.db.create_table("embeddings", schema=schema)
        
        self.table = self.db.open_table("embeddings")
    
    def table_names(self) -> List[str]:
        return self.db.table_names()
    
    async def add(self, entries: List[VectorEntry]) -> List[str]:
        """添加向量"""
        data = [
            {
                "id": e.id,
                "vector": e.vector,
                "text": e.text,
                "metadata": json.dumps(e.metadata),
                "created_at": e.created_at,
                "updated_at": e.updated_at
            }
            for e in entries
        ]
        
        self.table.add(data)
        return [e.id for e in entries]
    
    async def search(self, query_vector: List[float], 
                     top_k: int = 5,
                     filter_func: Callable = None) -> List[SearchResult]:
        """搜索相似向量"""
        results = self.table.search(query_vector).limit(top_k)
        
        if filter_func:
            results = results.where(filter_func)
        
        output = []
        for row in results.to_list():
            output.append(SearchResult(
                id=row["id"],
                score=1.0 - row["distance"],  # LanceDB用distance
                text=row["text"],
                metadata=json.loads(row["metadata"])
            ))
        
        return output
    
    async def delete(self, ids: List[str]) -> bool:
        """删除向量"""
        self.table.delete(f"id IN ({','.join(ids)})")
        return True
    
    async def get(self, id: str) -> Optional[VectorEntry]:
        """获取单个向量"""
        results = self.table.search([0] * 768).where(f"id = '{id}'").limit(1).to_list()
        
        if not results:
            return None
        
        row = results[0]
        return VectorEntry(
            id=row["id"],
            vector=row["vector"],
            text=row["text"],
            metadata=json.loads(row["metadata"]),
            created_at=row["created_at"],
            updated_at=row["updated_at"]
        )
    
    async def update(self, id: str, entry: VectorEntry) -> bool:
        """更新向量"""
        self.table.update(
            where=f"id = '{id}'",
            values={
                "vector": entry.vector,
                "text": entry.text,
                "metadata": json.dumps(entry.metadata),
                "updated_at": entry.updated_at
            }
        )
        return True
```

---

## 六、消息网关详细设计

### 6.1 消息接口

```python
# ==================== message_gateway.py ====================
from abc import ABC, abstractmethod
from typing import List, Optional
from dataclasses import dataclass
from enum import Enum

class MessageType(Enum):
    TEXT = "text"
    IMAGE = "image"
    FILE = "file"
    AUDIO = "audio"
    VIDEO = "video"

@dataclass
class Message:
    """消息"""
    msg_id: str
    msg_type: MessageType
    content: str
    from_user: str
    to_user: str
    timestamp: float
    metadata: dict

@dataclass
class ReceivedMessage:
    """接收到的消息"""
    message: Message
    raw_data: dict  # 原始数据

class IMessageGateway(ABC):
    """消息网关接口"""
    
    @abstractmethod
    async def send_message(self, message: Message) -> bool:
        """
        发送消息
        
        Args:
            message: 消息对象
            
        Returns:
            是否发送成功
        """
        pass
    
    @abstractmethod
    async def receive_messages(self) -> List[ReceivedMessage]:
        """
        接收消息
        
        Returns:
            消息列表
        """
        pass
    
    @abstractmethod
    def register_webhook_handler(self, handler: callable):
        """
        注册webhook处理函数
        
        Args:
            handler: 处理函数
        """
        pass
```

### 6.2 飞书实现

```python
# ==================== feishu_gateway.py ====================
import httpx
from typing import List
from dataclasses import dataclass

@dataclass
class FeishuConfig:
    """飞书配置"""
    app_id: str
    app_secret: str
    webhook_url: str = None
    verification_token: str = None

class FeishuGateway(IMessageGateway):
    """飞书消息网关"""
    
    def __init__(self, config: FeishuConfig):
        self.config = config
        self.client = httpx.AsyncClient()
        self.access_token = None
        self.token_expires_at = 0
    
    async def _get_access_token(self) -> str:
        """获取access_token"""
        import time
        if self.access_token and time.time() < self.token_expires_at:
            return self.access_token
        
        response = await self.client.post(
            "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal",
            json={
                "app_id": self.config.app_id,
                "app_secret": self.config.app_secret
            }
        )
        
        data = response.json()
        self.access_token = data["tenant_access_token"]
        self.token_expires_at = time.time() + data.get("expire", 7200) - 60
        
        return self.access_token
    
    async def send_message(self, message: Message) -> bool:
        """发送消息"""
        token = await self._get_access_token()
        
        # 根据消息类型构建不同的content
        if message.msg_type == MessageType.TEXT:
            content = {"text": message.content}
        elif message.msg_type == MessageType.IMAGE:
            content = {"image_key": message.metadata.get("image_key")}
        else:
            content = {"text": message.content}
        
        response = await self.client.post(
            "https://open.feishu.cn/open-apis/im/v1/messages",
            params={"receive_id_type": "open_id"},
            headers={"Authorization": f"Bearer {token}"},
            json={
                "receive_id": message.to_user,
                "msg_type": message.msg_type.value,
                "content": json.dumps(content)
            }
        )
        
        return response.status_code == 200
    
    async def receive_messages(self) -> List[ReceivedMessage]:
        """
        接收消息 (被动接收，通过webhook)
        
        飞书使用webhook模式，这里返回空列表
        实际消息通过webhook_handler处理
        """
        return []
    
    def register_webhook_handler(self, handler: callable):
        """注册webhook处理函数"""
        self.webhook_handler = handler
    
    async def handle_webhook(self, payload: dict) -> Message:
        """处理webhook事件"""
        event = payload.get("event", {})
        
        message = Message(
            msg_id=event.get("message_id", ""),
            msg_type=MessageType(event.get("msg_type", "text")),
            content=event.get("content", ""),
            from_user=event.get("sender", {}).get("open_id", ""),
            to_user=event.get("chat_id", ""),
            timestamp=event.get("create_time", 0) / 1000,
            metadata=payload
        )
        
        if hasattr(self, "webhook_handler"):
            await self.webhook_handler(message)
        
        return message
```

---

## 七、文件系统接口

```python
# ==================== file_system.py ====================
from abc import ABC, abstractmethod
from typing import List, Optional, AsyncGenerator
from dataclasses import dataclass
from enum import Enum

class FileType(Enum):
    FILE = "file"
    DIRECTORY = "directory"
    SYMBOLIC_LINK = "symlink"

@dataclass
class FileInfo:
    """文件信息"""
    path: str
    name: str
    file_type: FileType
    size: int
    created_at: float
    modified_at: float
    accessed_at: float
    permissions: str

@dataclass
class FileHandle:
    """文件句柄"""
    fd: int
    path: str
    mode: str
    offset: int = 0

class IFileSystem(ABC):
    """文件系统接口"""
    
    @abstractmethod
    async def open(self, path: str, mode: str) -> FileHandle:
        """打开文件"""
        pass
    
    @abstractmethod
    async def read(self, handle: FileHandle, size: int) -> bytes:
        """读取文件"""
        pass
    
    @abstractmethod
    async def write(self, handle: FileHandle, data: bytes) -> int:
        """写入文件"""
        pass
    
    @abstractmethod
    async def close(self, handle: FileHandle) -> bool:
        """关闭文件"""
        pass
    
    @abstractmethod
    async def list_directory(self, path: str) -> List[FileInfo]:
        """列出目录"""
        pass
    
    @abstractmethod
    async def get_file_info(self, path: str) -> FileInfo:
        """获取文件信息"""
        pass
    
    @abstractmethod
    async def create_directory(self, path: str) -> bool:
        """创建目录"""
        pass
    
    @abstractmethod
    async def delete(self, path: str) -> bool:
        """删除文件/目录"""
        pass
    
    @abstractmethod
    async def move(self, src: str, dst: str) -> bool:
        """移动文件"""
        pass
    
    @abstractmethod
    async def copy(self, src: str, dst: str) -> bool:
        """复制文件"""
        pass
    
    @abstractmethod
    async def exists(self, path: str) -> bool:
        """检查文件是否存在"""
        pass
```

---

## 八、Layer 1 接口定义

```python
# ==================== layer1.py ====================
class ILayer1:
    """
    Layer 1 接口定义
    
    这是硬件层的统一接口，定义所有硬件相关操作
    """
    
    # ==================== LLM接口 ====================
    async def chat(self, request: ChatRequest) -> ChatResponse:
        """大语言模型对话"""
        pass
    
    async def stream_chat(self, request: ChatRequest) -> AsyncGenerator[str, None]:
        """流式对话"""
        pass
    
    async def embed(self, request: EmbedRequest) -> EmbedResponse:
        """向量嵌入"""
        pass
    
    # ==================== 向量存储接口 ====================
    async def vector_search(self, query: str, top_k: int = 5, 
                           filter_func: callable = None) -> List[SearchResult]:
        """向量搜索"""
        pass
    
    async def vector_add(self, entries: List[VectorEntry]) -> List[str]:
        """添加向量"""
        pass
    
    async def vector_delete(self, ids: List[str]) -> bool:
        """删除向量"""
        pass
    
    # ==================== 消息网关接口 ====================
    async def send_message(self, message: Message) -> bool:
        """发送消息"""
        pass
    
    async def receive_message(self) -> Optional[Message]:
        """接收消息"""
        pass
    
    # ==================== 文件系统接口 ====================
    async def open(self, path: str, mode: str) -> FileHandle:
        """打开文件"""
        pass
    
    async def read(self, handle: FileHandle, size: int) -> bytes:
        """读取文件"""
        pass
    
    async def write(self, handle: FileHandle, data: bytes) -> int:
        """写入文件"""
        pass
    
    async def close(self, handle: FileHandle) -> bool:
        """关闭文件"""
        pass
    
    async def list_directory(self, path: str) -> List[FileInfo]:
        """列出目录"""
        pass
    
    # ==================== HTTP Client接口 ====================
    async def http_get(self, url: str, headers: dict = None) -> HttpResponse:
        """HTTP GET"""
        pass
    
    async def http_post(self, url: str, body: str, headers: dict = None) -> HttpResponse:
        """HTTP POST"""
        pass
```

---

## 九、与传统OS的类比

```
传统Linux Layer 1          │     AIOS Layer 1
────────────────────────────────────────────────────────────────────
CPU/GPU 物理计算          │     CPU计算 (无本地LLM)
                         │     - LLM推理走外部API
                         │     - 本地只做调度和结果处理
                         │
内存管理                  │     内存管理 (Agent内存分配)
                         │     - 每个Agent独立内存空间
                         │     - 内存配额控制
                         │
磁盘驱动                  │     文件系统 (本地磁盘)
                         │     - /root/aios/agents/{id}/
                         │     - Sandbox机制
                         │
网卡驱动                  │     HTTP Client (外部通信)
                         │     - LLM API调用
                         │     - 消息网关通信
                         │
声卡/显示器               │     消息网关 (飞书/其他IM)
                         │     - 用户交互接口
                         │
GPU CUDA驱动              │     本地向量嵌入 (Ollama)
                         │     - nomic-embed-text
                         │     - LanceDB向量存储
                         │
关键差异:
- AIOS不管理本地GPU (LLM走外部API)
- 本地计算主要是向量嵌入 (记忆检索)
- 通信主要是HTTP REST (调用外部服务)
- 消息网关替代传统显示/音频设备
```

---

## 十、请求处理示例

### 10.1 用户通过飞书发送消息

```
用户通过飞书发送: "帮我整理桌面的PDF"

Layer 1 处理流程:

1. 飞书Webhook接收
   └─> FeishuGateway.handle_webhook()
   └─> 解析消息内容: "帮我整理桌面的PDF"

2. 上层处理: Layer 5 Central Agent
   └─> Intent Parser: domain="file", action="organize"
   └─> Task Planner: 创建SubTask
   └─> FileAgent.execute()

3. Layer 3 系统调用
   └─> FS_Driver处理文件操作

4. Layer 1 File System
   └─> list_files("/Desktop/*.pdf")
   └─> move_file() → 移动到 /Documents/PDF/

5. 结果返回给上层

6. 飞书消息发送
   └─> FeishuGateway.send_message()
   └─> 发送结果给用户: "已整理2个PDF文件到 /Documents/PDF/"
```

### 10.2 Agent调用LLM

```
Agent需要调用LLM进行决策:

1. Layer 5 Agent调用
   └─> llm_ops.chat()

2. Layer 3 系统调用
   └─> LLM_Driver.dispatch()

3. Layer 1 LLM Gateway
   └─> OpenAIGateway.chat()
   └─> HTTP POST到OpenAI API

4. 返回结果给Agent
```

---

## 十一、简化后的优势

```
✅ 降低复杂度 
   - 无需本地LLM推理
   - 无GPU资源管理
   - 无模型更新维护

✅ 降低成本
   - 使用外部API按量计费
   - 本地只需跑向量嵌入模型
   - 避免GPU采购和维护

✅ 灵活扩展
   - 轻松切换LLM provider
   - 消息网关可扩展到多平台
   - 支持新的硬件抽象

✅ 专注核心
   - 精力集中在Agent编排和调度
   - 记忆系统作为核心竞争力
   - 架构清晰，易于维护
```

---

## 十二、验收标准

- [ ] LLM Gateway 接口定义完整
- [ ] OpenAI Gateway 实现正确
- [ ] Anthropic Gateway 实现正确
- [ ] LLM Factory 工厂模式正确
- [ ] 向量存储接口定义完整
- [ ] LanceDB 实现正确
- [ ] 消息网关接口定义完整
- [ ] 飞书Gateway 实现正确
- [ ] 文件系统接口定义完整
- [ ] HTTP Client 实现正确
- [ ] Layer 1 统一接口定义完整
- [ ] 模块实现可替换（接口抽象）

---

*文档版本: v1.1*
*创建时间: 2026-04-10*
*更新: 2026-04-10 18:10 - 增强各模块详细设计和实现*
