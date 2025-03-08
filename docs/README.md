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