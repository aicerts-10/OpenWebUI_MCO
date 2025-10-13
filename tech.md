
### **Technical Flow Diagram**

This diagram shows all the actors involved, including the external services.

```
+-----------------------------------------------------------------------------------------+
|                                  YOUR SELF-HOSTED SERVER                                |
|                                   (Docker Network)                                      |
|                                                                                         |
|  +---------------+      (OpenAPI)      +-------------+      (MCP over HTTP)     +--------------------+
|  |               | ------------------> |             | ----------------------> |                    |
|  |  Open WebUI   | <------------------ |  mcpo Proxy | <---------------------- | Klavis MCP Server  |
|  |    Server     |      (JSON Result)  |             |      (JSON Result)      | (e.g., Gmail)      |
|  +---------------+                     +-------------+                         +--------------------+
|         ^                                                                               |
|         |                                                                               | (Authenticates itself
|         |                                                                               |  using YOUR Klavis
|         |                                                                               |  Developer API Key)
+---------|-------------------------------------------------------------------------------|
          |                                                                               |
          | (User interacts via browser)                                                  v
          v
+-----------------+                                                               +---------------------+      +---------------------+
|                 | <--------------------- (Redirects) ------------------------> |                     |      |                     |
|  User's Browser |                                                             |  Klavis AI Backend  |----->|  Google OAuth & API |
|                 | ---------------------> (OAuth Flow) -----------------------> | (Manages User Tokens) |<-----|  (External Service) |
+-----------------+                                                             +---------------------+      +---------------------+
```



### **High-Level Architecture Overview**

First, let's establish the role of each component in your self-hosted environment:

*   **Open WebUI Server:** The user-facing application. It talks to the LLM and, when a tool is needed, sends a request to its configured tool server (`mcpo`).
*   **`mcpo` Proxy Server:** The translator. It receives standard OpenAPI requests from Open WebUI and converts them into MCP-compatible requests for the Klavis server. Its job is to make the MCP server look like a standard web API.
*   **Klavis MCP Server (e.g., Gmail):** The tool connector. It knows how to perform specific actions (like `read_email`). It is responsible for handling authentication for its specific service by communicating with the Klavis AI Backend.

These three containers run on your server and communicate over a private Docker network.

---

---

### **Sequence 1: The First-Time User Authentication (OAuth Flow)**

This is the most complex sequence. It happens only once per user per service.

**Goal:** Securely get the user's permission for the tool to access their Gmail data.

1.  **User Action:** A user in Open WebUI runs a prompt requiring Gmail for the first time (e.g., "summarize my latest emails").
2.  **Request to Tool:** Open WebUI sends the tool call request to the `mcpo` proxy (`http://mcpo-proxy:8000`).
3.  **Translation:** `mcpo` translates the request and forwards it to the Klavis `gmail-mcp-server` (`http://gmail-mcp-server:5000/mcp/`).
4.  **Auth Check:** The `gmail-mcp-server` sees that it has no OAuth token for this specific user. It cannot proceed.
5.  **Request Auth URL:** It makes a backend API call to the **Klavis AI Backend**.
    *   **Payload:** "I need an authentication URL for `user_id: 'some_unique_user_id'`."
    *   **Authentication:** This request is authenticated using the **`KLAVIS_API_KEY`** you provided in the Docker environment variables. This proves to Klavis that the request is coming from your legitimate server.
6.  **Receive Auth URL:** The Klavis AI Backend generates a unique Google OAuth URL and sends it back to your `gmail-mcp-server`.
7.  **Return URL to User:** The URL is passed all the way back through `mcpo` and Open WebUI to the user's browser, typically as a button that says "Connect to Gmail".
8.  **User Redirect to Google:** The user clicks the button. Their browser is redirected to `accounts.google.com`.
9.  **User Consent:** The user logs into Google (if not already) and grants permission on the consent screen ("Klavis AI wants to access...").
10. **Redirect to Klavis:** Google redirects the user's browser to Klavis's pre-configured callback URL (`https://api.klavis.ai/oauth/gmail/callback`) and includes a temporary `authorization_code`.
11. **Token Exchange:** The **Klavis AI Backend** receives the `authorization_code`. It immediately makes a secure, server-to-server call to Google, exchanging the code for a long-lived `refresh_token` and a short-lived `access_token`.
12. **Secure Storage:** Klavis encrypts and securely stores these tokens, linking them to the user's ID from Step 5. **Your server never sees or stores these user tokens.**
13. **Success:** Klavis redirects the user's browser back to your Open WebUI instance. The connection is complete. The original request can now be re-tried.

