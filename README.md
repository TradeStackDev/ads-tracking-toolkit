# 📣 ads-tracking-toolkit

> **Server-side tracking for Meta Conversions API and Google Ads Enhanced Conversions — WordPress plugin + Python backend.**

[![PHP](https://img.shields.io/badge/PHP-8.1+-777bb4?style=flat&logo=php&logoColor=white)](https://php.net)
[![Python](https://img.shields.io/badge/Python-3.11+-3776ab?style=flat&logo=python&logoColor=white)](https://python.org)
[![WordPress](https://img.shields.io/badge/WordPress-6.4+-21759b?style=flat&logo=wordpress&logoColor=white)](https://wordpress.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## Overview

Browser-based ad tracking is broken. iOS privacy changes, ad blockers, and cookie restrictions are killing your attribution data — meaning you're flying blind on ad spend.

`ads-tracking-toolkit` moves your conversion tracking server-side, sending events directly from your server to Meta and Google. No browser. No blockers. No lost conversions.

**What this fixes:**
- Meta Pixel events blocked by iOS 14+ or ad blockers ✅
- Google Ads conversions lost due to cookie restrictions ✅
- Inaccurate ROAS caused by under-reporting ✅
- Duplicate events from pixel + server firing together ✅

---

## Architecture

```
Browser                    Your Server                  Ad Platforms
─────────                  ───────────                  ────────────
Meta Pixel (thin)    →     Python Event Server    →     Meta CAPI
Google gtag (thin)   →     (deduplication layer)  →     Google Enhanced Conv
GTM dataLayer        →     WordPress REST API     →     GA4 Measurement API
```

The browser pixel fires a lightweight event. Your server receives it, enriches it with server-side data (email hash, phone hash, order value), deduplicates it, and forwards it to the ad platform APIs.

---

## Components

### 1. WordPress Plugin (`/wp-plugin`)
Handles the WordPress-side of tracking:
- WooCommerce purchase, add-to-cart, and checkout events
- WordPress user data hashing (email, phone) for CAPI matching
- REST API endpoint that receives browser events from GTM
- Admin dashboard showing event delivery status

### 2. Python Event Server (`/event-server`)
FastAPI microservice that handles server-to-server delivery:
- Receives events from WordPress REST API
- Deduplicates browser + server events using `event_id`
- Enriches events with IP, user agent, and server-side data
- Forwards to Meta CAPI and Google Enhanced Conversions APIs
- Retry logic with exponential backoff on API failures
- Event log with delivery status and error tracking

---

## Quick Start

### WordPress Plugin

```bash
cd wp-content/plugins/
git clone https://github.com/TradeStackDev/ads-tracking-toolkit.git
cd ads-tracking-toolkit
composer install
npm run build
# Activate in WP Admin → Plugins
```

Configure under **Settings → Ads Tracking**:
- Meta Pixel ID + Conversions API access token
- Google Ads conversion ID + label
- Event server URL

### Python Event Server

```bash
git clone https://github.com/TradeStackDev/ads-tracking-toolkit.git
cd ads-tracking-toolkit/event-server

pip install -r requirements.txt

cp .env.example .env
# Edit .env with your API credentials

uvicorn app.main:app --host 0.0.0.0 --port 8080
```

---

## Meta CAPI Integration

```python
from app.capi import MetaCAPIClient

client = MetaCAPIClient(
    pixel_id=os.getenv("META_PIXEL_ID"),
    access_token=os.getenv("META_CAPI_TOKEN")
)

# Send a Purchase event
await client.send_event({
    "event_name": "Purchase",
    "event_time": int(time.time()),
    "event_id": "order_12345",          # For deduplication with browser pixel
    "action_source": "website",
    "user_data": {
        "em": hash_email("customer@example.com"),
        "ph": hash_phone("+15551234567"),
        "client_ip_address": request.client.host,
        "client_user_agent": request.headers.get("user-agent"),
        "fbp": fbp_cookie,
        "fbc": fbc_cookie,
    },
    "custom_data": {
        "currency": "USD",
        "value": 149.99,
        "contents": [{"id": "SKU-123", "quantity": 1}],
        "order_id": "12345",
    },
})
```

---

## Google Enhanced Conversions

```python
from app.google import GoogleEnhancedConversions

gec = GoogleEnhancedConversions(
    customer_id=os.getenv("GOOGLE_ADS_CUSTOMER_ID"),
    conversion_action_id=os.getenv("GOOGLE_ADS_CONVERSION_ID"),
    developer_token=os.getenv("GOOGLE_ADS_DEVELOPER_TOKEN"),
)

await gec.upload_conversion({
    "order_id": "12345",
    "conversion_value": 149.99,
    "currency_code": "USD",
    "user_identifier": {
        "hashed_email": hash_email("customer@example.com"),
        "hashed_phone_number": hash_phone("+15551234567"),
    },
})
```

---

## Event Deduplication

Both Meta and Google require an `event_id` to deduplicate browser + server events. The toolkit handles this automatically:

```php
// WordPress plugin generates a unique event_id per conversion
$event_id = 'purchase_' . $order_id . '_' . time();

// Fires in browser pixel
fbq('track', 'Purchase', $purchase_data, ['eventID' => $event_id]);

// Also sends to Python server with same event_id
wp_remote_post($event_server_url, ['body' => json_encode([
    'event_name' => 'Purchase',
    'event_id'   => $event_id,
    'order_id'   => $order_id,
    ...
])]);
```

---

## Admin Dashboard

The WordPress plugin includes an event log dashboard showing:

| Column | Description |
|--------|-------------|
| Event | Purchase, AddToCart, Lead, etc. |
| Order ID | Associated WooCommerce order |
| Meta Status | Delivered / Failed / Pending |
| Google Status | Delivered / Failed / Pending |
| Timestamp | When event was received |
| Retry Count | Number of delivery attempts |

---

## GTM Setup

The plugin outputs a dataLayer push for every WooCommerce event, making GTM integration straightforward:

```javascript
// Auto-pushed by plugin on purchase
dataLayer.push({
  event: 'purchase',
  event_id: 'purchase_12345_1716830400',
  ecommerce: {
    transaction_id: '12345',
    value: 149.99,
    currency: 'USD',
    items: [{ item_id: 'SKU-123', item_name: 'Product Name', price: 149.99 }]
  }
});
```

---

## Requirements

- WordPress 6.4+ with WooCommerce 8.0+
- PHP 8.1+
- Python 3.11+ (for event server)
- Meta Business account with Conversions API access
- Google Ads account with Enhanced Conversions enabled

---

## License

MIT © Adam Johnson
