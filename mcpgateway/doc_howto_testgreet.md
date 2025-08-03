# 如何註冊與測試遠端 MCP 工具

本文件將引導您完成將一個遠端的 MCP 工具（在此範例中為 `greet` 工具）註冊到 ContextForge Gateway，並成功呼叫它的完整流程。

我們將採用最佳實踐，將遠端工具伺服器註冊為一個「Gateway」，讓 ContextForge 自動發現其提供的所有工具。

## 先決條件

在開始之前，請確保您已完成以下設定：

1.  **ContextForge Gateway 已啟動**：
    *   您已經依照 `README.md` 的指示，透過 `make dev` 或 `make serve` 啟動了主閘道。
    *   我們假設主閘道運行在 `http://localhost:4444`。

2.  **遠端 MCP 工具伺服器已準備好**：
    *   您有一個正在運行的 MCP 伺服器。為了本教學，我們將使用以下簡單的 `server.py` 範例。

### 範例 `server.py` (greet 工具)

在您的專案目錄外建立一個新資料夾（例如 `my-mcp-tool`），並將以下內容儲存為 `server.py`：

```python
# my-mcp-tool/server.py
import os
from mcp.server.fastmcp import FastMCP
from pydantic import BaseModel, Field

# 建議使用 Pydantic 模型來定義輸入結構
class GreetInput(BaseModel):
    name: str = Field(
        ...,
        description="The name of the person to greet.",
        examples=["World", "Alice"]
    )

mcp = FastMCP(name="MyGreetServer")

@mcp.tool()
def greet(data: GreetInput) -> str:
    """Greets a user by their name."""
    return f"Hello, {data.name}!"

if __name__ == "__main__":
    # FastMCP 會自動從環境變數或 .env 檔案載入設定
    # 預設 transport 為 streamable-http，運行在 8000 port
    transport_type = os.getenv("FASTMCP_TRANSPORT", "streamable-http")
    host = mcp.settings.host
    port = mcp.settings.port
    
    print(f"--> Starting greet tool server on http://{host}:{port} using {transport_type}...")
    mcp.run(transport=transport_type)
```

## 步驟 1：啟動所有服務

您需要同時運行 ContextForge Gateway 和您的 `greet` 工具伺服器。

1.  **啟動 ContextForge Gateway** (在 `mcp-context-forge` 專案目錄中):
    ```bash
    # 建議使用開發模式，日誌更詳細
    make dev
    ```
    此服務將運行在 `http://localhost:8000` (uvicorn) 或 `http://localhost:4444` (gunicorn)。本文件將使用 `http://localhost:4444`。

2.  **啟動 `greet` 工具伺服器** (在 `my-mcp-tool` 目錄中):
    ```bash
    # 安裝必要的套件
    pip install "fastmcp[standard]" pydantic

    # 啟動伺服器
    python server.py
    ```
    此服務將運行在 `http://localhost:8000`。

## 步驟 2：取得認證權杖 (Token)

ContextForge 的 API 端點受 JWT 保護。您需要先產生一個 Bearer Token。

(在 `mcp-context-forge` 專案目錄中執行)

```bash
# 從 .env.example 複製一份 .env 檔案 (如果尚未建立)
cp -n .env.example .env

# 讀取 .env 中的 JWT_SECRET_KEY 並產生一個 token
# 這個 token 預設有效期為 7 天
export MCPGATEWAY_BEARER_TOKEN=$(python3 -m mcpgateway.utils.create_jwt_token \
    --username admin \
    --secret $(grep JWT_SECRET_KEY .env | cut -d '=' -f2))

# 驗證 token 是否已設定
echo "Your token is: $MCPGATEWAY_BEARER_TOKEN"
```

## 步驟 3：將工具伺服器註冊為一個 Gateway

現在，我們將 `greet` 工具所在的伺服器 (`http://localhost:8000/mcp`) 註冊到 ContextForge。

```bash
curl -X POST -H "Authorization: Bearer $MCPGATEWAY_BEARER_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{
           "name": "my-greet-server",
           "url": "http://localhost:8000/mcp",
           "description": "A server that provides the greet tool",
           "transport": "STREAMABLEHTTP"
         }' \
     http://localhost:4444/gateways | jq
```