### **Sequence 2: Subsequent Tool Use (Authenticated API Call Flow)**

This flow is much simpler and happens every time a connected user uses the tool.

**Goal:** Use the stored credentials to fetch data from the Gmail API.

1.  **User Action:** The user runs another prompt: "Send an email to bob@example.com with the subject 'Hello'".
2.  **Request to Tool:** Open WebUI -> `mcpo` -> `gmail-mcp-server`.
3.  **Request Access Token:** The `gmail-mcp-server` knows it needs to call the Gmail API. It makes a backend API call to the **Klavis AI Backend**.
    *   **Payload:** "Give me the `access_token` for `user_id: 'some_unique_user_id'`."
    *   **Authentication:** Again, this request is authenticated using your **`KLAVIS_API_KEY`**.
4.  **Receive Access Token:** The Klavis AI Backend retrieves the stored, valid `access_token` for that user and sends it back to your `gmail-mcp-server`. (If the token was expired, Klavis would first use the `refresh_token` to get a new one from Google automatically before responding).
5.  **Call Google API:** Your `gmail-mcp-server` now has the user's temporary key. It makes a direct API call to the **Google Gmail API** (`https://gmail.googleapis.com/...`).
    *   **Authentication:** This call is authenticated using the user's `access_token` in the `Authorization: Bearer <token>` header.
6.  **Receive Data:** The Google API validates the token, executes the request (sends the email), and returns a success response.
7.  **Return Result:** The `gmail-mcp-server` sends the success message back through `mcpo` to Open WebUI.
8.  **Display to User:** Open WebUI shows the user the final result: "OK, I've sent the email."


### FAQS:

#### **1. How will OAuth work for the user, and will Klavis provide it?**

**Yes, Klavis provides and manages the entire OAuth process.**

Here’s how it works for the end-user:
*   **Initiation:** When a user in Open WebUI tries to use a tool for the first time (e.g., "read my emails"), the Klavis MCP server recognizes that it needs permission.
*   **Redirect:** The user is then redirected to the standard, secure login page for that service (e.g., Google's login page).
*   **Consent:** After logging in, the user sees a consent screen that says "Klavis AI wants permission to..." and lists the required access (e.g., read and send emails).
*   **Authorization:** Once the user clicks "Allow," the service sends an authorization code back to Klavis's secure servers.
*   **Token Management:** Klavis exchanges this code for an `access token` and a `refresh token`, which it then encrypts and manages securely. Open WebUI never sees or stores these user credentials.

The beauty of this system is that Klavis handles all the complexity. You simply run their server with your developer API key, and they manage the per-user authentications.

#### **2. Do we need to set up OAuth?**

**No, you do not need to set up your own OAuth application to get started.**

By simply providing your Klavis API key when you run the `gmail-mcp-server` container, you are instructing it to use Klavis's pre-configured, shared OAuth application. The consent screen will show the "Klavis AI" branding to your users.

You only need to set up your own OAuth application if you want to **white-label** the experience—that is, replace the "Klavis AI" branding with your own app's name and logo on the consent screen. The documentation provides the steps for this optional customization.


#### ** The Admin's Setup Journey**

1.  **Get Klavis API Key:** The admin signs up on the Klavis AI website and gets a developer API key.
2.  **Create a Docker Network:** The admin creates a dedicated network so the services can communicate.
    ```bash
    docker network create open-webui-tools
    docker network connect open-webui-tools your-open-webui-container-name
    ```
3.  **Launch Gmail Server:** The admin runs the Klavis Gmail MCP server, connecting it to the network and providing the API key.
    ```bash
    docker run -d --name gmail-mcp-server --network open-webui-tools -e KLAVIS_API_KEY=your_klavis_api_key --restart always ghcr.io/klavis-ai/gmail-mcp-server:latest
    ```
4.  **Launch the `mcpo` Proxy:** The admin runs the `mcpo` proxy to make the Gmail server compatible with Open WebUI. It's also connected to the same network.
    ```bash
    docker run -d -p 8000:8000 --name mcpo-proxy --network open-webui-tools --restart always ghcr.io/open-webui/mcpo:main --host 0.0.0.0 --port 8000 --server-type "streamable-http" -- http://gmail-mcp-server:5000/mcp/
    ```
5.  **Configure Open WebUI:** The admin logs into Open WebUI, navigates to **Admin Settings > Tools**, and adds the `mcpo` proxy as a new tool server.

    *   **What URL to put?** The URL must be accessible from the Open WebUI backend. Since both containers are on the same Docker network, you can use the container's name as the hostname.
    *   **URL:** `http://mcpo-proxy:8000`

    The setup is now complete.

