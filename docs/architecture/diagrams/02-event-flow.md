# Event Flow Diagram

This diagram illustrates how notification events flow through the system, from generation to delivery.

```mermaid
flowchart TD
    subgraph EventSources[Event Sources]
        CRUD[CRUD Events<br>Asset Added, Comments]
        Action[Action Events<br>Download, Share Viewed]
        Time[Time Events<br>Share Not Viewed in 5d]
        Other[Other Events]
    end
    
    CRUD --> Publisher
    Action --> Publisher
    Time --> Publisher
    Other --> Publisher
    
    Publisher[Event Publisher] -->|NotificationEvents| Queue[Message Queue]
    Queue --> Processor[Event Processor]
    
    subgraph BatchingLogic[Batching Logic]
        Processor -->|Check preferences| UserPrefDB[(User Preferences DB)]
        Processor -->|Create| NEvents[NotificationEvents]
        NEvents -->|Batched into| InAppBatcher[In-App Batcher<br>2 min window]
        NEvents -->|Batched into| EmailBatcher[Email Batcher<br>10 min window]
    end
    
    InAppBatcher -->|Create/Update| BatchedNotifs[BatchedNotifications<br>in Notification DB]
    EmailBatcher -->|Create/Update| BatchedNotifs
    
    BatchedNotifs -->|Query via| NotifAPI[Notification API<br>/api/notifications]
    BatchedNotifs -->|Trigger| EmailService[Email Delivery Service]
    BatchedNotifs -->|Trigger| WebSocket[WebSocket Service]
    
    NotifAPI --> WebApp[Air Web App]
    WebSocket --> WebApp
    EmailService -->|Create| EmailRecords[EmailDelivery Records]
    EmailService --> ThirdParty[3rd Party Email Service]
    
    classDef source fill:#ccf2ff,stroke:#0099cc,stroke-width:2px,color:#333
    classDef process fill:#b3d9ff,stroke:#0066cc,stroke-width:2px,color:#333
    classDef storage fill:#ffe6cc,stroke:#ff9933,stroke-width:2px,color:#333
    classDef delivery fill:#d9f2d9,stroke:#339933,stroke-width:2px,color:#333
    classDef thirdparty fill:#e6ccff,stroke:#9966cc,stroke-width:2px,color:#333
    
    class CRUD,Action,Time,Other source
    class Publisher,Processor,InAppBatcher,EmailBatcher,NotifAPI,EmailService,WebSocket process
    class Queue,NEvents,BatchedNotifs,EmailRecords,UserPrefDB storage
    class WebApp delivery
    class ThirdParty thirdparty
```

## Event Flow Process

1. **Event Sources**: Various system activities generate notification events:
   - CRUD events (asset added, comments created)
   - Action events (downloads, share link views)
   - Time-based events (notifications triggered after time passes)

2. **Event Publishing**: The Event Publisher formats events as NotificationEvents and sends them to the Message Queue

3. **Event Processing**: The Event Processor consumes NotificationEvents and processes them:
   - Checks user NotificationPreferences to determine delivery channels
   - Creates records in the NotificationEvents table
   - Routes events to appropriate batchers based on delivery channel

4. **Batching**: NotificationEvents are batched according to their type and delivery channel:
   - In-App notifications use a 2-minute debouncing window
   - Email notifications use a 10-minute debouncing window
   - Events are aggregated into BatchedNotifications with counts and summaries

5. **Storage**: BatchedNotifications are stored in the Notification DB

6. **Delivery**: Notifications are delivered through appropriate channels:
   - In-App notifications via Notification API and WebSocket Service to the Web App
   - Email notifications via Email Delivery Service to a 3rd Party Email Provider
   - Email delivery status tracked via EmailDelivery records 