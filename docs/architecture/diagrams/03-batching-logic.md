# Batching Logic Flow

This diagram shows how notification events are batched into consumable notifications, highlighting how event type determines grouping logic and recipient selection.

```mermaid
flowchart TD
    Event[NotificationEvent] --> EventType{Event Type?}
    
    EventType -->|Asset Added| G1["Group by:<br>• In-App: User+Board<br>• Email: Board"]
    EventType -->|Comment Created| G2["Group by:<br>• In-App: User+Asset<br>• Email: Asset"]
    EventType -->|Assets Downloaded| G3["Group by:<br>• In-App: User+Board<br>• Email: Board"]
    EventType -->|Share Link Viewed| G4["Group by:<br>• In-App: Asset<br>• Email: Asset"]
    
    G1 --> Recipients[Determine Recipients]
    G2 --> Recipients
    G3 --> Recipients
    G4 --> Recipients
    
    Recipients --> TimeWindow[Apply Time Window]
    TimeWindow --> BatchCreate[Create/Update<br>BatchedNotification]
    BatchCreate --> LinkEvents[Create<br>NotificationEventBatch]
    LinkEvents --> Delivery[Deliver via Channels]
    
    subgraph Details[Key Process Details]
        direction TB
        R1["Recipient Logic:<br>• Asset owner(s)<br>• Board members<br>• @mentions<br>• Action performers"]
        T1["Debouncing windows:<br>• In-App: 2 minutes (batchStartTime to batchEndTime)<br>• Email: 10 minutes (batchStartTime to batchEndTime)"]
        B1["For each BatchedNotification:<br>• Aggregate similar events<br>• Track count<br>• Generate summary content"]
        L1["NotificationEventBatch:<br>• Links events to batched notifications<br>• Maintains event order<br>• Records when each event was added"]
        D1["Delivery channels (based on preferences):<br>• In-App via WebSocket (if inAppEnabled)<br>• Email via 3rd Party Provider (if emailEnabled)"]
    end
    
    R1 -.- Recipients
    T1 -.- TimeWindow
    B1 -.- BatchCreate
    L1 -.- LinkEvents
    D1 -.- Delivery
    
    classDef main fill:#b3d9ff,stroke:#0066cc,stroke-width:2px,color:#333
    classDef decision fill:#ffccdd,stroke:#cc3366,stroke-width:2px,color:#333
    classDef group fill:#d9f2d9,stroke:#339933,stroke-width:1px,color:#333
    classDef detail fill:#ccf2ff,stroke:#0099cc,stroke-width:1px,color:#333
    classDef recipients fill:#e6ccff,stroke:#9966cc,stroke-width:2px,color:#333
    
    class Event,TimeWindow,BatchCreate,LinkEvents,Delivery main
    class EventType decision
    class G1,G2,G3,G4 group
    class Recipients recipients
    class R1,T1,B1,L1,D1 detail
```

## Recipient Determination Rules

The system determines notification recipients based on the event type and context:

| **Notification Type** | **Recipients** | **Logic** |
| --------------------- | -------------- | --------- |
| Asset Added | Board members | All users with access to the board receive notifications when assets are added |
| Comment Created | Asset viewers + @mentions | Users who have viewed the asset + any users specifically @mentioned in the comment |
| Assets Downloaded | Asset owners | The users who own the assets that were downloaded |
| Share Link Viewed | Link creator | The user who created the share link |
| Share Link Not Viewed | Link creator | The user who created the share link |

After determining recipients, each recipient's NotificationPreferences are checked (inAppEnabled, emailEnabled) to filter delivery channels.

## Simple Batching Example

**Scenario**: UserA uploads 10 assets to "Marketing Board" and 3 assets to "Design Board" within 2 minutes. UserB uploads 5 assets to "Marketing Board" in the same timeframe.

**Processing Steps**:
1. Each upload creates a NotificationEvent
2. NotificationEvents are linked to BatchedNotifications via NotificationEventBatch records
3. The BatchedNotification stores count, content, and time window (batchStartTime, batchEndTime)

**Recipients**:
- All members of Marketing Board (UserC, UserD, UserE)
- All members of Design Board (UserC, UserF)

**Results**:
- **In-App BatchedNotifications** (if inAppEnabled):
  - For UserC, UserD, UserE: "UserA uploaded 10 assets to Marketing Board" and "UserB uploaded 5 assets to Marketing Board"
  - For UserC, UserF: "UserA uploaded 3 assets to Design Board"

- **Email BatchedNotifications** (if emailEnabled):
  - For UserC, UserD, UserE: "15 assets were uploaded to Marketing Board" (sent via 3rd party email provider)
  - For UserC, UserF: "3 assets were uploaded to Design Board" (sent via 3rd party email provider)

## Notification Type Grouping Logic

| **Notification Type** | **Group By for In-App** | **Group By for Email** | **Example BatchedNotification** |
| --------------------- | ----------------------- | ---------------------- | ----------- |
| Asset Added           | User + Board            | Board                  | "Mark uploaded 10 assets to Marketing Board" |
| Comment Created       | User + Asset            | Asset                  | "Sarah commented on file.jpg" |
| Assets Downloaded     | User + Board            | Board                  | "Alex downloaded 5 files from Product Launch" |
| Share Link Viewed     | Asset                   | Asset                  | "Share link for report.pdf was viewed 3 times" |
| Share Link Not Viewed | Asset                   | Asset                  | "Share link for logo.png hasn't been viewed in 5 days" | 