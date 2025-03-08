# Notification Creation Sequence Diagram

This diagram illustrates the sequence of operations that occur during notification creation and processing, including real-time delivery via websockets and email delivery tracking.

```mermaid
sequenceDiagram
    actor User
    participant WebApp as Air Web App
    participant API as Air REST API
    participant Publisher as Event Publisher
    participant Queue as Message Queue
    participant Processor as Event Processor
    participant UserPrefDB as User Preferences DB
    participant NotifDB as Notification DB
    participant WebSocket as WebSocket Service
    participant EmailService as Email Delivery Service
    participant EmailSvc as 3rd Party Email Service
    
    User->>API: Perform action (e.g., upload asset)
    API->>Publisher: Generate notification event
    Publisher->>Queue: Publish NotificationEvent
    Queue->>Processor: Consume NotificationEvent
    
    Processor->>UserPrefDB: Check NotificationPreferences
    UserPrefDB-->>Processor: Return preferences (inAppEnabled, emailEnabled)
    Processor->>NotifDB: Create/Store NotificationEvent
    
    Processor->>Processor: Apply batching rules
    Processor->>Processor: Group by criteria (User+Board, etc.)
    Processor->>Processor: Apply time window
    Processor->>Processor: Determine recipients
    
    alt New batch
        Processor->>NotifDB: Create new BatchedNotification
        Processor->>NotifDB: Create NotificationEventBatch links
    else Update existing batch
        Processor->>NotifDB: Update existing BatchedNotification (count, content)
        Processor->>NotifDB: Add NotificationEventBatch link
    end
    
    alt In-App notification enabled
        NotifDB->>WebSocket: New/Updated BatchedNotification event
        WebSocket->>WebApp: Send via websocket (real-time)
        WebApp->>User: Display notification instantly
    end
    
    alt Email notification enabled
        Processor->>NotifDB: Create EmailDelivery record (status: pending)
        Processor->>EmailService: Request email delivery with tracking ID
        EmailService->>NotifDB: Get BatchedNotification content and recipient info
        EmailService->>EmailSvc: Send email via 3rd party API
        EmailSvc-->>User: Send email (async)
        EmailSvc-->>EmailService: Delivery status callback
        EmailService->>NotifDB: Update EmailDelivery status (sent/failed)
        Note over EmailSvc,EmailService: Provider webhooks for opened/clicked events
    end
    
    %% Set participant styling
    note over User: End User
    note over WebApp: Client App
    note over API: API Server
    note over Publisher: Event Publisher
    note over Queue: Message Queue
    note over Processor: Event Processor
    note over UserPrefDB: User Preferences Database
    note over NotifDB: Notification Database
    note over WebSocket: Real-time Service
    note over EmailService: Email Service
    note over EmailSvc: 3rd Party Provider
```

## Notification Creation and Delivery Process

1. **Triggering Event**: A user performs an action in the Air application (e.g., uploads assets, creates comments)

2. **Event Generation**: The Air REST API detects the action and generates a NotificationEvent

3. **Event Publishing**: The NotificationEvent is published to a message queue for asynchronous processing

4. **Event Processing**: The Event Processor:
   - Consumes the NotificationEvent from the queue
   - Checks user NotificationPreferences in the User Preferences DB
   - Stores the NotificationEvent in the Notification DB
   - Applies batching rules based on notification type
   - Groups events by appropriate criteria (User+Board, Asset, etc.)
   - Applies the debouncing time window
   - Determines who should receive the notification

5. **BatchedNotification Creation**:
   - If this is a new notification group, a new BatchedNotification is created
   - If a similar notification exists within the time window, it's updated (incrementing count, updating content)
   - NotificationEventBatch records are created to link NotificationEvents to their BatchedNotifications
   - The BatchedNotification stores attributes like count, content, and time window information

6. **Notification Delivery**:
   - **In-App Notifications** (if inAppEnabled is true):
     - New/Updated BatchedNotifications trigger the WebSocket Service
     - Notifications are delivered in real-time to connected clients
     - Displayed instantly in the web app UI
   - **Email Notifications** (if emailEnabled is true):
     - EmailDelivery record created in database (with pending status)
     - Email Service retrieves BatchedNotification content and recipient data
     - Email Service sends via 3rd Party Provider API
     - Provider sends email to recipients
     - Delivery status callbacks update the EmailDelivery record
     - Email provider webhooks update additional status events (opened, clicked, etc.)

This approach ensures comprehensive notification handling with:
- Clear relationship tracking between individual events and batched notifications
- Preference-based delivery to respect user notification settings
- Real-time delivery for active users via websockets
- Reliable email delivery with complete tracking through EmailDelivery records
- Proper separation of concerns between data storage and service communication 