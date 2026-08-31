# Lab 04 — Weather Agent with Remote MCP Server

**Họ tên:** Đào Ngọc Duy  
**MSSV:** 2A202601780

A weather agent built with Google ADK that connects to an MCP server via Streamable HTTP transport.

## Architecture

```
┌─────────────────┐   Streamable HTTP    ┌─────────────────┐      REST       ┌─────────────────┐
│   ADK Agent     │ ──────────────────── │   MCP Server    │ ─────────────── │  WeatherAPI.com │
│  (mcp-client)   │   localhost:8085/mcp │  (mcp-server)   │                 │                 │
└─────────────────┘                      └─────────────────┘                 └─────────────────┘
```

## Tools

| Tool | Description |
|------|-------------|
| `get_current_weather(city)` | Get current weather conditions for a city |
| `get_forecast(city, days)` | Get weather forecast (1–3 days) |
| `health_check()` | Verify server is running |

## ADK làm gì trong Lab này?

ADK (Agent Development Kit) đóng vai trò **MCP Client** 
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  1. KẾT NỐI tới MCP Server qua Streamable HTTP                  │
│     StreamableHTTPConnectionParams(url="localhost:8085/mcp")    │
│                                                                 │
│  2. KHÁM PHÁ tools tự động (list_tools)                         │
│     McpToolset → tự hỏi server "anh có tool gì?"                │
│     → nhận về: get_current_weather, get_forecast, health_check  │
│                                                                 │
│  3. TRUYỀN tools cho LLM (Gemini)                               │
│     Agent(model="gemini-2.5-flash", tools=[weather_tools])      │
│     → Gemini biết nó có thể gọi 3 tools trên                    │
│                                                                 │
│  4. ĐIỀU PHỐI vòng lặp Function Calling                         │
│     User hỏi → Gemini chọn tool → ADK gọi MCP Server            │
│     → nhận kết quả → đưa lại cho Gemini tổng hợp                │
│                                                                 │
│  5. CUNG CẤP giao diện web (adk web)                            │
│     → http://localhost:8000 để chat với agent                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

So với bài 02 (viết client thủ công bằng `mcp.ClientSession`), ADK giúp bạn **không phải viết vòng lặp function calling thủ công** nữa. Toàn bộ luồng list_tools → model quyết định → call_tool → model tổng hợp được ADK xử lý tự động.

## Setup

The two keys are independent: `WEATHERAPI_KEY` stays only on the MCP server,
while `GOOGLE_API_KEY` stays only in the ADK client. Never commit either key.

### 1. MCP Server

```bash
cd mcp-server
uv sync

# Copy .env.example to .env, then set WEATHERAPI_KEY.
# On PowerShell: Copy-Item .env.example .env
# On macOS/Linux: cp .env.example .env

# Start the server (runs on port 8085 by default)
uv run python weather.py
```

The server will be available at `http://localhost:8085/mcp`.

### Optional: run the MCP server in Docker

The Docker build uses the committed `uv.lock` and excludes `.env`. Pass the
WeatherAPI key only at runtime:

```bash
cd mcp-server
docker build -t weather-mcp-server .
docker run --rm -p 8085:8085 --env-file .env weather-mcp-server
```

### 2. ADK Agent (Client)

```bash
cd mcp-client
uv sync

# Copy .env.example to .env and set GOOGLE_API_KEY.
# On PowerShell: Copy-Item .env.example .env
# On macOS/Linux: cp .env.example .env

# Start ADK web interface
uv run adk web
```

Open http://localhost:8000 in your browser, select `weather_agent`, and ask about the weather.

## Configuration

| Variable | Where | Description |
|----------|-------|-------------|
| `WEATHERAPI_KEY` | mcp-server | API key from weatherapi.com |
| `GOOGLE_API_KEY` | mcp-client/.env | Gemini API key |
| `PORT` | mcp-server (env) | Override server port (default: 8085) |
| `MCP_SERVER_URL` | mcp-client/.env | Optional MCP endpoint override |
| `GEMINI_MODEL` | mcp-client/.env | Optional model override (default: `gemini-2.5-flash`) |

## Verify before opening the UI

With the MCP server running, open a second terminal:

```bash
cd mcp-client
uv run python verify_setup.py
uv run python smoke_test.py
```

The check confirms that the client key, dependencies, agent package, and the
configured `MCP_SERVER_URL` are reachable. A `404`, `405`, or `406` response to its
HTTP `GET` is accepted because the MCP endpoint uses a different request shape.
`smoke_test.py` then uses the MCP protocol to discover all three tools and calls
`health_check`; it does not require either external API key.
