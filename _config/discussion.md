# 💡 Development Discussion & Planning

This document captures design discussions, architectural decisions, and implementation plans for new features.

---

## Dynamic Authorization & User Context Forwarding

**Date:** 2024-07-31

### 1. Objective

To enhance the MCP Gateway's capabilities by allowing it to dynamically inject authentication credentials (specifically, a Bearer Token) and user-specific information into the headers of outgoing requests made to downstream tools.

This feature will enable more sophisticated integration patterns where tools can securely identify and act on behalf of the end-user who initiated the request through the gateway.

### 2. Proposed Implementation Plan

The implementation will follow these key steps:

#### Step 1: Extend the Tool Data Model

We will introduce a new configuration field to the `Tool` schema to specify which pieces of user information should be forwarded.

-   **File to Modify:** `mcpgateway/schemas.py`
-   **Schema to Update:** `ToolUpdate` and `ToolCreate`
-   **New Field:** `forward_claims_as_headers: Optional[List[str]] = None`
-   **Description:** This field will hold a list of JWT claims to be extracted from the incoming request's token and forwarded as headers to the tool's endpoint.
-   **Special Value:** We will define a special keyword, `@token`, which, when included in the list, will instruct the gateway to forward the original, unmodified JWT Bearer token in the `Authorization` header.

This change will also require adding a corresponding `JSON` type column to the `Tool` database model to persist this setting.

#### Step 2: Pass User Context Through the Service Layer

The logic that handles tool invocation must be made aware of the current user's context.

-   The primary tool invocation service (e.g., `ToolService.invoke_tool`) will be updated to accept the `current_user` object as an argument. This object, containing the decoded JWT claims, will be supplied by FastAPI's dependency injection system at the API endpoint level.
-   This `user_claims` dictionary will then be passed down to the HTTP client responsible for making the external request.

#### Step 3: Modify the HTTP Client for Dynamic Header Injection

This is the core of the implementation, where the dynamic headers are constructed and added to the outgoing request.

-   The HTTP client (e.g., a `ResilientHttpClient` class) will be modified.
-   **Logic Flow:**
    1.  Prepare the standard request headers, including any statically configured authentication for the tool.
    2.  Check if the `tool.forward_claims_as_headers` list is configured and if `user_claims` are available.
    3.  Iterate through the `forward_claims_as_headers` list:
        -   If an item is `'@token'`, add or overwrite the `Authorization` header with `Bearer <raw_jwt_from_user_claims>`.
        -   For any other item (e.g., `'sub'`, `'email'`), retrieve the corresponding value from `user_claims` and add it as a new header, prefixed for clarity (e.g., `X-Forwarded-User-Sub: <claim_value>`).
    4.  Send the request with the combined static and dynamic headers.

This approach ensures that dynamic, user-specific authentication can gracefully override any static credentials configured on the tool, providing maximum flexibility.

#### Step 4: Update the Admin UI

To make this feature configurable by administrators, we will update the Admin UI.

-   **File to Modify:** The Jinja2 template for the "Add/Edit Tool" page.
-   **Change:** Add a new text input or textarea field corresponding to the `forward_claims_as_headers` attribute.
-   **User Experience:** Administrators can provide a comma-separated string of claim names (e.g., `@token, sub, email`).
-   **Backend Logic:** The corresponding admin route (e.g., `admin_edit_tool` in `mcpgateway/admin.py`) will parse this comma-separated string into a list before passing it to the service layer for storage.

### 3. Impact & Benefits

-   **Enhanced Security:** Tools can receive and validate the end-user's JWT, enabling per-user authorization and auditing.
-   **Increased Flexibility:** The gateway can now support tools that require dynamic, per-request authentication without hardcoding credentials.
-   **Seamless Integration:** User context (like user ID, email, or roles) can be passed transparently to downstream services, enabling personalization and context-aware tool behavior.
-   **Configuration-Driven:** The feature is entirely opt-in and configured on a per-tool basis, ensuring no impact on existing tool integrations.