# Agent Communication Protocol (ACP) - Complete Guide

## Mục lục
- [ACP là gì?](#acp-là-gì)
- [Tại sao cần ACP?](#tại-sao-cần-acp)
- [Đặc điểm chính](#đặc-điểm-chính)
- [Kiến trúc và Core Concepts](#kiến-trúc-và-core-concepts)
- [So sánh với các Protocol khác](#so-sánh-với-các-protocol-khác)
- [Hướng dẫn cài đặt](#hướng-dẫn-cài-đặt)
- [Ví dụ thực tế](#ví-dụ-thực-tế)
- [Tài nguyên và Community](#tài-nguyên-và-community)

## ACP là gì?

**Agent Communication Protocol (ACP)** là một giao thức mở với quản trị mở (open governance) được thiết kế để giải quyết thách thức ngày càng tăng trong việc kết nối các AI agents, ứng dụng và con người.

ACP được phát triển bởi IBM và BeeAI, hiện là một phần của chương trình Linux Foundation AI & Data, tạo ra một chuẩn giao tiếp thống nhất cho các AI agents hoạt động trên các framework, team và hạ tầng khác nhau.

### Định nghĩa kỹ thuật
Trong ACP, một agent là một dịch vụ phần mềm giao tiếp thông qua các thông điệp đa phương tiện (multimodal messages), chủ yếu được điều khiển bằng ngôn ngữ tự nhiên thông qua RESTful API chuẩn hóa.

## Tại sao cần ACP?

### Vấn đề hiện tại
- **Phân mảnh**: Các AI agents hiện tại thường được xây dựng độc lập trên các framework khác nhau (LangGraph, Microsoft Autogen, OpenAI Agents, BeeAI, v.v.)
- **Thiếu khả năng tương tác**: Agents không thể giao tiếp hiệu quả với nhau
- **Tích hợp phức tạp**: Cần viết custom integrations mỗi khi team cập nhật thiết kế agent hoặc đổi framework
- **Chậm trễ đổi mới**: Sự phân mảnh làm chậm quá trình đổi mới và khiến việc hợp tác giữa các agents trở nên khó khăn

### Giải pháp ACP mang lại
- **Chuẩn hóa giao tiếp**: Tạo ra ngôn ngữ chung cho các agents
- **Khả năng tương tác**: Cho phép agents từ các framework khác nhau làm việc cùng nhau
- **Giảm phức tạp**: Loại bỏ nhu cầu custom integrations
- **Mở rộng dễ dàng**: Hỗ trợ mở rộng hệ thống multi-agent một cách linh hoạt

## Đặc điểm chính

### 🔄 **Giao tiếp đa dạng**
- **Synchronous**: Giao tiếp đồng bộ cho các tác vụ cần phản hồi ngay lập tức
- **Asynchronous**: Giao tiếp bất đồng bộ cho các tác vụ dài hạn
- **Streaming**: Hỗ trợ streaming cho dữ liệu real-time

### 🌐 **RESTful API**
- Sử dụng chuẩn REST APIs quen thuộc
- Dễ dàng tích hợp với hệ thống hiện có
- HTTP-based communication

### 📱 **Multimodal Messages**
- Hỗ trợ nhiều loại nội dung: text, image, JSON, file
- Cấu trúc message linh hoạt với MessageParts

### ⚡ **High Availability**
- Hỗ trợ thiết lập fault-tolerant
- Khả năng chịu lỗi cao

### 🔒 **Local-first**
- Được thiết kế cho giao tiếp local, độ trễ thấp
- Tối ưu cho edge computing và team-specific setups

## Kiến trúc và Core Concepts

### 1. **Agent**
Một service software giao tiếp thông qua multimodal messages, được điều khiển chủ yếu bằng natural language.

### 2. **Message Structure**
```json
{
  "parts": [
    {
      "content": "Hello, world!",
      "content_type": "text/plain",
      "content_encoding": "plain"
    }
  ],
  "role": "user"  // Để nhận diện agent
}
```

### 3. **MessagePart**
Các thành phần cấu tạo nên Message, có thể là text, image, JSON, v.v.

### 4. **Run**
Một phiên thực thi của agent, quản lý lifecycle và state của một tác vụ cụ thể.

### 5. **Await Mechanism**
Cho phép agents tạm dừng và chờ input từ user hoặc agent khác trong quá trình thực thi.

## So sánh với các Protocol khác

| Đặc điểm | MCP | ACP | A2A |
|----------|-----|-----|-----|
| **Phạm vi** | Single model + tools | Multiple agents collaboration | Web-native cross-vendor |
| **Mục tiêu** | Context enrichment | Agent coordination | Flexible interactions |
| **Kiến trúc** | Model ↔ Tools | Agent ↔ Agent | Agent ↔ Agent (web-native) |
| **Tối ưu cho** | Tool integration | Local/edge systems | Cloud-based collaboration |
| **Giao tiếp** | Request-response | RESTful structured | Natural language |

### ACP vs MCP
- **MCP**: Tập trung mở rộng khả năng của một model đơn lẻ
- **ACP**: Tập trung vào sự hợp tác giữa nhiều agents

### ACP vs A2A
- **ACP**: Local-first, structured RESTful interfaces
- **A2A**: Web-native, flexible natural interactions

## Hướng dẫn cài đặt

### Yêu cầu hệ thống
- Python >= 3.11
- uv (package manager được khuyến nghị)

### Bước 1: Khởi tạo project
```bash
uv init --python '>=3.11' my_acp_project
cd my_acp_project
```

### Bước 2: Cài đặt ACP SDK
```bash
uv add acp-sdk
```

### Bước 3: Tạo Agent đơn giản
Tạo file `agent.py`:

```python
# agent.py
import asyncio
from collections.abc import AsyncGenerator
from acp_sdk.models import Message
from acp_sdk.server import Context, RunYield, RunYieldResume, Server

server = Server()

@server.agent()
async def echo(
    input: list[Message], context: Context
) -> AsyncGenerator[RunYield, RunYieldResume]:
    """Echoes everything"""
    for message in input:
        await asyncio.sleep(0.5)
        yield {"thought": "I should echo everything"}
        await asyncio.sleep(0.5)
        yield message

server.run()
```

### Bước 4: Khởi chạy ACP Server
```bash
uv run agent.py
```

Server sẽ chạy tại `http://localhost:8000`

### Bước 5: Verify Agent
```bash
curl http://localhost:8000/agents
```

Kết quả:
```json
{
  "agents": [
    { "name": "echo", "description": "Echoes everything", "metadata": {} }
  ]
}
```

### Bước 6: Test Agent qua HTTP
```bash
curl -X POST http://localhost:8000/runs \
  -H "Content-Type: application/json" \
  -d '{
    "agent_name": "echo",
    "input": [
      {
        "parts": [
          {
            "content": "Howdy!",
            "content_type": "text/plain"
          }
        ]
      }
    ]
  }'
```

### Bước 7: Tạo ACP Client
Tạo file `client.py`:

```python
# client.py
import asyncio
from acp_sdk.client import Client
from acp_sdk.models import Message, MessagePart

async def example() -> None:
    async with Client(base_url="http://localhost:8000") as client:
        run = await client.run_sync(
            agent="echo",
            input=[
                Message(
                    parts=[MessagePart(
                        content="Howdy to echo from client!!", 
                        content_type="text/plain"
                    )]
                )
            ],
        )
        print(run.output)

if __name__ == "__main__":
    asyncio.run(example())
```

### Bước 8: Chạy Client
```bash
uv run client.py
```

## Ví dụ thực tế

### Multi-Agent System
```python
# Ví dụ: Coding Agent + Research Agent collaboration
@server.agent()
async def coding_agent(input: list[Message], context: Context):
    """Agent chuyên về coding"""
    # Xử lý yêu cầu coding
    yield {"thought": "Analyzing code requirements..."}
    # Implementation...

@server.agent()
async def research_agent(input: list[Message], context: Context):
    """Agent chuyên về research"""
    # Xử lý yêu cầu research
    yield {"thought": "Gathering research data..."}
    # Implementation...
```

### Streaming Response
```python
@server.agent()
async def streaming_agent(input: list[Message], context: Context):
    """Agent với streaming response"""
    for i in range(10):
        await asyncio.sleep(0.1)
        yield {"partial_result": f"Processing step {i+1}/10"}
```

### Error Handling
```python
@server.agent()
async def robust_agent(input: list[Message], context: Context):
    """Agent với error handling"""
    try:
        # Process input
        yield {"status": "processing"}
        # ... logic ...
        yield {"status": "completed"}
    except Exception as e:
        yield {"error": str(e), "status": "failed"}
```

## Available Libraries và SDKs

### Python SDK
- **Server implementation**: Tạo ACP agents
- **Client libraries**: Tương tác với ACP agents  
- **Model definitions**: Cấu trúc dữ liệu chuẩn

### TypeScript SDK
- **Client library**: Full TypeScript support
- **Type definitions**: Type-safe interactions

### OpenAPI Specification
- **REST endpoints**: Đầy đủ API documentation
- **Request/Response formats**: Chuẩn hóa data models
- **Protocol definition**: Đặc tả kỹ thuật chi tiết

## Integration với hệ thống khác

### ACP-MCP Bridge
Có thể tích hợp ACP với MCP thông qua bridge server:
```bash
# Example bridge setup
git clone https://github.com/GongRzhe/ACP-MCP-Server
# Follow setup instructions
```

### Docker Support
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["python", "agent.py"]
```

## Best Practices

### 1. **Agent Design**
- Giữ agents tập trung vào một chức năng cụ thể
- Implement proper error handling
- Sử dụng async/await đúng cách

### 2. **Message Structure**
- Sử dụng appropriate content_type
- Validate input messages
- Handle multimodal content properly

### 3. **Performance**
- Implement streaming cho long-running tasks
- Use appropriate timeouts
- Monitor agent health

### 4. **Security**
- Validate all inputs
- Implement authentication/authorization
- Secure communication channels

## Troubleshooting

### Common Issues

**1. Port already in use**
```bash
# Check running processes
lsof -i :8000
# Kill process if needed
kill -9 <PID>
```

**2. Import errors**
```bash
# Reinstall SDK
uv remove acp-sdk
uv add acp-sdk
```

**3. Connection timeouts**
```python
# Increase timeout in client
async with Client(base_url="http://localhost:8000", timeout=30) as client:
    # ...
```

## Tài nguyên và Community

### 📚 Documentation
- **Official Docs**: https://agentcommunicationprotocol.dev
- **API Reference**: OpenAPI specification included

### 💻 Code & Examples
- **Main Repository**: https://github.com/i-am-bee/acp
- **Examples**: https://github.com/i-am-bee/acp/tree/main/examples/python
- **Bridge Projects**: https://github.com/GongRzhe/ACP-MCP-Server

### 🎓 Learning Resources
- **DeepLearning.AI Course**: https://www.deeplearning.ai/short-courses/acp-agent-communication-protocol/
- **Hands-on Guide**: https://blog.dailydoseofds.com/p/a-hands-on-guide-to-agent-communication
- **Technical Articles**: Medium, InfoWorld, MarkTechPost

### 🤝 Community
- **Linux Foundation AI & Data**: https://lfaidata.foundation/projects/
- **GitHub Discussions**: Repository issues và discussions
- **Maintainers**: Xem MAINTAINERS.md trong repository

### 📝 Contributing
ACP là một dự án mã nguồn mở với open governance. Để đóng góp:
1. Fork repository
2. Tạo feature branch
3. Submit pull request
4. Tham gia discussions

### 🔄 Updates & Roadmap
- **High Availability Support**: Fault-tolerant setups
- **Enhanced Documentation**: Comprehensive guides
- **More SDK Languages**: Mở rộng hỗ trợ ngôn ngữ
- **Performance Optimizations**: Cải thiện hiệu suất

---

## Kết luận

Agent Communication Protocol (ACP) đại diện cho một bước tiến quan trọng trong việc chuẩn hóa giao tiếp giữa các AI agents. Với kiến trúc mở, hỗ trợ đa dạng và community mạnh mẽ, ACP hứa hẹn sẽ trở thành nền tảng quan trọng cho việc phát triển hệ thống multi-agent trong tương lai.

Bắt đầu với ACP ngay hôm nay để xây dựng các AI agents có khả năng tương tác và hợp tác hiệu quả!
