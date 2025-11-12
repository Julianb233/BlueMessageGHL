# BlueMessageGHL Implementation Guide

## Quick Start for Developers

This guide provides step-by-step instructions for implementing the core features of BlueMessageGHL.

---

## 1. Environment Setup

### Prerequisites
```bash
# Install Node.js 18+
node --version  # Should be v18.x or higher

# Install PostgreSQL 14+
psql --version  # Should be 14.x or higher

# Install Redis 6+
redis-server --version  # Should be 6.x or higher
```

### Clone and Install
```bash
# Clone the repository
git clone https://github.com/Julianb233/BlueMessageGHL.git
cd BlueMessageGHL

# Install dependencies (to be created)
npm install

# Copy environment template
cp .env.example .env
```

### Environment Variables
Create a `.env` file with the following:

```env
# Application
NODE_ENV=development
PORT=3000
APP_URL=http://localhost:3000

# Database
DATABASE_URL=postgresql://postgres:password@localhost:5432/bluemessageghl_dev

# Redis
REDIS_URL=redis://localhost:6379

# SendBlue API
SENDBLUE_API_KEY=your_api_key_here
SENDBLUE_API_SECRET=your_api_secret_here
SENDBLUE_WEBHOOK_SECRET=your_webhook_secret_here

# GoHighLevel OAuth
GHL_CLIENT_ID=your_client_id_here
GHL_CLIENT_SECRET=your_client_secret_here
GHL_REDIRECT_URI=http://localhost:3000/oauth/callback

# AWS (for production)
AWS_ACCESS_KEY_ID=your_aws_key
AWS_SECRET_ACCESS_KEY=your_aws_secret
AWS_S3_BUCKET=bluemessageghl-media
AWS_REGION=us-east-1

# Stripe
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Security
JWT_SECRET=generate_a_random_secret_here
ENCRYPTION_KEY=generate_a_32_char_key_here
```

---

## 2. Database Setup

### Create Database
```bash
# Create the database
createdb bluemessageghl_dev

# Run migrations (to be created)
npm run db:migrate

# Seed initial data (optional)
npm run db:seed
```

### Migration Structure
Migrations should be created in `src/database/migrations/`:

**Example: Create Tenants Table**
```sql
-- migrations/001_create_tenants.sql
CREATE TABLE IF NOT EXISTS public.tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    ghl_agency_id VARCHAR(255) UNIQUE NOT NULL,
    schema_name VARCHAR(63) UNIQUE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tenants_ghl_agency_id ON public.tenants(ghl_agency_id);
```

---

## 3. Core Implementation Steps

### Step 1: Implement GHL OAuth Flow

**File: `src/api/routes/oauth.js`**
```javascript
const express = require('express');
const router = express.Router();
const axios = require('axios');

// Redirect to GHL authorization
router.get('/authorize', (req, res) => {
  const authUrl = `https://marketplace.gohighlevel.com/oauth/chooselocation?` +
    `response_type=code&` +
    `redirect_uri=${encodeURIComponent(process.env.GHL_REDIRECT_URI)}&` +
    `client_id=${process.env.GHL_CLIENT_ID}&` +
    `scope=contacts.readonly contacts.write conversations.readonly conversations.write conversations.message.readonly conversations.message.write`;
  
  res.redirect(authUrl);
});

// Handle OAuth callback
router.get('/callback', async (req, res) => {
  const { code } = req.query;
  
  try {
    // Exchange code for access token
    const tokenResponse = await axios.post('https://services.leadconnectorhq.com/oauth/token', {
      client_id: process.env.GHL_CLIENT_ID,
      client_secret: process.env.GHL_CLIENT_SECRET,
      grant_type: 'authorization_code',
      code: code,
      redirect_uri: process.env.GHL_REDIRECT_URI
    });
    
    const { access_token, refresh_token, locationId, companyId } = tokenResponse.data;
    
    // Create or update tenant
    // Store tokens securely
    // Provision phone number
    // Redirect to dashboard
    
    res.redirect('/dashboard?success=true');
  } catch (error) {
    console.error('OAuth error:', error);
    res.redirect('/error?message=oauth_failed');
  }
});

module.exports = router;
```

### Step 2: Implement SendBlue Message Sending

**File: `src/services/sendblue.js`**
```javascript
const axios = require('axios');

class SendBlueService {
  constructor() {
    this.apiKey = process.env.SENDBLUE_API_KEY;
    this.apiSecret = process.env.SENDBLUE_API_SECRET;
    this.baseUrl = 'https://api.sendblue.co';
  }

  async sendMessage({ to, message, mediaUrl = null }) {
    try {
      const response = await axios.post(
        `${this.baseUrl}/api/send-message`,
        {
          number: to,
          content: message,
          media_url: mediaUrl,
          send_style: 'invisible' // For iMessage
        },
        {
          headers: {
            'sb-api-key-id': this.apiKey,
            'sb-api-secret-key': this.apiSecret,
            'Content-Type': 'application/json'
          }
        }
      );
      
      return {
        success: true,
        messageId: response.data.message_handle,
        status: response.data.status
      };
    } catch (error) {
      console.error('SendBlue API error:', error.response?.data || error.message);
      return {
        success: false,
        error: error.response?.data?.message || 'Failed to send message'
      };
    }
  }

  async checkiMessageSupport(phoneNumber) {
    try {
      const response = await axios.get(
        `${this.baseUrl}/api/evaluate-service`,
        {
          params: { number: phoneNumber },
          headers: {
            'sb-api-key-id': this.apiKey,
            'sb-api-secret-key': this.apiSecret
          }
        }
      );
      
      return response.data.is_iMessage;
    } catch (error) {
      console.error('Error checking iMessage support:', error);
      return false;
    }
  }
}

