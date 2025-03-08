# High-Level Architecture Diagram

This diagram shows the overall architecture of the Air Unified Notification System, highlighting the key components and their interactions.

```mermaid
flowchart TD
    %% Core Components
    RestAPI[Air REST API Server] -->|Events| Publisher[Event Publisher]
    Publisher -->|NotificationEvents| Queue[Message Queue<br>e.g., Kafka/SQS]
    Queue --> Processor[Event Processor & Batcher]
    
    %% Data Flow and Databases
    Processor -->|Reads| UserPrefDB[(User Preferences DB)]
    Processor -->|Creates NotificationEvents| NotifDB[(Notification DB)]
    Processor -->|Creates BatchedNotifications| NotifDB
    
    %% Notification Service and API
    Processor --> NotifService[Notification Service]
    NotifService <-->|REST API<br>/api/notifications| WebApp[Client Web App]
    NotifService <-->|Reads/Updates<br>BatchedNotifications| NotifDB
    NotifService <-->|Reads<br>NotificationPreferences| UserPrefDB
    
    %% WebSocket for Real-time Notifications
    NotifDB -->|New BatchedNotifications| WebSocket[WebSocket Service]
    WebSocket -->|Real-time Updates| WebApp
    
    %% Email Delivery
    Processor --> EmailService[Email Delivery Service]
    EmailService -->|Updates EmailDelivery records| NotifDB
    EmailService -->|API Integration| ThirdParty[3rd Party Email Provider<br>e.g., SendGrid, SES]
    ThirdParty -->|Delivery Status Callbacks| EmailService
    
    %% Styling
    classDef primary fill:#b3d9ff,stroke:#0066cc,stroke-width:2px,color:#333
    classDef storage fill:#ffe6cc,stroke:#ff9933,stroke-width:2px,color:#333
    classDef client fill:#d9f2d9,stroke:#339933,stroke-width:2px,color:#333
    classDef thirdparty fill:#e6ccff,stroke:#9966cc,stroke-width:2px,color:#333
    classDef realtime fill:#d1f0e0,stroke:#2eb886,stroke-width:2px,color:#333
    
    class RestAPI,Publisher,Processor,NotifService,EmailService primary
    class Queue,NotifDB,UserPrefDB storage
    class WebApp client
    class ThirdParty thirdparty
    class WebSocket realtime
```

## Key Components

- **Air REST API Server**: The main application API that generates notification events based on user actions
- **Event Publisher**: Responsible for formatting and publishing notification events
- **Message Queue**: Provides asynchronous communication between components (e.g., Kafka, AWS SQS)
- **Event Processor & Batcher**: Processes events, applies batching rules, and determines recipients for notifications
- **Notification Service**: Manages notification delivery and REST API endpoints for client applications
- **WebSocket Service**: Handles real-time notification delivery to connected clients
- **Notification DB**: Stores notification data including NotificationEvents, BatchedNotifications, and EmailDelivery records
- **User Preferences DB**: Stores user NotificationPreferences including delivery channel settings (inAppEnabled, emailEnabled)
- **Email Delivery Service**: Handles email template rendering and communicates with third-party providers
- **3rd Party Email Provider**: External service for sending emails (e.g., Amazon SES, SendGrid)

## Communication Flows

1. **Event Generation**: User actions in the app generate NotificationEvents through the REST API, which are published to the message queue via the Event Publisher.

2. **Event Processing**: The Event Processor consumes NotificationEvents from the queue, checks user NotificationPreferences, applies batching logic, and creates or updates BatchedNotifications.

3. **Notification Delivery**:
   - **In-App**: New BatchedNotifications in the Notification DB trigger the WebSocket Service to push real-time updates to connected clients
   - **Email**: Processor requests Email Delivery Service to send emails via 3rd Party Provider, with EmailDelivery status tracked in the database

4. **API Access**: 
   - Clients interact with the Notification Service via REST API endpoints (e.g., `/api/notifications`, `/api/notifications/preferences`) to access historical notifications and manage preferences
   - Notification Service handles all database interactions, providing a clean API abstraction for the client

This architecture ensures separation of concerns, scalability, and reliability across the notification system. 