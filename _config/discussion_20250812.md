# 開發討論與規劃：整合外部 SSO 認證

**日期：** 2025-08-12

## 1. 目標

將外部 SSO（單一登入）認證流程整合至 MCP Gateway。使用者將能夠透過外部身份提供者（Identity Provider, IdP）登入，成功後，Gateway 會為使用者產生一個自己的 JWT，並將其作為一個安全的、僅限 HTTP 存取的 Cookie 儲存在瀏覽器中，用於後續所有 API 請求的身份驗證。

此設計基於 `sso_client/app.py` 中展示的客戶端邏輯，並將其更穩固地實作於 Gateway 中。

## 2. 實施計畫

### 第一步：新增 SSO 組態設定

在 Gateway 的設定中加入與外部 SSO 服務對接所需的參數。

-   **目標檔案**：`mcpgateway/config.py`
-   **變更內容**：在 `Settings` Pydantic 模型中新增 SSO 相關的組態欄位。
    -   `SSO_ENABLED: bool = False`
    -   `SSO_AUTHORIZATION_URL: Optional[str] = None`
    -   `SSO_TOKEN_URL: Optional[str] = None`
    -   `SSO_CLIENT_ID: Optional[str] = None`
    -   `SSO_CLIENT_SECRET: Optional[str] = None`
    -   `SSO_REDIRECT_URI: Optional[str] = "http://localhost:4444/sso/callback"`
    -   `SSO_SCOPES: str = "openid profile email"`

### 第二步：建立 SSO 專用路由

建立新的 API 端點來處理 SSO 登入流程。

-   **目標檔案**：建立新的 `mcpgateway/sso.py` 檔案，並在 `mcpgateway/main.py` 中引入其路由器。
-   **變更內容**：
    1.  **`/sso/login` (GET)**：
        -   SSO 流程的起點。
        -   建構 SSO 服務的授權 URL，並將使用者重定向過去。
        -   重定向之前，產生一個 `state` 參數並存入 session/cookie 中，以防止 CSRF 攻擊。
    2.  **`/sso/callback` (GET)**：
        -   SSO 服務成功認證後的回呼目標。
        -   接收來自 SSO 服務的 `code` 和 `state`。
        -   驗證 `state` 的一致性。
        -   向 SSO 服務的權杖端點發送後端請求，用 `code` 交換 `access_token` 和 `id_token`。
        -   驗證 `id_token`，並從中提取使用者資訊。
        -   為該使用者產生一個 Gateway 專屬的 JWT。
        -   將此 JWT 設置為一個安全的 `HttpOnly` Cookie。
        -   將使用者重定向到登入後的目標頁面（例如 `/admin`）。

### 第三步：更新現有的認證機制

Gateway 現有的認證邏輯需要能夠識別由 SSO 流程設定的 Cookie。

-   **目標檔案**：`mcpgateway/utils/verify_credentials.py`
-   **變更內容**：
    -   修改 `require_auth` 依賴項。
    -   檢查順序應為：
        1.  優先檢查請求標頭中的 `Authorization: Bearer <token>`。
        2.  如果標頭中沒有權杖，則檢查請求中是否有名為 `jwt_token` 的 Cookie。
        3.  如果兩者都不存在或無效，對於 API 客戶端返回 401 錯誤；對於瀏覽器使用者，則將其重定向到 `/sso/login`。

### 第四步：在管理介面新增 SSO 登入按鈕

在登入頁面提供一個入口以發起 SSO 流程。

-   **目標檔案**：`mcpgateway/templates/admin.html` (或相關的登入模板)
-   **變更內容**：
    -   在現有的使用者名稱/密碼表單旁邊，新增一個「使用 SSO 登入」的按鈕。
    -   此按鈕的連結指向 `/sso/login` 端點。

## 3. 測試策略

### 1. 單元測試 (Unit Tests)

-   **`mcpgateway/config.py`**：驗證 `Settings` 模型能正確載入所有 `SSO_*` 相關設定。
-   **`mcpgateway/sso.py`**：
    -   測試 `/sso/login` 路由是否能正確產生重定向 URL。
    -   使用 `httpx` 的 mock 功能模擬對 SSO 權杖端點的呼叫，測試 `/sso/callback` 路由的成功與失敗案例（如 `state` 不匹配、`id_token` 無效等）。
-   **`mcpgateway/utils/verify_credentials.py`**：測試 `require_auth` 依賴項是否能正確處理帶有 JWT Cookie 的請求。

### 2. 整合測試 (Integration Tests)

-   **測試目標**：使用一個模擬的 SSO 伺服器來測試完整的認證流程。
-   **測試設置**：
    1.  在測試環境中，啟動一個模擬的 SSO 伺服器（可使用 `sso_server/app.py` 作為基礎）。
    2.  將 Gateway 的測試組態指向此模擬 SSO 伺服器。
-   **測試流程** (使用 FastAPI 的 `TestClient`)：
    1.  向 Gateway 的受保護端點發送請求，斷言其被重定向到模擬 SSO 伺服器。
    2.  模擬使用者在 SSO 伺服器登入，並讓其重定向回 Gateway 的 `/sso/callback`。
    3.  斷言 Gateway 的後端有向模擬 SSO 伺服器的 `/token` 端點發起請求。
    4.  斷言 `/sso/callback` 的最終回應是一個重定向，並且在回應中設定了 `jwt_token` Cookie。
    5.  使用上一步取得的 Cookie，再次請求受保護的端點，斷言請求成功。

### 3. 端到端 / UI 測試 (End-to-End / UI Tests)

-   **測試目標**：在真實瀏覽器中驗證使用者操作流程。
-   **測試工具**：Playwright 或 Selenium。
-   **測試情境**：
    1.  同時啟動 MCP Gateway 和模擬的 SSO 伺服器。
    2.  使用自動化瀏覽器訪問 Gateway 的 `/admin` 頁面。
    3.  斷言瀏覽器被重定向到模擬 SSO 伺服器的登入頁。
    4.  在模擬的登入頁上，自動填寫使用者名稱和密碼並提交。
    5.  斷言瀏覽器被重定向回 Gateway，並最終顯示 `/admin` 的儀表板內容。
    6.  檢查瀏覽器的 Cookie，確認已成功設定 `jwt_token`。