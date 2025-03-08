# Air Unified Notification System

This directory contains documentation for the Air Unified Notification System, designed to enhance how users stay informed about activities related to their digital assets.

## Documentation Structure

- `architecture/` - Architecture diagrams and technical specifications
  - `diagrams/` - Mermaid-based diagrams for system components
  - `README.md` - Architecture overview and diagram explanations

## System Features

The notification system supports:

1. **Multiple Notification Channels**
   - In-App notifications displayed in the Air web app
   - Email notifications sent to users' email addresses

2. **Various Notification Types**
   - CRUD events (Asset Added, Comment Created)
   - Action events (Assets Downloaded, Asset Share Link Viewed)
   - Time-based events (Asset Share Link Not Viewed)

3. **Intelligent Batching**
   - Groups similar notifications within configurable time windows
   - Different batching rules for in-app (2 min) and email (10 min) notifications
   - Customized grouping criteria for different notification types

4. **User Preferences**
   - Users can configure preferences for both in-app and email notifications
   - Opt in/out per notification type
   - Default settings provide a balance of information and minimal noise

## Design Goals

The system is designed with these key goals in mind:

1. **Minimal Performance Impact** - Asynchronous processing to avoid affecting the main application
2. **Extensibility** - Easy to add new notification types and delivery channels
3. **User Control** - Comprehensive preference management
4. **Reliability** - Ensuring notifications are delivered even during system disruptions
5. **Scalability** - Handles growing volumes of notifications efficiently

## Implementation Approach

The implementation uses:

- **PostgreSQL** for data storage (via Prisma ORM)
- **TypeScript/Node.js** for backend logic
- **Message Queue** for asynchronous processing
- **RESTful API** for client interaction
- **Third-party Email Service** for email delivery 

## Cloud Infrastructure Integration

The notification system leverages cloud services for scalability, reliability, and operational efficiency:

### AWS Implementation
1. **Compute & API Services**
   - Lambda for event processing and batching logic
   - ECS (Fargate) for WebSocket service
   - API Gateway for REST and WebSocket endpoints

2. **Data Storage**
   - RDS PostgreSQL for notification data with read replicas
   - ElastiCache (Redis) for WebSocket session management
   - S3 for archiving older notifications

3. **Messaging & Events**
   - SQS for reliable message queuing
   - EventBridge for time-based notification triggers

4. **Delivery & Monitoring**
   - SES for email delivery
   - CloudWatch and X-Ray for observability

### High Availability & Disaster Recovery
- Multi-AZ deployment for critical components
- Database backups and point-in-time recovery
- Dead-letter queues for failed notification processing

## Operational Considerations

### Monitoring & Observability
The system includes comprehensive monitoring to ensure reliable operation:

1. **Key Metrics**
   - Notification processing latency
   - Batching efficiency (events per batched notification)
   - Channel delivery success rates
   - WebSocket connection stability
   - Email delivery and open rates

2. **Logging Strategy**
   - Correlation IDs to track notifications through the pipeline
   - Structured logging with consistent fields
   - Variable log levels based on environment
   - PII redaction in logs

3. **Alerting**
   - Notification delivery failures above threshold
   - Processing backlogs
   - Abnormal notification volumes
   - Third-party email service degradation

### Edge Cases & Failure Handling

1. **Delivery Failures**
   - Exponential backoff retry for email delivery failures
   - Alternative delivery path if primary channel fails
   - Notification persistence for offline users

2. **System Overload**
   - Dynamic batch window adjustment during traffic spikes
   - Priority queuing for critical notifications
   - Graceful degradation of real-time features

3. **Data Consistency**
   - Idempotent event processing to prevent duplicate notifications
   - Database transactions for notification batching operations
   - Versioned user preferences to prevent conflict during updates

4. **User Experience Edge Cases**
   - Handling of notifications for deleted content
   - Treatment of notifications for revoked user access
   - Limit mechanisms to prevent notification spam

## Assumptions, Trade-Offs & Alternative Approaches

### Key Technical Decisions

1. **Event-Driven Architecture**
   - **Decision**: We chose an event-driven architecture with a message queue rather than direct API calls.
   - **Reasoning**: This decouples notification generation from core application logic, minimizing performance impact on the main Air application. Event-driven design also allows for better scalability and resilience.

