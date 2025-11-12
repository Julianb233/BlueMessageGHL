# BlueMessageGHL Database Schema

## Date: November 12, 2025

---

## 1. Overview

This document outlines the database schema for the BlueMessageGHL platform. The schema is designed to support a multi-tenant architecture, where each GoHighLevel agency is a tenant with its own isolated data.

**Database System**: PostgreSQL

**Multi-Tenancy Strategy**: Shared Database, Schema per Tenant. Each tenant's data will reside in a separate schema (e.g., `tenant_12345`). A `public` schema will hold global data like the `tenants` table itself.

---

## 2. Public Schema

This schema contains tables that are shared across all tenants.

### `tenants`

Stores information about each agency (tenant) that has installed the application.

| Column          | Type                | Constraints              | Description                                      |
|-----------------|---------------------|--------------------------|--------------------------------------------------|
| `id`            | `UUID`              | `PRIMARY KEY`            | Unique identifier for the tenant.                |
| `name`          | `VARCHAR(255)`      | `NOT NULL`               | Name of the agency.                              |
| `ghl_agency_id` | `VARCHAR(255)`      | `UNIQUE NOT NULL`        | The agency ID from GoHighLevel.                  |
| `schema_name`   | `VARCHAR(63)`       | `UNIQUE NOT NULL`        | The name of the PostgreSQL schema for this tenant. |
| `created_at`    | `TIMESTAMPTZ`       | `NOT NULL DEFAULT NOW()` | Timestamp of when the tenant was created.        |
| `updated_at`    | `TIMESTAMPTZ`       | `NOT NULL DEFAULT NOW()` | Timestamp of the last update.                    |

---

## 3. Tenant Schema

Each tenant will have the following tables within their dedicated schema.

### `users`

Stores user accounts for the agency, allowing staff to log in to the BlueMessageGHL dashboard.

| Column             | Type                | Constraints              | Description                                          |
|--------------------|---------------------|--------------------------|------------------------------------------------------|
| `id`               | `UUID`              | `PRIMARY KEY`            | Unique identifier for the user.                      |
| `ghl_user_id`      | `VARCHAR(255)`      | `UNIQUE`                 | The user ID from GoHighLevel (if linked).            |
| `email`            | `VARCHAR(255)`      | `UNIQUE NOT NULL`        | User's email address (for login).                    |
| `password_hash`    | `VARCHAR(255)`      | `NOT NULL`               | Hashed password for authentication.                  |
| `role`             | `VARCHAR(50)`       | `NOT NULL DEFAULT 'user'`| User role (e.g., 'admin', 'user').                   |
| `created_at`       | `TIMESTAMPTZ`       | `NOT NULL DEFAULT NOW()` | Timestamp of when the user was created.              |

### `locations`

Stores information about the GoHighLevel locations (sub-accounts) associated with the tenant.

| Column              | Type                | Constraints              | Description                                          |
|---------------------|---------------------|--------------------------|------------------------------------------------------|
| `id`                | `UUID`              | `PRIMARY KEY`            | Unique identifier for the location.                  |
| `ghl_location_id`   | `VARCHAR(255)`      | `UNIQUE NOT NULL`        | The location ID from GoHighLevel.                    |
| `name`              | `VARCHAR(255)`      | `NOT NULL`               | Name of the location.                                |
| `access_token`      | `TEXT`              | `NOT NULL`               | Encrypted OAuth access token for this location.      |
| `refresh_token`     | `TEXT`              | `NOT NULL`               | Encrypted OAuth refresh token for this location.     |
| `token_expires_at`  | `TIMESTAMPTZ`       | `NOT NULL`               | Expiration timestamp for the access token.           |
| `created_at`        | `TIMESTAMPTZ`       | `NOT NULL DEFAULT NOW()` | Timestamp of when the location was linked.           |

### `phone_numbers`

Stores the phone numbers provisioned for the tenant's locations.

| Column            | Type                | Constraints                 | Description                                          |
|-------------------|---------------------|-----------------------------|------------------------------------------------------|
| `id`              | `UUID`              | `PRIMARY KEY`               | Unique identifier for the phone number.              |
| `location_id`     | `UUID`              | `REFERENCES locations(id)`  | The location this number belongs to.                 |
| `number`          | `VARCHAR(20)`       | `UNIQUE NOT NULL`           | The phone number in E.164 format.                    |
| `provider`        | `VARCHAR(50)`       | `NOT NULL DEFAULT 'sendblue'`| The messaging provider (e.g., 'sendblue', 'twilio'). |
| `provider_id`     | `VARCHAR(255)`      | `NULL`                      | The ID of the number from the provider's system.     |
| `capabilities`    | `JSONB`             | `NOT NULL DEFAULT '[]'`     | Capabilities of the number (e.g., 'imessage', 'sms').|
| `is_active`       | `BOOLEAN`           | `NOT NULL DEFAULT true`     | Whether the number is active.                        |
| `created_at`      | `TIMESTAMPTZ`       | `NOT NULL DEFAULT NOW()`    | Timestamp of when the number was provisioned.        |

### `contacts`

