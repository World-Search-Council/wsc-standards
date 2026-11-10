# WSC Analytics & Event Tracking Standard Specification (v1.0)
**World Search Council (WSC) — Universal Analytics, Bot Classification & Edge Ingestion Standard**

---

## 1. Executive Summary & Philosophy

The **WSC Analytics Standard** defines a privacy-first, platform-agnostic specification for event collection, client tracking, edge buffering, cryptographic transport, and webhook-driven report delivery.

### Core Tenets:
1. **Privacy-First by Design (ePrivacy & GDPR Compliant)**:
   - Zero persistent tracking cookies in default cookieless mode.
   - Raw IP addresses are **never stored** in the database; ephemeral hashing with daily salt is strictly enforced.
   - First-class support for Consent Management Platforms (CMP, WP Consent API, TCF 2.2).
2. **Ultra-Low Overhead & Non-Blocking**:
   - Zero-dependency client runtime (`< 4 KB`).
   - Non-blocking async execution (`defer`/`async`).
   - Guaranteed transmission upon page unloads using `navigator.sendBeacon` and `fetch(keepalive)`.
3. **Local-First Edge Buffer**:
   - Local buffer (e.g. SQLite FIFO queue) decouples user traffic from remote network latencies.
   - Fault-tolerant batch exports with exponential backoff and transient retry locks.
4. **End-to-End Cryptographic Security**:
   - Outbound batch events are encrypted via **AES-256-CBC** using project-specific keys.
   - API authentication and webhook inbound delivery are verified via **HMAC-SHA256 (`X-WSC-Sig`)**.

---

## 2. Canonical Event Taxonomy

All events captured across web clients, mobile apps, or server-side frameworks must conform to the canonical JSON schema.

### 2.1 Event Schema Definition

```json
{
  "t": 1724838000,
  "e": "pageview",
  "p": "/blog/seo-standard-2026",
  "r": "https://www.google.com/",
  "m": {
    "title": "SEO Standard 2026 - WSC",
    "lang": "en-US",
    "screen": "1920x1080"
  }
}
```

| Field | Type | Description | Required |
| :--- | :--- | :--- | :--- |
| `t` | `integer` | UTC timestamp in epoch seconds | Yes |
| `e` | `string` | Canonical event name (lowercase, snake_case) | Yes |
| `p` | `string` | Path or full page URL (stripped of sensitive query tokens) | Yes |
| `r` | `string` | Referrer URL (or empty string) | No |
| `m` | `object` | Metadata map containing event-specific parameters | No |

---

### 2.2 Standard Event Catalog

#### 1. `pageview`
Triggered once per unique page view navigation.
```json
{
  "e": "pageview",
  "m": {
    "title": "Home - Example",
    "lang": "de-DE",
    "screen": "1440x900"
  }
}
```

#### 2. `time_on_page` (Engaged Active Time)
Measures active engagement vs idle time. Emitted via sparse heartbeats (every 60s) and guaranteed on page exit (`pagehide` / `beforeunload`).
```json
{
  "e": "time_on_page",
  "m": {
    "duration_ms": 45000,
    "active_ms": 38000,
    "final": true,
    "exit_via": "pagehide"
  }
}
```

#### 3. `scroll_depth`
Emitted at standardized vertical scroll milestones: `25%`, `50%`, `75%`, `100%`.
```json
{
  "e": "scroll_depth",
  "m": {
    "percent": 75,
    "max_reached": 82
  }
}
```

#### 4. `time_to_first_interaction` & `time_to_first_click` & `time_to_first_scroll`
Measures user responsiveness and core web engagement.
```json
{
  "e": "time_to_first_interaction",
  "m": {
    "time_ms": 1420,
    "trigger": "click"
  }
}
```

#### 5. `download`
Automatically tracked file downloads (`pdf`, `zip`, `docx`, `xlsx`, `csv`, `mp3`, `mp4`, etc.).
```json
{
  "e": "download",
  "m": {
    "href": "https://example.com/files/whitepaper.pdf",
    "filename": "whitepaper.pdf",
    "extension": "pdf",
    "text": "Download Whitepaper (PDF)"
  }
}
```

#### 6. `video_play`, `video_progress`, `video_complete`
Tracks HTML5 and embedded media engagement.
```json
{
  "e": "video_progress",
  "m": {
    "percent": 50,
    "src": "https://example.com/media/intro.mp4",
    "currentTime": 60,
    "duration": 120
  }
}
```

#### 7. `outbound_click`
Tracks external link clicks leaving the current domain.
```json
{
  "e": "outbound_click",
  "m": {
    "href": "https://external-partner.com",
    "hostname": "external-partner.com",
    "text": "Visit Partner Site"
  }
}
```

#### 8. `form_submit` & `form_field_timing`
Tracks lead captures, contact inquiries, and form interactions (with server-side deduplication).
```json
{
  "e": "form_submit",
  "m": {
    "action": "https://example.com/contact",
    "method": "post",
    "form_id": "contact-form-1",
    "form_title": "Contact Us",
    "source": "php_hook"
  }
}
```