module.exports = new SendBlueService();
```

### Step 3: Implement Inbound Webhook Handler

**File: `src/webhooks/sendblue-inbound.js`**
```javascript
const crypto = require('crypto');
const { addToQueue } = require('../workers/queue');

function verifyWebhookSignature(req) {
  const signature = req.headers['sb-signing-secret'];
  const secret = process.env.SENDBLUE_WEBHOOK_SECRET;
  
  if (!signature || !secret) return false;
  
  const payload = JSON.stringify(req.body);
  const computedSignature = crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex');
  
  return signature === computedSignature;
}

async function handleInboundMessage(req, res) {
  // Verify webhook signature
  if (!verifyWebhookSignature(req)) {
    return res.status(403).json({ error: 'Invalid signature' });
  }
  
  const { 
    message_handle, 
    from_number, 
    to_number, 
    content, 
    media_url,
    status 
  } = req.body;
  
  // Add to processing queue
  await addToQueue('inbound-messages', {
    messageId: message_handle,
    from: from_number,
    to: to_number,
    content: content,
    mediaUrl: media_url,
    status: status,
    receivedAt: new Date().toISOString()
  });
  
  // Acknowledge receipt immediately
  res.status(200).json({ success: true });
}

module.exports = { handleInboundMessage };
```

### Step 4: Implement Message Queue Worker

**File: `src/workers/inbound-processor.js`**
```javascript
const { Queue, Worker } = require('bullmq');
const ghlService = require('../services/ghl');
const db = require('../database/client');

const inboundQueue = new Queue('inbound-messages', {
  connection: { url: process.env.REDIS_URL }
});

const worker = new Worker('inbound-messages', async (job) => {
  const { from, to, content, mediaUrl, messageId } = job.data;
  
  try {
    // 1. Find the tenant and location for this phone number
    const phoneNumber = await db.query(
      'SELECT * FROM phone_numbers WHERE number = $1',
      [to]
    );
    
    if (!phoneNumber.rows[0]) {
      throw new Error(`Phone number ${to} not found`);
    }
    
    const location = phoneNumber.rows[0].location_id;
    
    // 2. Find or create contact
    let contact = await db.query(
      'SELECT * FROM contacts WHERE phone_number = $1',
      [from]
    );
    
    if (!contact.rows[0]) {
      // Create new contact in GHL
      const ghlContact = await ghlService.createContact({
        locationId: location,
        phone: from
      });
      
      // Store in local database
      contact = await db.query(
        'INSERT INTO contacts (ghl_contact_id, phone_number) VALUES ($1, $2) RETURNING *',
        [ghlContact.id, from]
      );
    }
    
    // 3. Post message to GHL Conversations
    await ghlService.sendMessage({
      locationId: location,
      contactId: contact.rows[0].ghl_contact_id,
      type: 'SMS',
      message: content
    });
    
    // 4. Log message in database
    await db.query(
      'INSERT INTO messages (contact_id, phone_number_id, direction, channel, body, status, provider_message_id) VALUES ($1, $2, $3, $4, $5, $6, $7)',
      [contact.rows[0].id, phoneNumber.rows[0].id, 'inbound', 'imessage', content, 'received', messageId]
    );
    
    return { success: true };
  } catch (error) {
    console.error('Error processing inbound message:', error);
    throw error; // Will trigger retry
  }
}, {
  connection: { url: process.env.REDIS_URL },
  concurrency: 10
});

module.exports = { inboundQueue, worker };
```

---

## 4. Testing

### Unit Tests
```bash
# Run unit tests
npm test

# Run with coverage
npm run test:coverage
```

### Integration Tests
```bash
# Test OAuth flow
npm run test:oauth

# Test message sending
npm run test:send-message

# Test webhook handling
npm run test:webhooks
```

### Manual Testing Checklist
- [ ] OAuth installation flow works
- [ ] Message can be sent from GHL to recipient
- [ ] Inbound message appears in GHL Conversations
- [ ] Media attachments work correctly
- [ ] Usage metrics are tracked accurately
- [ ] Billing integration works

---

## 5. Deployment

### Staging Deployment
```bash
# Build for production
npm run build

# Deploy to staging
npm run deploy:staging

# Run smoke tests
npm run test:smoke
```

### Production Deployment
```bash
# Deploy to production
npm run deploy:production

# Monitor logs
npm run logs:production
```

---

## 6. Monitoring & Maintenance

### Key Metrics to Monitor
- Message delivery rate
- Webhook processing time
- API response times
- Error rates
- Queue depth

### Logging
Use structured logging with Winston or similar:
```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
});

logger.info('Message sent', { messageId, to, status });
```

---

## 7. Common Issues & Troubleshooting

### Issue: Messages not syncing to GHL
**Solution**: Check that the GHL access token is valid and has the correct scopes.

### Issue: Webhook signature verification failing
**Solution**: Ensure the webhook secret matches what's configured in SendBlue dashboard.

### Issue: Database connection errors
**Solution**: Check that the `search_path` is set correctly for the tenant schema.

---

## Next Steps

1. Set up the development environment
2. Implement the OAuth flow
3. Integrate SendBlue API
4. Build the webhook handlers
5. Create the message queue workers
6. Develop the admin dashboard
7. Write comprehensive tests
8. Deploy to staging
9. Conduct beta testing
10. Launch to production

For questions or support, refer to the main README or create an issue on GitHub.