Stores contacts synced from GoHighLevel.

| Column            | Type                | Constraints              | Description                                          |
|-------------------|---------------------|--------------------------|------------------------------------------------------|
| `id`              | `UUID`              | `PRIMARY KEY`            | Unique identifier for the contact.                   |
| `ghl_contact_id`  | `VARCHAR(255)`      | `UNIQUE NOT NULL`        | The contact ID from GoHighLevel.                     |
| `phone_number`    | `VARCHAR(20)`       | `NOT NULL`               | Contact's primary phone number.                      |
| `first_name`      | `VARCHAR(255)`      | `NULL`                   | Contact's first name.                                |
| `last_name`       | `VARCHAR(255)`      | `NULL`                   | Contact's last name.                                 |
| `email`           | `VARCHAR(255)`      | `NULL`                   | Contact's email address.                             |
| `created_at`      | `TIMESTAMPTZ`       | `NOT NULL DEFAULT NOW()` | Timestamp of when the contact was created.           |

### `messages`

Stores a log of all inbound and outbound messages.

| Column              | Type                | Constraints                 | Description                                          |
|---------------------|---------------------|-----------------------------|------------------------------------------------------|
| `id`                | `UUID`              | `PRIMARY KEY`               | Unique identifier for the message.                   |
| `ghl_conversation_id`| `VARCHAR(255)`      | `NULL`                      | The conversation ID from GoHighLevel.                |
| `contact_id`        | `UUID`              | `REFERENCES contacts(id)`   | The contact associated with the message.             |
| `phone_number_id`   | `UUID`              | `REFERENCES phone_numbers(id)`| The phone number used for the message.               |
| `direction`         | `VARCHAR(10)`       | `NOT NULL`                  | 'inbound' or 'outbound'.                              |
| `channel`           | `VARCHAR(10)`       | `NOT NULL`                  | 'imessage' or 'sms'.                                 |
| `body`              | `TEXT`              | `NULL`                      | The text content of the message.                     |
| `status`            | `VARCHAR(20)`       | `NOT NULL`                  | 'queued', 'sent', 'delivered', 'failed', 'received'. |
| `provider_message_id`| `VARCHAR(255)`      | `NULL`                      | The message ID from the provider (SendBlue).         |
| `sent_at`           | `TIMESTAMPTZ`       | `NULL`                      | Timestamp of when the message was sent.              |
| `delivered_at`      | `TIMESTAMPTZ`       | `NULL`                      | Timestamp of when the message was delivered.         |
| `created_at`        | `TIMESTAMPTZ`       | `NOT NULL DEFAULT NOW()`    | Timestamp of when the message was created.           |

### `media`

Stores information about media attachments.

| Column            | Type                | Constraints              | Description                                          |
|-------------------|---------------------|--------------------------|------------------------------------------------------|
| `id`              | `UUID`              | `PRIMARY KEY`            | Unique identifier for the media.                     |
| `message_id`      | `UUID`              | `REFERENCES messages(id)`| The message this media is attached to.               |
| `s3_url`          | `VARCHAR(255)`      | `NOT NULL`               | The URL of the media file in AWS S3.                 |
| `mime_type`       | `VARCHAR(100)`      | `NOT NULL`               | The MIME type of the media file.                     |
| `file_size`       | `INTEGER`           | `NULL`                   | The size of the file in bytes.                       |
| `created_at`      | `TIMESTAMPTZ`       | `NOT NULL DEFAULT NOW()` | Timestamp of when the media was uploaded.            |

### `usage_metrics`

Stores aggregated usage data for billing purposes.

| Column            | Type                | Constraints              | Description                                          |
|-------------------|---------------------|--------------------------|------------------------------------------------------|
| `id`              | `UUID`              | `PRIMARY KEY`            | Unique identifier for the usage record.              |
| `location_id`     | `UUID`              | `REFERENCES locations(id)`| The location this usage belongs to.                  |
| `billing_cycle`   | `DATE`              | `NOT NULL`               | The start date of the billing cycle.                 |
| `imessages_sent`  | `INTEGER`           | `NOT NULL DEFAULT 0`     | Count of iMessages sent in this cycle.               |
| `sms_sent`        | `INTEGER`           | `NOT NULL DEFAULT 0`     | Count of SMS messages sent in this cycle.            |
| `messages_received`| `INTEGER`           | `NOT NULL DEFAULT 0`     | Count of messages received in this cycle.            |
| `media_storage_mb`| `INTEGER`           | `NOT NULL DEFAULT 0`     | Media storage used in MB.                            |
| `updated_at`      | `TIMESTAMPTZ`       | `NOT NULL DEFAULT NOW()` | Timestamp of the last usage update.                  |

---

## 4. Relationships

- A `tenant` has many `users` and `locations`.
- A `location` has many `phone_numbers` and `usage_metrics`.
- A `contact` can have many `messages`.
- A `phone_number` can be used for many `messages`.
- A `message` can have many `media` attachments.

This schema provides a solid foundation for building the BlueMessageGHL platform, with a clear separation of concerns and a scalable multi-tenant design.