#### 9. `conversion` / `purchase` (E-Commerce & Goals)
Tracks goal completions, purchases, newsletter signups, or custom milestones.
```json
{
  "e": "conversion",
  "m": {
    "conversion_type": "purchase",
    "order_id": 1042,
    "order_number": "1042",
    "revenue": 89.90,
    "subtotal": 75.55,
    "tax": 14.35,
    "shipping": 4.90,
    "currency": "EUR",
    "item_count": 2,
    "items": [
      {
        "product_id": 42,
        "name": "Organic Coffee Blend",
        "sku": "COF-ORG-01",
        "price": 37.77,
        "quantity": 2,
        "total": 75.55
      }
    ],
    "payment_method": "stripe",
    "coupons": ["SPRING10"],
    "source": "woocommerce_server"
  }
}
```

#### 10. `add_to_cart` & `remove_from_cart`
Tracks cart item modifications (both server-side and client AJAX).
```json
{
  "e": "add_to_cart",
  "m": {
    "product_id": 42,
    "product_name": "Organic Coffee Blend",
    "price": 37.77,
    "quantity": 1,
    "currency": "EUR",
    "sku": "COF-ORG-01",
    "categories": ["Coffee", "Beverages"],
    "source": "woocommerce_single_page"
  }
}
```

#### 11. `view_item`, `view_cart` & `begin_checkout`
Tracks product page impressions and the checkout funnel.
```json
{
  "e": "view_item",
  "m": {
    "product_id": 42,
    "name": "Organic Coffee Blend",
    "price": 37.77,
    "currency": "EUR",
    "sku": "COF-ORG-01",
    "categories": ["Coffee"],
    "in_stock": true
  }
}
```

#### 12. `bot_hit` (Server-Side Synthetic Event)
Tracks search engine crawlers and AI LLM bots at the edge server.
```json
{
  "e": "bot_hit",
  "p": "/docs/api-standard",
  "m": {
    "bot_name": "ChatGPT-User",
    "bot_category": "ai",
    "bot_type": "rag",
    "status_code": 200
  }
}
```

---

### 2.3 Acquisition Channels & AI Attribution Taxonomy

Every incoming session/event is attributed to one of the 5 canonical acquisition channels:

| Channel | Description | Identification Patterns |
| :--- | :--- | :--- |
| `ai` | AI Search Engines, Copilots, LLM Citations | `chatgpt.com`, `perplexity.ai`, `claude.ai`, `gemini.google.com`, `copilot.microsoft.com`, `you.com`, `phind.com`, `deepseek.com`, `kagi.com`, `grok.com`, or `utm_medium=ai` |
| `search` | Traditional Organic & Paid Search Engines | `google.*`, `bing.com`, `duckduckgo.com`, `yahoo.com`, `yandex.*`, `baidu.com`, `ecosia.org`, `startpage.com`, `brave.com` |
| `social` | Social Media Platforms & Communities | `facebook.com`, `instagram.com`, `t.co`, `twitter.com`, `x.com`, `linkedin.com`, `pinterest.com`, `reddit.com`, `tiktok.com`, `threads.net`, `youtube.com` |
| `direct` | Direct URL entry, bookmarks, untracked apps | Empty Referrer / no acquisition markers |
| `referral` | External Websites & Backlinks | All other HTTP referrers |

---

### 2.4 Bot Classification Taxonomy

| Category | Bot Type | Key Bots | Purpose & Impact |
| :--- | :--- | :--- | :--- |
| `ai` | `rag` | `ChatGPT-User`, `OAI-SearchBot`, `Claude-User`, `Claude-SearchBot`, `Claude-Web`, `PerplexityBot`, `Perplexity-User`, `Gemini-Deep-Research`, `YouBot` | **Live User Search / Real-time Retrieval**. High intent; user is actively seeking answer from page. |
| `ai` | `training` | `GPTBot`, `ClaudeBot`, `CCBot`, `Bytespider`, `Google-Extended`, `Meta-ExternalAgent`, `xAI-Bot`, `DeepSeekBot`, `Amazonbot`, `Cohere-AI`, `Diffbot`, `Applebot-Extended` | **Model Pre-Training & Indexing**. Large scale scraping for future foundation model checkpoints. |
| `search` | `search` | `Googlebot`, `Google-InspectionTool`, `GoogleOther`, `Bingbot`, `MSNBot`, `DuckDuckBot`, `Yahoo Slurp`, `YandexBot`, `Baiduspider`, `Sogou`, `Exabot`, `Applebot`, `Internet Archive` | **Traditional Search Indexing & Crawl Budget**. |
| `seo_tool`| `seo_tool` | `SemrushBot`, `AhrefsBot`, `Majestic`, `DotBot`, `Screaming Frog`, `Sitebulb` | **Third-party Auditing & Competitive SEO Tools**. |

