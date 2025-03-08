# Air Notification System Architecture

This directory contains architecture diagrams and documentation for the Air Unified Notification System. The system is designed to support both in-app and email notifications, with intelligent batching and user preferences.

## Directory Structure

- `diagrams/` - Contains all architecture diagrams in Mermaid format
  - `01-high-level-architecture.md` - Overall system architecture
  - `02-event-flow.md` - Flow of notification events through the system
  - `03-batching-logic.md` - Logic for batching notifications and user preferences
  - `04-data-model.md` - Database schema for the notification system
  - `05-notification-creation.md` - Sequence diagram for notification creation
  - `06-notification-api.md` - API endpoints and notification fetch flow
  - `07-websocket-connection.md` - WebSocket connection and subscription process

## Key Components

- **Event Publisher**: Generates notification events from user actions
- **Event Processor & Batcher**: Processes events and applies batching rules
- **Notification Service**: Manages notification delivery and API
- **WebSocket Service**: Delivers real-time notifications to connected clients
- **Email Delivery Service**: Handles email generation and delivery tracking
- **3rd Party Email Provider**: External service used for sending emails (e.g., SendGrid, Amazon SES)

## Viewing the Diagrams

These diagrams are created using Mermaid, a Markdown-based diagramming tool. You can view them in:

1. GitHub (which renders Mermaid diagrams automatically)
2. VS Code with the Mermaid extension
3. Notion (which supports Mermaid diagrams)
4. Any Markdown viewer that supports Mermaid

## Next Steps

After reviewing the architecture, these are the recommended next steps for implementation:

1. **Set up the database schema** using Prisma
2. **Implement the notification event generation** in the Air REST API
3. **Build the event processor and batching logic**
4. **Create the notification API endpoints**
5. **Implement the WebSocket service** for real-time notifications
6. **Configure the email delivery service** with 3rd party provider integration 