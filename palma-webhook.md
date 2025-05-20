# Webhook and Real-time Functionalities

This document outlines the webhook and real-time communication systems implemented in the PalmaBot project.

## 1. PalmaBot Executor Webhook (`/xc`)

-   **File:** `palma/palma.py`
-   **Endpoint:** `{SERVER_NAME}/xc`
-   **Purpose:** This webhook is designed to connect external strategies or systems to the PalmaBot trading executor.
-   **Functionality:**
    -   When a request hits this endpoint, the system generates or retrieves a unique `xc_code` for the user (chat ID).
    -   It expects a connection string in the request, which should include:
        -   `code`: The unique user code.
        -   `strategy_id`: An identifier for the trading strategy.
        -   `demo`: (YES/NO) Specifies if the execution should be in demo mode.
        -   `command`: The trading command (e.g., `sell`, `buy`).
        -   `exchange`: The target exchange (e.g., `binance`).
        -   `coinmarket`: The coin and market pair (e.g., `BTCUSDT`).
        -   `amount`: The quantity for the trade.
    -   The system logs the incoming request.
    -   It then sends a message to the user via Telegram, providing the full connection string they need to use and a link to a tutorial on how to connect.
-   **Usage Example (from code):**
    `code={xc_code} strategy_id=MyStrategy demo=YES command=sell exchange=binance coinmarket=BTCUSDT amount=0.05`
    Webhook URL: `{SERVER_NAME}/xc`

## 2. Viber Integration Webhook

-   **File:** `palma/palma_whatsapp.py` (primary implementation)
-   **File (Legacy/Reference):** `palma/palma_social_channels.py` (contains commented-out Viber webhook setup)
-   **Endpoint:** `https://realtime.palmabot.com/viber`
-   **Purpose:** Enables PalmaBot to receive and send messages via the Viber platform.
-   **Functionality:**
    -   The `myViber` class in `palma/palma_whatsapp.py` configures the Viber bot with an authentication token and sets up a webhook to `https://realtime.palmabot.com/viber`.
    -   This allows PalmaBot to listen for events (like incoming messages) from Viber users.
    -   The `text_message` method is used to format outgoing text messages to be sent via Viber.

## 3. Trading Dashboard WebSocket

-   **File:** `palma/palma_trading_dashboard.py`
-   **Purpose:** Provides real-time communication capabilities for the Palma Trading Dashboard.
-   **Functionality:**
    -   A WebSocket server is implemented using the `websockets` library.
    -   It listens on a host and port defined in the configuration (`WEBSOCKET_HOST`, `WEBSOCKET_PORT_PTD`).
    -   **User Connection Management:**
        -   Users connect to the WebSocket, typically identified by a `user_uuid`.
        -   Supports `prijava` (register/login) and `odjava` (unregister/logout) events from clients.
        -   Maintains a record of connected users (`CONNECTED_USERS`).
    -   **Message Handling:**
        -   Receives messages from connected clients (JSON format).
        -   A `message_consumer_task` likely handles processing messages from a queue (e.g., Redis Pub/Sub via `pubsub.start_consumer_async`) to push updates to the relevant connected clients.
        -   This allows the trading dashboard to display real-time updates for orders, market data, or other relevant trading information.
    -   This is not a traditional HTTP webhook but achieves real-time, bidirectional communication between the server and the trading dashboard clients.

## Security Considerations for Webhooks

-   **Authentication/Authorization:**
    -   The `/xc` webhook relies on a generated `code` for identifying the user. Ensure this code is securely generated and managed.
    -   Viber webhooks use an `auth_token` for the bot configuration.
-   **Input Validation:**
    -   All incoming webhook data (e.g., parameters in the `/xc` connection string, messages from Viber) should be strictly validated to prevent injection attacks or unexpected behavior.
-   **HTTPS:**
    -   All webhook endpoints (e.g., `https://realtime.palmabot.com/viber`, `{SERVER_NAME}/xc`) must use HTTPS to protect data in transit.
-   **Rate Limiting:**
    -   Consider implementing rate limiting on webhook endpoints to prevent abuse.
-   **Logging and Monitoring:**
    -   Maintain comprehensive logs for all webhook requests and responses for auditing and troubleshooting. The current implementation shows logging for the `/xc` endpoint.