---

## 3. Client Runtime Architecture (`wsc-tracker.js`)

### 3.1 Global Configuration Interface
```javascript
window.WSC_CONFIG = {
  endpoint: "/wp-json/wsc/v1/track",
  mode: "cookieless", // "cookieless" | "consent" | "strict"
  consentKey: "wsc_consent",
  auto: {
    downloads: true,
    media: true,
    outbound: true,
    forms: true,
    scroll: true,
    timeOnPage: true,
    timeToInteraction: true,
    videoProgress: true,
    formFieldTiming: true,
    serverFormPlugins: true
  }
};
```

### 3.2 Public JavaScript API
```javascript
// Track custom event
window.WSC.track("newsletter_signup", { plan: "pro", campaign: "summer_2026" });

// Consent controls
window.WSC.grantConsent();
window.WSC.revokeConsent();
const isAllowed = window.WSC.hasConsent();

// Force immediate batch queue flush
window.WSC.flush();
```

---

## 4. Edge Buffer & Cryptographic Protocol

### 4.1 Client -> Edge REST Route
- **Method**: `POST`
- **Route**: `/wp-json/wsc/v1/track`
- **Payload**: JSON Array of Canonical Events `[ { ... }, { ... } ]`
- **Security**: Rate limiting per IP, payload size limit (50 KB), sanitized input.

### 4.2 Edge Server-Side Enrichment & Local Queue
1. The Edge server enriches events with server headers (User-Agent, normalized IP, proxy headers).
2. The batch is serialized and encrypted via **AES-256-CBC**:
   `Ciphertext = AES-256-CBC(JSON(Batch), local_aes_key, IV)`
   `Payload = Base64(IV + Ciphertext)`
3. The encrypted payload is inserted into local FIFO SQLite (`queue_temp.sqlite`).

### 4.3 Outbound Gateway Ingest (`POST /v1/ingest`)
- **Headers**:
  - `Content-Type: application/json`
  - `X-WSC-Sig: <HMAC-SHA256(api_secret, raw_body)>`
- **Payload**:
```json
{
  "project_id": "proj_abcdef123456",
  "timestamp": 1724838000,
  "data": [
    "<Base64_IV_Ciphertext_1>",
    "<Base64_IV_Ciphertext_2>"
  ]
}
```

---

## 5. Inbound Webhook & 3-Card Dashboard Pipeline

### 5.1 Gateway -> Edge Webhook Route
- **Method**: `POST`
- **Route**: `/wp-json/wsc/v1/webhook`
- **Signature Header**: `X-WSC-Sig: <HMAC-SHA256(api_secret, raw_body)>`

### 5.2 3-Card Payload Structure

```json
{
  "project_id": "proj_abcdef123456",
  "generated_at": 1724838000,
  "insights": [
    {
      "report_type": "card1_traffic",
      "period_days": 7,
      "unique_visitors": 1240,
      "pageviews": 4580,
      "sessions": 1890,
      "sources": {
        "search": 62.5,
        "ai": 14.2,
        "direct": 15.1,
        "social": 5.2,
        "referral": 3.0
      },
      "sparkline": [ 120, 145, 130, 180, 210, 195, 260 ],
      "human_text": "Traffic up 18% WoW driven by AI search referrals."
    },
    {
      "report_type": "card2_insights",
      "period_days": 7,
      "traffic_change_pct": 18.2,
      "top_channel": "Search / Organic",
      "top_page": "/blog/seo-standard-2026",
      "interaction_rate": 64.5,
      "avg_engaged_time": 42,
      "form_submits": 28,
      "human_text": "High engagement on technical tutorials; recommended action: expand internal linking."
    },
    {
      "report_type": "card3_bots",
      "period_days": 7,
      "human_pct": 78.4,
      "bot_pct": 21.6,
      "ai_training_pct": 8.2,
      "ai_rag_pct": 5.4,
      "search_crawler_pct": 8.0,
      "top_bots": [
        { "name": "GPTBot", "category": "ai_training", "hits": 340 },
        { "name": "Googlebot", "category": "search", "hits": 610 }
      ],
      "quality_rating": "High Quality",
      "human_text": "AI bots crawling technical documentation actively."
    }
  ]
}
```

---

## 6. Portability Blueprint for Non-WordPress Platforms

To implement the WSC Standard on **Shopify, Drupal, TYPO3, Laravel, or Headless Next.js**:
1. **Client**: Embed `wsc-tracker.js` and set `window.WSC_CONFIG.endpoint` to the platform's ingest handler.
2. **Edge Ingest**: Receive events, enrich with server headers, encrypt via AES-256-CBC, and append to a local queue.
3. **Cron Exporter**: Dispatch batches to `POST https://api.worldsearchcouncil.org/v1/ingest` signed with HMAC-SHA256.
4. **Webhook Handler**: Expose an endpoint for `POST /internal/webhook` or `/api/wsc/webhook` to receive the 3-Card Reports.