**參數說明**：
*   `name`: 給您的工具伺服器一個在 ContextForge 中唯一的名稱，例如 `my-greet-server`。
*   `url`: 您的 MCP 伺服器的完整端點 URL。
*   `transport`: 您的 MCP 伺服器使用的傳輸協定。`fastmcp` 預設使用 `streamable-http`，對應到 ContextForge 的 `STREAMABLEHTTP`。

如果成功，您會看到一個包含新建立的 Gateway 資訊的 JSON 回應。

## 步驟 4：驗證工具是否被自動發現

稍等幾秒鐘，讓 ContextForge 完成同步。然後，列出所有可用的工具來確認 `greet` 工具是否已被發現。

```bash
curl -s -H "Authorization: Bearer $MCPGATEWAY_BEARER_TOKEN" http://localhost:4444/tools | jq
```

您應該會看到類似以下的輸出。請特別注意 `name` 欄位，它現在包含了 Gateway 的名稱作為前綴。

```json
[
  {
    "id": "...",
    "originalName": "greet",
    "url": "http://localhost:8000/mcp",
    "description": "Greets a user by their name.",
    "requestType": "STREAMABLEHTTP",
    "integrationType": "MCP",
    "inputSchema": {
      "title": "GreetInput",
      "type": "object",
      "properties": {
        "name": {
          "description": "The name of the person to greet.",
          "examples": [
            "World",
            "Alice"
          ],
          "title": "Name",
          "type": "string"
        }
      },
      "required": [
        "name"
      ]
    },
    "name": "my-greet-server-greet",
    "gatewayId": "...",
    "gatewaySlug": "my-greet-server",
    "originalNameSlug": "greet"
    // ... 其他欄位 ...
  }
]
```

`"name": "my-greet-server-greet"` 是我們下一步呼叫工具時需要使用的名稱。

## 步驟 5：透過 RPC 呼叫工具

現在，您可以使用工具的完整名稱，透過 ContextForge 的 `/rpc` 端點來呼叫它。

```bash
curl -X POST -H "Authorization: Bearer $MCPGATEWAY_BEARER_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{
           "jsonrpc": "2.0",
           "id": "greet-test-1",
           "method": "my-greet-server-greet",
           "params": {
             "name": "ContextForge"
           }
         }' \
     http://localhost:4444/rpc | jq
```

**請求內容解析**：
*   `method`: 必須是上一步驟中看到的、帶有命名空間的工具名稱 (`my-greet-server-greet`)。
*   `params`: 一個 JSON 物件，其鍵 (`name`) 和值 (`ContextForge`) 對應到您 `server.py` 中 `GreetInput` 模型定義的參數。

### 預期成功的回應

如果一切順利，您將會收到來自 `greet` 工具的回應，並由 ContextForge 封裝成 JSON-RPC 格式：

```json
{
  "jsonrpc": "2.0",
  "id": "greet-test-1",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Hello, ContextForge!"
      }
    ],
    "isError": false
  }
}
```

恭喜！您已成功註冊並透過 ContextForge 呼叫了一個遠端 MCP 工具。

## (選用) 步驟 6: 建立虛擬伺服器

您可以將一組工具打包成一個「虛擬伺服器」，提供給客戶端一個乾淨的、獨立的端點。

1.  **取得工具的 ID**
    從 `步驟 4` 的輸出中找到 `greet` 工具的 `id`。

2.  **建立虛擬伺服器**
    將工具 ID 列表傳遞給 `/servers` 端點。

    ```bash
    # 將 YOUR_TOOL_ID 替換為實際的工具 ID
    TOOL_ID="YOUR_TOOL_ID"

    curl -X POST -H "Authorization: Bearer $MCPGATEWAY_BEARER_TOKEN" \
         -H "Content-Type: application/json" \
         -d '{
               "name": "MyVirtualServer",
               "description": "A server for greeting people.",
               "associated_tools": ["'"$TOOL_ID"'"]
             }' \
         http://localhost:4444/servers | jq
    ```

3.  **使用虛擬伺服器**
    現在，MCP 客戶端可以直接連接到這個虛擬伺服器的端點，它只會看到您指定的工具。

    *   **SSE URL**: `http://localhost:4444/servers/VIRTUAL_SERVER_ID/sse`
    *   **RPC URL**: `http://localhost:4444/servers/VIRTUAL_SERVER_ID/rpc`

這使得客戶端的設定更加簡單和安全。