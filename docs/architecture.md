# BlueMessageGHL Technical Architecture

## Date: November 12, 2025

---

## 1. Guiding Principles

This architecture is designed with the following principles in mind:

- **Scalability**: The system must handle a growing number of agencies, locations, and message volumes.
- **Reliability**: Ensure high uptime and message delivery rates.
- **Security**: Protect customer data and secure all communication channels.
- **Maintainability**: Use modern technologies and best practices to create a clean, well-documented codebase.
- **Cost-Effectiveness**: Leverage serverless technologies and efficient resource management to minimize operational costs.

---

## 2. System Components

The BlueMessageGHL platform is composed of several key components that work together to deliver a seamless messaging experience.

![System Architecture Diagram](https://i.imgur.com/example.png)  <!-- Placeholder for a real diagram -->

### 2.1. Frontend Application (Admin Dashboard)

- **Technology**: React.js with Next.js
- **Purpose**: Provides a web interface for agencies to manage their accounts, view usage analytics, configure settings, and manage billing.
- **Key Features**:
  - User authentication and management
  - Agency and location management
  - Usage dashboards and reports
  - Billing and subscription management
  - Phone number configuration
  - White-label settings

### 2.2. Backend API

- **Technology**: Node.js with Express.js
- **Purpose**: The core of the application, handling all business logic, data processing, and communication with external services.
- **Key Responsibilities**:
  - GHL OAuth 2.0 flow management
  - Securely storing and managing API keys and tokens
  - Exposing a REST API for the frontend dashboard
  - Handling requests for sending messages
  - Processing data from the message queue

### 2.3. Webhook Handlers

- **Technology**: AWS Lambda (Serverless Functions)
- **Purpose**: Receive real-time notifications from GoHighLevel and SendBlue.
- **Key Handlers**:
  - **Inbound Message Webhook**: Receives incoming messages from SendBlue.
  - **GHL Action Webhook**: Triggered by actions in GHL (e.g., sending a message from the Conversations tab).
  - **Delivery Status Webhook**: Receives message status updates from SendBlue.
  - **Stripe Billing Webhook**: Handles billing events like successful payments or subscription changes.

### 2.4. Message Queue

- **Technology**: Redis (using BullMQ library)
- **Purpose**: Decouple intensive tasks from the main API and webhook handlers to ensure reliability and scalability.
- **Queues**:
  - **Outbound Message Queue**: Queues messages to be sent via SendBlue.
  - **Inbound Message Queue**: Queues incoming messages to be synced with GHL.
  - **Status Update Queue**: Queues delivery status updates to be processed.

### 2.5. Database

- **Technology**: PostgreSQL (hosted on AWS RDS)
- **Purpose**: The primary data store for the application.
- **Key Data**:
  - User and tenant information
  - Message history and logs
  - Usage metrics
  - Billing and subscription data
  - Phone number configurations

### 2.6. External Services

- **GoHighLevel API**: For CRM integration, contact management, and conversation sync.
- **SendBlue API**: For sending and receiving iMessages and SMS.
- **Stripe API**: For processing payments and managing subscriptions.
- **AWS S3**: For storing media files (images, videos, voice notes).

---

## 3. Multi-Tenant Architecture

To support a resale model for multiple agencies, a multi-tenant architecture is essential. We will adopt a **database-per-tenant** approach for maximum data isolation and security, which is a common pattern for SaaS applications with stringent data privacy requirements.

### 3.1. Tenant Identification

- A **Tenant** represents a single GoHighLevel Agency account.
- Each tenant will have a unique `tenant_id` (UUID).
- When an agency installs the app, a new tenant record is created.

### 3.2. Data Isolation Strategy

- **Shared Database, Schema per Tenant**: We will use a single PostgreSQL database instance, but each tenant will have its own schema (`tenant_123`, `tenant_456`, etc.).
- **How it Works**:
  1. When a request comes in, the `tenant_id` is identified from the authenticated user's session or API key.
  2. A middleware sets the PostgreSQL `search_path` for the current transaction to the appropriate tenant's schema (e.g., `SET search_path TO tenant_123, public;`).
  3. All subsequent queries in that transaction are automatically scoped to that tenant's tables.
- **Benefits**:
  - **Strong Data Isolation**: Prevents data leakage between tenants.
  - **Scalability**: Easier to manage than a separate database for every tenant.
  - **Backup/Restore**: Can back up and restore data for individual tenants.

### 3.3. Tenant Provisioning Workflow

1. Agency admin installs the BlueMessageGHL app from the GHL Marketplace.
2. The user is redirected to the BlueMessageGHL platform for the OAuth flow.
3. Upon successful authorization, a `tenant_created` event is triggered.
4. A background worker:
   - Creates a new schema in the PostgreSQL database for the new tenant.
   - Runs migrations to create the necessary tables within that schema.
   - Provisions a default phone number via the SendBlue API.
   - Creates an admin user account for the agency.
5. The agency admin is redirected to the BlueMessageGHL dashboard to complete setup.

---

## 4. Data Flow Diagrams

### 4.1. Outbound Message Flow (GHL to Recipient)

```
1. User sends message in GHL Conversations tab.
   |
   v
2. GHL sends a webhook to BlueMessageGHL's API Gateway.
   |
   v
3. API Gateway authenticates the request and places the message job in the 'outbound-messages' queue (Redis).
   |
   v
4. A worker process picks up the job from the queue.
   |
   v
5. Worker calls the SendBlue API to send the message.
   |
   v
6. SendBlue delivers the message (iMessage or SMS) to the recipient.
   |
   v
7. SendBlue sends a 'delivery_status' webhook back to BlueMessageGHL.
   |
   v
8. The status webhook is processed, and the message status is updated in the database and synced back to GHL.
```

### 4.2. Inbound Message Flow (Recipient to GHL)

```
1. Recipient sends a reply via iMessage or SMS.
   |
   v
2. SendBlue receives the message and sends a webhook to BlueMessageGHL's 'inbound-message' handler (AWS Lambda).
   |
   v
3. The Lambda function validates the webhook and places the message job in the 'inbound-messages' queue (Redis).
   |
   v
4. A worker process picks up the job from the queue.
   |
   v
5. Worker formats the message for the GHL API.
   |
   v
6. Worker calls the GHL Conversations API to post the message to the correct contact's conversation thread.
   |
   v
7. The message is stored in the BlueMessageGHL database for logging and analytics.
```

---

## 5. Security & Compliance

- **Authentication**: All API endpoints will be protected. Frontend access will use JWT-based sessions, and API access for GHL will use the OAuth 2.0 access tokens.
- **Data Encryption**: All data will be encrypted in transit (TLS 1.3) and at rest (AWS RDS encryption). Sensitive credentials (API keys, secrets) will be stored in AWS Secrets Manager.
- **Webhook Security**: All incoming webhooks will be verified using HMAC signatures (e.g., `sb-signing-secret` from SendBlue).
- **Compliance**: The application will have built-in features to help agencies comply with TCPA, including automated opt-out handling (STOP keyword) and consent tracking.

---

## 6. Scalability and Performance

- **Load Balancing**: An Application Load Balancer (ALB) will distribute traffic across multiple instances of the backend API.
- **Auto-Scaling**: The backend service will be containerized (Docker) and deployed on an auto-scaling platform like AWS Fargate or ECS.
- **Serverless Webhooks**: Using AWS Lambda for webhook ingestion allows for massive scalability, as each webhook invocation runs in its own isolated environment.
- **Database Read Replicas**: For the analytics dashboard, read replicas of the PostgreSQL database can be used to avoid impacting the performance of the primary database.
- **CDN**: The frontend application and static assets will be served through a CDN (Vercel's CDN or AWS CloudFront) for fast global performance.