2. **Two-Tier Notification Storage**
   - **Decision**: Store both raw notification events and batched notifications with explicit linking between them.
   - **Reasoning**: While this requires more storage, it provides full event history for auditing, debugging, and analytics. The many-to-many relationship through `NotificationEventBatch` allows for flexible batching rules and rebatching if needed.

3. **WebSockets for Real-Time Delivery**
   - **Decision**: Use WebSockets rather than polling for real-time in-app notifications.
   - **Reasoning**: While more complex to implement and maintain, WebSockets reduce server load and provide a better user experience with instant notifications. This is particularly important for collaborative features where users need immediate feedback.

4. **Third-Party Email Providers**
   - **Decision**: Integrate with established email providers rather than self-hosting email delivery.
   - **Reasoning**: Improves deliverability rates and reduces maintenance overhead. The dedicated `EmailDelivery` tracking model ensures we maintain visibility despite using external services.

### Recipient Determination Assumptions

A key aspect of our notification system is determining who should receive each notification. We made several assumptions about this crucial logic:

1. **Access-Based Recipient Model**
   - **Asset Added**: We assume all members with access to a board should be notified when assets are added to it, regardless of who added them. This supports collaborative workflows where team members need visibility into board changes.
   
   - **Comment Created**: We assume only users who have previously viewed the asset should receive these notifications, plus any users explicitly @mentioned in the comment. This prevents notification spam for assets a user hasn't shown interest in.
   
   - **Assets Downloaded**: We assume only the asset owners (creators/uploaders) care about downloads. This protects the privacy of downloaders while keeping creators informed about asset usage.
   
   - **Share Link Viewed/Not Viewed**: We assume only the user who created the share link needs these notifications. This supports follow-up workflows without notifying the entire team about individual sharing activities.

2. **Implicit Hierarchies**
   - We assume board owners should receive more comprehensive notifications than regular members. The system is designed to support future enhancements where notification routing could be role-based.
   
   - We assume organization administrators should be able to configure organization-wide notification defaults, overriding the system defaults but not individual user preferences.

3. **Content Visibility Implications**
   - We assume a user will only receive notifications for content they have permission to access. The system filters notification recipients based on current access rights, not just at the time of the event.
   
   - We assume when a user loses access to content, they should no longer receive new notifications about it, but historical notifications will remain visible (marked with a "no longer accessible" indicator if clicked).

4. **Cross-Organizational Boundaries**
   - We assume notifications should respect organizational boundaries, except for explicitly shared content. This prevents information leakage between separate organizations.
   
   - For guest users, we assume they should only receive notifications about the specific content they have been invited to access.

These assumptions were made to balance comprehensive awareness with notification fatigue, respecting both content security and user preferences.

### Performance, Scaling & Security Considerations

1. **Database Scaling Strategy**
   - The notification system's database load will grow with user activity. To address this, we've designed for horizontal partitioning (sharding) by user or time periods.
   - Indexes are carefully selected for the most common query patterns (retrieving unread notifications, batching similar events).
   - Older notifications can be archived to cold storage to maintain performance.

2. **Rate Limiting & Quotas**
   - Built-in rate limiting for email notifications to prevent accidental flooding of user inboxes.
   - Respect for third-party email provider quotas through configurable throttling.
   - Queue-based processing that can slow down during traffic spikes without failing.

3. **Security Measures**
   - Token-based authentication for WebSocket connections with regular rotation.
   - Encrypted personal data within notification content.
   - Sanitization of user-generated content in notifications to prevent XSS attacks.
   - Strict validation of notification preferences to prevent preference spoofing.

### Balancing Real-Time with Batch Processing

1. **Dual Time Windows**
   - Different debouncing windows for in-app (2 min) and email (10 min) notifications balance immediacy with noise reduction.
   - In-app notifications prioritize timeliness while email batching minimizes inbox clutter.

2. **Hybrid Delivery Approach**
   - WebSockets deliver immediate updates for active users.
   - Batched email notifications provide comprehensive updates for offline users.
   - The system adapts to user engagement patterns, increasing batching for less active users.

