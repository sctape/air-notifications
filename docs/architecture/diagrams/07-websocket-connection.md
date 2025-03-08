# WebSocket Connection and Subscription Flow

This diagram details the process of establishing a WebSocket connection and subscribing to notification channels for real-time updates.

```mermaid
sequenceDiagram
    actor User
    participant WebApp as Air Web App
    participant API as REST API
    participant Auth as Auth Service
    participant WebSocket as WebSocket Service
    participant Channels as Channel Manager
    participant NotifDB as Notification DB
    
    %% Authentication and Connection
    User->>WebApp: Log in/Open application
    WebApp->>API: Authentication request
    API->>Auth: Validate credentials
    Auth-->>API: Generate session token
    API-->>WebApp: Return token in response
    
    %% WebSocket Connection
    WebApp->>WebSocket: Connect (with auth token)
    WebSocket->>Auth: Verify token
    Auth-->>WebSocket: Token verification result
    
    alt Invalid Token
        WebSocket-->>WebApp: Connection rejected
    else Valid Token
        WebSocket-->>WebApp: Connection established
        
        %% Channel Subscription
        WebApp->>WebSocket: Subscribe to user's notification channel
        WebSocket->>Channels: Register client to channel
        Channels-->>WebSocket: Subscription confirmed
        WebSocket-->>WebApp: Subscription acknowledgment
        
        %% Notification Delivery
        Note over NotifDB,WebApp: Real-time notification flow
        NotifDB->>Channels: New BatchedNotification created
        Channels->>WebSocket: Notify subscribed clients
        WebSocket->>WebApp: Push BatchedNotification data
        WebApp->>User: Display notification
        
        %% Connection Maintenance
        Note over WebApp,WebSocket: Connection maintenance
        WebApp->>WebSocket: Ping (heartbeat)
        WebSocket-->>WebApp: Pong (acknowledgment)
        
        %% Disconnection
        User->>WebApp: Close app/Log out
        WebApp->>WebSocket: Close connection
        WebSocket->>Channels: Unregister from channels
    end
    
    %% Set participant styling
    note over User: End User
    note over WebApp: Client Application
    note over API: REST API Server
    note over Auth: Authentication Service
    note over WebSocket: WebSocket Server
    note over Channels: Channel Management
    note over NotifDB: Notification Database
```

## WebSocket Connection Lifecycle

### 1. Authentication and Token Acquisition
The client first obtains an authentication token through the standard REST API:
- User logs in or opens the application
- Client requests authentication from the API
- API validates credentials and returns a session token
- This token will be used to authenticate the WebSocket connection

### 2. WebSocket Connection Establishment
The client establishes a WebSocket connection:
- Client initiates WebSocket connection with the auth token
- WebSocket server verifies the token with Auth Service
- If valid, connection is established; if invalid, connection is rejected

### 3. Channel Subscription
After connection is established, the client subscribes to relevant notification channels:
- Client requests subscription to user's notification channel
- WebSocket server registers client with Channel Manager
- Subscription confirmation is sent back to client

### 4. Real-time Notification Delivery
Once subscribed, the client receives BatchedNotifications in real time:
- When a new BatchedNotification is created or updated in the Notification DB
- Channel Manager identifies subscribed clients
- WebSocket server pushes BatchedNotification data to relevant clients
- Client displays the notification without polling

### 5. Connection Maintenance
To keep the connection alive:
- Client sends periodic ping messages (heartbeats)
- WebSocket server responds with pong acknowledgments
- This prevents connection timeouts and confirms both sides are active

### 6. Disconnection
When the user closes the app or logs out:
- Client initiates connection close
- WebSocket server unregisters client from notification channels
- Resources are released on both ends

## Implementation Considerations

- **Scalability**: Consider using Redis Pub/Sub or similar technology for channel management in distributed environments
- **Reconnection Strategy**: Implement exponential backoff for reconnection attempts when connection is lost
- **Message Delivery Guarantees**: Consider using acknowledgments for critical notifications
- **Security**: Implement token expiration, rotation, and validation to prevent unauthorized access
- **Fallback**: Provide REST API polling as a fallback for environments where WebSockets are blocked 