## Sequence Diagrams

### 1. PalmaBot Executor Webhook (`/xc`)

```mermaid
sequenceDiagram
    participant ExternalSystem as External System
    participant PalmaBotServer as PalmaBot Server (`palma.py`)
    participant Database as Database
    participant TelegramAPI as Telegram API

    ExternalSystem->>PalmaBotServer: POST {SERVER_NAME}/xc with connection string
    PalmaBotServer->>Database: Get or create xc_code for user (chat_id)
    Database-->>PalmaBotServer: xc_code
    PalmaBotServer->>Database: Log incoming request
    PalmaBotServer->>TelegramAPI: Send message to user (chat_id) with full connection string and tutorial link
    TelegramAPI-->>PalmaBotServer: Message sent confirmation
    PalmaBotServer-->>ExternalSystem: HTTP 200 OK (or appropriate response)
```

### 2. Viber Integration Webhook

```mermaid
sequenceDiagram
    participant ViberUser as Viber User
    participant ViberPlatform as Viber Platform
    participant PalmaBotServer as PalmaBot Server (`palma_whatsapp.py`)
    participant PalmaLogic as PalmaBot Core Logic

    Note over PalmaBotServer: Webhook `https://realtime.palmabot.com/viber` is set during initialization.

    ViberUser->>ViberPlatform: Sends a message
    ViberPlatform->>PalmaBotServer: HTTP POST to webhook with message data
    activate PalmaBotServer
    PalmaBotServer->>PalmaLogic: Process incoming Viber message
    PalmaLogic-->>PalmaBotServer: Response message (if any)
    PalmaBotServer->>ViberPlatform: Send reply message using Viber API
    ViberPlatform-->>ViberUser: Delivers reply message
    deactivate PalmaBotServer
    PalmaBotServer-->>ViberPlatform: HTTP 200 OK
```

### 3. Trading Dashboard WebSocket

#### Connection and Registration:
```mermaid
sequenceDiagram
    participant DashboardClient as Trading Dashboard Client
    participant PalmaWebSocketServer as Palma WebSocket Server (`palma_trading_dashboard.py`)

    DashboardClient->>PalmaWebSocketServer: WebSocket Connection Request
    PalmaWebSocketServer-->>DashboardClient: WebSocket Connection Established
    DashboardClient->>PalmaWebSocketServer: Send JSON message: `{"event": "prijava", "user_uuid": "USER_UUID_HERE"}`
    activate PalmaWebSocketServer
    PalmaWebSocketServer->>PalmaWebSocketServer: register_user_connection(websocket, USER_UUID_HERE)
    PalmaWebSocketServer->>PalmaWebSocketServer: Store user_uuid and websocket connection
    PalmaWebSocketServer-->>DashboardClient: Send JSON message: `{"status": "registered", "uuid": "USER_UUID_HERE"}`
    deactivate PalmaWebSocketServer
```

#### Real-time Updates to Dashboard:
```mermaid
sequenceDiagram
    participant PalmaLogic as PalmaBot Core Logic (e.g., order execution, market data update)
    participant MessageQueue as Message Queue (e.g., Redis Pub/Sub)
    participant PalmaWebSocketServer as Palma WebSocket Server (`palma_trading_dashboard.py`)
    participant DashboardClient as Trading Dashboard Client (connected with USER_UUID_HERE)

    PalmaLogic->>MessageQueue: Publish update (e.g., new order status for USER_UUID_HERE)
    activate PalmaWebSocketServer
    PalmaWebSocketServer->>MessageQueue: Consume message (message_consumer_task)
    MessageQueue-->>PalmaWebSocketServer: Update data for USER_UUID_HERE
    PalmaWebSocketServer->>PalmaWebSocketServer: Identify relevant WebSocket connection for USER_UUID_HERE
    PalmaWebSocketServer-->>DashboardClient: Push JSON update message over WebSocket
    deactivate PalmaWebSocketServer
    activate DashboardClient
    DashboardClient->>DashboardClient: Update UI with new data
    deactivate DashboardClient
```

## Future Enhancements/Documentation

-   Specific error codes and responses for the `/xc` webhook.
-   More information on the types of messages and data exchanged over the Trading Dashboard WebSocket.