3. **Graceful Degradation**
   - If real-time services are overloaded, the system can temporarily increase batching windows.
   - Fallback to REST polling when WebSockets are unavailable or blocked.

### Alternative Approaches Considered

1. **Notification Storage**
   - **Alternative**: Store only batched notifications without raw events.
     - **Pro**: Simpler data model and reduced storage requirements.
     - **Con**: Loss of detailed event history and limited ability to rebatch or audit.
   - **Alternative**: Use a NoSQL database for flexible schema evolution.
     - **Pro**: Easier to add new notification types without schema migrations.
     - **Con**: More complex query patterns and potential consistency issues.

2. **Real-Time Delivery**
   - **Alternative**: Server-sent events (SSE) instead of WebSockets.
     - **Pro**: Simpler to implement, works over HTTP.
     - **Con**: One-way communication only, potential issues with certain proxies/firewalls.
   - **Alternative**: Smart polling with exponential backoff.
     - **Pro**: Simplest implementation, works everywhere.
     - **Con**: Higher server load, not truly real-time, increased client complexity.

3. **Batching Strategy**
   - **Alternative**: Fixed time windows instead of rolling debounce windows.
     - **Pro**: Simpler to implement and reason about.
     - **Con**: Less responsive to user activity patterns, potentially delayed notifications.
   - **Alternative**: Client-side batching with server coordination.
     - **Pro**: Reduced server processing, more personalized experience.
     - **Con**: Inconsistent experience across devices, higher client complexity.

4. **Email Delivery**
   - **Alternative**: Direct SMTP email sending.
     - **Pro**: No third-party dependencies or costs.
     - **Con**: Poor deliverability, high maintenance overhead, complex scaling.
   - **Alternative**: Triggered emails without delivery tracking.
     - **Pro**: Simpler implementation.
     - **Con**: Limited analytics and troubleshooting capabilities.

## System Extension Examples

### Adding a New Notification Type

To demonstrate the system's extensibility, here's how a new notification type could be added:

1. **Define the New Event Type**
   ```typescript
   // In notificationTypes.ts
   export enum NotificationType {
     // Existing types
     ASSET_ADDED = 'asset_added',
     COMMENT_CREATED = 'comment_created',
     ASSETS_DOWNLOADED = 'assets_downloaded',
     SHARE_LINK_VIEWED = 'share_link_viewed',
     SHARE_LINK_NOT_VIEWED = 'share_link_not_viewed',
     
     // New notification type
     ASSET_UPDATED = 'asset_updated'
   }
   ```

2. **Configure Batching Rules**
   ```typescript
   // In batchingConfig.ts
   export const batchingRules = {
     // Existing rules
     [NotificationType.ASSET_ADDED]: {
       inApp: { groupBy: ['userId', 'boardId'], window: 120 }, // 2 min
       email: { groupBy: ['boardId'], window: 600 } // 10 min
     },
     // ...
     
     // New notification type rules
     [NotificationType.ASSET_UPDATED]: {
       inApp: { groupBy: ['userId', 'assetId'], window: 120 },
       email: { groupBy: ['assetId'], window: 600 }
     }
   };
   ```

3. **Add Content Templates**
   ```typescript
   // In notificationTemplates.ts
   export const templates = {
     // Existing templates
     [NotificationType.ASSET_ADDED]: {
       single: '${user.firstName} uploaded an asset to ${board.name}',
       multiple: '${user.firstName} uploaded ${count} assets to ${board.name}'
     },
     // ...
     
     // New notification type templates
     [NotificationType.ASSET_UPDATED]: {
       single: '${user.firstName} updated "${asset.name}"',
       multiple: '${user.firstName} updated ${count} assets in ${board.name}'
     }
   };
   ```

4. **Set Default User Preferences**
   ```typescript
   // In defaultPreferences.ts
   export const defaultPreferences = {
     // Existing defaults
     [NotificationType.ASSET_ADDED]: { inAppEnabled: true, emailEnabled: true },
     // ...
     
     // New notification type defaults
     [NotificationType.ASSET_UPDATED]: { inAppEnabled: true, emailEnabled: false }
   };
   ```

The system automatically handles the rest - event processing, batching according to the new rules, and delivery based on user preferences. 