# WSC API & Endpoints Reference

This document provides complete technical specifications for all public, internal, and remote webhook endpoints across the WSC ecosystem.

---

## 1. Central Gateway Endpoints (`https://api.worldsearchcouncil.org` — `v2.3.0`)

### 1.0 Health Probe & Prometheus OpenMetrics Exporter

* **Health Endpoint**: `GET /v1/health` (alias: `GET /health`)
  * **Authentication**: Public
  * **Response (`200 OK`)**: `{"status": "ok", "version": "2.3.0", "mysql": "up", "redis": "up", "clickhouse": "up"}`
* **Prometheus Metrics Endpoint**: `GET /metrics`
  * **Authentication**: Optional `Authorization: Bearer <METRICS_BEARER_TOKEN>` (when `METRICS_BEARER_TOKEN` is configured)
  * **Content-Type**: `text/plain; version=0.0.4; charset=utf-8`
  * **Exposed Metrics**: HTTP request counters & latency histograms (`/__unmatched_404__` scanner cardinality protection), `/v1/ingest` batch & event throughput, passive RabbitMQ queue depths & consumer counts across all 8 primary/DLX queues, ClickHouse query latency (`wsc_clickhouse_ping_duration_seconds`), and upstream dependency health (`wsc_dependency_up`).

---

### 1.1 Project Registration

Registers a new CMS or shop instance and establishes cryptographic credentials.

* **Endpoint**: `POST /v1/register`
* **Authentication**: None (Open during initial setup)
* **Content-Type**: `application/json`

#### Request Body
```json
{
  "domain": "https://example.com",
  "local_aes_key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
  "platform": "shopware6",
  "webhook_url": "https://example.com/api/wsc/webhook",
  "version": "2.2.3"
}
```

| Parameter | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `domain` | `string` | Yes | Base URL of the client instance (e.g. `https://example.com`). |
| `local_aes_key` | `string` | Yes | 64-character hexadecimal representation of a 256-bit AES key generated locally by the CMS. |
| `platform` | `string` | No | Platform identifier: `wordpress`, `drupal`, `drupal7`, `joomla`, `shopware6`, `magento2`, `shopify`. Default: `wordpress`. |
| `webhook_url` | `string` | No | Optional custom webhook URL. If omitted, the standard platform endpoint is inferred. |

#### Response (`200 OK`)
```json
{
  "project_id": "proj_a1b2c3d4e5f67890",
  "api_secret": "9f8e7d6c5b4a3210fedcba9876543210"
}
```

---

### 1.2 Telemetry Ingestion

Receives encrypted micro-batches of telemetry events from CMS instances.

* **Endpoint**: `POST /v1/ingest`
* **Authentication**: Header `X-WSC-Sig` (HMAC-SHA256 signature)
* **Content-Type**: `application/json`

#### Required Headers
| Header | Format | Description |
| :--- | :--- | :--- |
| `X-WSC-Sig` | `string (hex)` | HMAC-SHA256 signature of the raw HTTP request body using `api_secret`. |

#### Request Body
```json
{
  "project_id": "proj_a1b2c3d4e5f67890",
  "timestamp": 1724930000,
  "source": "shopware6",
  "version": "2.2.3",
  "data": [
    "BASE64_ENCRYPTED_IV_AND_CIPHERTEXT_ROW_1",
    "BASE64_ENCRYPTED_IV_AND_CIPHERTEXT_ROW_2"
  ]
}
```

#### Decrypted Micro-Batch Payload Structure
Each string in `data` decrypts (via `AES-256-CBC` + `local_aes_key`) into the canonical Zero-PII envelope:
```json
{
  "server_data": {
    "session_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "ua": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36...",
    "headers": {
      "referer": "https://google.com/",
      "sec_ch_ua": "\"Chromium\";v=\"124\", \"Google Chrome\";v=\"124\""
    }
  },
  "events": [
    {
      "t": 1724930000,
      "e": "pageview",
      "p": "/shop/product-123",
      "r": "https://google.com/",
      "m": { "title": "High-Performance Shoes" }
    }
  ]
}
```

#### Response Codes
* `200 OK`: `{"status": "ok", "queued_events": 2}`
* `401 Unauthorized`: Missing or invalid `X-WSC-Sig` HMAC header.
* `403 Forbidden`: Project is blocked or inactive (`status: 999`).
* `404 Not Found`: `project_id` not found in registry.

---

### 1.3 Shopify Web Pixel Ingestion

Receives unencrypted telemetry events from sandboxed Shopify Web Pixel workers.

* **Endpoint**: `POST /v1/shopify/pixel`
* **Authentication**: `project_id` in payload
* **Content-Type**: `application/json`

#### Request Body
```json
{
  "project_id": "proj_a1b2c3d4e5f67890",
  "context": {
    "shop": "wsc-demo.myshopify.com",
    "currency": "EUR"
  },
  "events": [
    {
      "event": "pageview",
      "timestamp": 1724930000,
      "data": { "url": "https://wsc-demo.myshopify.com/products/shoes" }
    },
    {
      "event": "purchase",
      "timestamp": 1724930005,
      "data": {
        "order_id": "gid://shopify/Order/123456",
        "total_amount": 89.99,
        "currency": "EUR"
      }
    }
  ]
}
```

#### Response (`202 Accepted`)
```json
{
  "ok": true,
  "received": 2
}
```

---

### 1.4 Internal Webhook Dispatcher

Called internally by the report generation worker to push computed 3-Card reports back to client instances.

* **Endpoint**: `POST /internal/dispatch-insights`
* **Access**: Restricted to internal Docker network (`http://gateway:8000`)
* **Content-Type**: `application/json`

#### Request Body
```json
{
  "project_id": "proj_a1b2c3d4e5f67890",
  "card1": { ... },
  "card2": { ... },
  "card3": { ... },
  "insights": [ ... ]
}
```

---

### 1.5 AI Coverage & URL Inspector Query

Queries the distributed sensor network for AI bot visibility (Live RAG vs. Training scrapers) across a domain or single URL.

* **Endpoint**: `POST /v1/ai-coverage/query`
* **Authentication**: Header `X-WSC-Sig` (HMAC-SHA256 signature)
* **Content-Type**: `application/json`

#### Request Body
```json
{
  "project_id": "proj_a1b2c3d4e5f67890",
  "query": "https://example.com/blog/ai-future",
  "timeframe": "30d"
}
```

#### Response (`200 OK`)
```json
{
  "status": "success",
  "query": {
    "raw": "https://example.com/blog/ai-future",
    "root_domain": "example.com",
    "url_path": "/blog/ai-future",
    "is_cross_domain": false
  },
  "url_metrics": {
    "is_known_in_network": true,
    "total_hits_30d": 42,
    "rag": {
      "visited": true,
      "hits_7d": 5,
      "hits_30d": 18,
      "last_seen": 1756425000,
      "last_seen_human": "2 hours ago",
      "top_bots": [{ "name": "ChatGPT-User", "hits": 12 }, { "name": "PerplexityBot", "hits": 6 }]
    },
    "training": {
      "visited": true,
      "hits_7d": 8,
      "hits_30d": 24,
      "last_seen": 1756410000,
      "last_seen_human": "6 hours ago",
      "top_bots": [{ "name": "GPTBot", "hits": 14 }, { "name": "ClaudeBot", "hits": 10 }]
    }
  },
  "domain_coverage": {
    "root_domain": "example.com",
    "known_urls_count": 840,
    "ai_covered_urls_count": 684,
    "ai_coverage_pct": 81.4,
    "rag_coverage_pct": 62.1,
    "training_coverage_pct": 79.5,
    "network_confidence": "high"
  }
}
```

---

### 1.6 Industry Benchmark Intelligence & NLP Classification

Provides Google NLP Content Category classification and $k$-anonymized comparative vertical benchmarks.

#### A. Automated NLP Classification
* **Endpoint**: `POST /v1/benchmarks/classify`
* **Request**:
```json
{
  "project_id": "proj_a1b2c3d4e5f67890",
  "title": "Fashion Store - Sneakers & Streetwear",
  "meta_description": "Exclusive organic cotton apparel and designer sneakers.",
  "keywords": "fashion, sneakers, streetwear"
}
```
* **Response (`200 OK`)**:
```json
{
  "status": "success",
  "vertical": "/Shopping/Apparel",
  "vertical_label": "E-Commerce: Fashion & Apparel",
  "confidence": 0.94,
  "method": "nlp_centroid",
  "k_anonymity_status": "active",
  "active_nodes_in_vertical": 142
}
```

#### B. List Active Benchmark Verticals
* **Endpoint**: `GET /v1/benchmarks/verticals`
* **Response (`200 OK`)**: Lists all supported Google NLP categories, node counts, and $k \ge 25$ unlock status.

#### C. Query Vertical Benchmarks
* **Endpoint**: `POST /v1/benchmarks/query`
* **Authentication**: Header `X-WSC-Sig` (HMAC-SHA256 signature)
* **Response (`200 OK`)**: Returns percentile metrics ($p_{50}, p_{75}, p_{90}$) and delta ratings (`above_average`, `below_average`, `top_10`).

---

### 1.7 Domain & Tech Health Sentinel Endpoints

Provides infrastructure surveillance, email security, TLS validation, security headers score, and drift tracking.

#### A. Trigger On-Demand Scan (Asynchronous Queue or Synchronous RPC)
* **Endpoint**: `POST /v1/sentinel/scan`
* **Authentication**: Header `X-WSC-Sig` (HMAC-SHA256 signature)
* **Request**:
```json
{
  "project_id": "proj_a1b2c3d4e5f67890",
  "domain": "example.com",
  "sync": false
}
```

* **Asynchronous Response (`202 Accepted` - Default for Web/UI Requests)**:
Returned immediately when `sync: false`. The task is enqueued to RabbitMQ `wsc_sentinel_tasks` for the dedicated `worker-sentinel`.
```json
{
  "status": "queued",
  "task_id": "sentinel_a1b2c3d4e5f67890",
  "message": "Domain & Tech Health scan enqueued in Cloud Mesh"
}
```

* **Synchronous Response (`200 OK` - Direct RPC mode when `sync: true`)**:
```json
{
  "status": "success",
  "report": {
    "project_id": "proj_a1b2c3d4e5f67890",
    "domain": "example.com",
    "last_scan_at": 1756425600,
    "last_change_at": 1752800000,
    "days_since_change": 42,
    "drift_status": "NO_CHANGE",
    "summary": {
      "overall_health": "GOOD",
      "ssl_status": "OK",
      "ssl_days_left": 64,
      "mail_health": "GOOD",
      "web_health": "GOOD",
      "dns_health": "GOOD",
      "headers_score": "5/6"
    },
    "mail_security": {
      "provider": "Google Workspace",
      "mx_status": "OK",
      "spf_status": "OK",
      "dkim_status": "OK",
      "dmarc_status": "GOOD",
      "dmarc_policy": "reject"
    },
    "web_security": {
      "http_to_https": "OK",
      "www_canonicalization": "OK",
      "canonical_url": "https://example.com/",
      "hsts_enabled": "GOOD",
      "ipv6_accessible": "Yes"
    },
    "security_headers": {
      "score": 5,
      "total": 6,
      "hsts": true,
      "csp": true,
      "x_frame_options": true,
      "x_content_type_options": true,
      "referrer_policy": true,
      "permissions_policy": false
    },
    "dns_security": {
      "dnssec_status": "GOOD",
      "nameserver_count": 4
    },
    "identified_services": ["Google Workspace", "Meta (Facebook)", "Shopify", "DocuSign"]
  }
}
```

#### B. Query Latest Domain Health
* **Endpoint**: `GET /v1/sentinel/health/{project_id}`
* **Response (`200 OK`)**: Latest cached health report from Redis cache conforming to `schemas/v1/domain-health.schema.json`.

#### C. WordPress Plugin REST Client Endpoints
1. **Trigger Scan**: `POST /wp-json/wsc/v1/sentinel/scan`
   - Nonce-authenticated (`X-WP-Nonce`) admin action.
   - Dispatches async `run_scan(false, true)` to Gateway.
   - Returns `202 Accepted` with `{"status": "queued", "pending": true}`.
2. **Poll Status / Report**: `GET /wp-json/wsc/v1/sentinel/report`
   - Nonce-authenticated (`X-WP-Nonce`) query.
   - Returns `{ "status": "success", "pending": boolean, "report": {...} }`.
   - Polls every 2 seconds until `pending: false`, then triggers smooth UI reload.
3. **Inbound Webhook**: `POST /wp-json/wsc/v1/insights` (or `/webhook`)
   - HMAC-SHA256 authenticated webhook received from Gateway `worker-sentinel`.
   - Payload: `{ "report_type": "domain_health", "report": {...} }`.
   - Stores report in SQLite and WP options, and invalidates pending scan transients.

---

### 1.8 SpeedSentinel Dual-Indicator PageSpeed & RAG-Readiness Endpoints

Provides self-hosted, API-key-free synthetic lab performance audits combining **🤖 RAG & KI-Crawler Readiness** with **👤 Human Core Web Vitals**.

#### A. Trigger On-Demand Speed & RAG Audit (Asynchronous Queue)
* **Endpoint**: `POST /v1/pagespeed/audit`
* **Authentication**: Header `X-WSC-Sig` (HMAC-SHA256 signature using `api_secret`)
* **Content-Type**: `application/json`
* **Request**:
```json
{
  "project_id": "proj_a1b2c3d4e5f67890",
  "url": "https://example.com/",
  "strategy": "mobile",
  "sync": false
}
```
* **Response (`202 Accepted`)** (Standard Asynchronous Mode):
```json
{
  "status": "queued",
  "task_id": "ps_dad179e4ea43",
  "message": "PageSpeed audit queued. Results will be saved to ClickHouse and dispatched via webhook.",
  "report": null
}
```
* **Response (`200 OK`)** (Synchronous Debugging Mode when `sync: true`):
Returns immediate `PageSpeedAuditReport` payload containing `rag_readiness`, `human_ux`, `metrics`, and `action_items`.

#### B. Query Latest Performance Scorecard from ClickHouse
* **Endpoint**: `GET /v1/pagespeed/{project_id}?strategy=mobile`
* **Response (`200 OK`)**: Latest assembled audit report retrieved directly from ClickHouse storage (`wsc_assembled_reports` or `wsc_pagespeed_daily`). Does not generate fake or mock data. If no audit has been performed yet, returns HTTP 404.

---

### 1.9 Historical & Latest Bento Reports Pull (`/v1/reports`)

Allows CMS connectors (including localhost/staging sites without inbound webhook reachability) to pull assembled 7-Card Bento snapshots and up to 730 days (2 years) of paginated historical time series directly from ClickHouse.

* **Endpoint**: `GET /v1/reports`
* **Authentication**: Header `X-WSC-Sig` (HMAC-SHA256 signature)
* **Query Parameters**:
  * `project_id`: Registered `proj_...` identifier
  * `mode`: `latest` | `history` | `overview`
  * `days`: `1`..`730`
  * `limit` & `offset`: Chunked pagination (`pagination.has_more`, `pagination.next_offset`)
  * `include_cards` / `include_daily` / `card_types` / `trigger_push`

---

## 2. Remote CMS & Shop Webhook Endpoints

When the Gateway dispatches insights, it makes an authenticated POST request to the remote instance:

| Platform | Webhook Endpoint URL | AI Manifest (`/llms.txt`) |
| :--- | :--- | :--- |
| **WordPress / WooCommerce** | `POST {domain}/wp-json/wsc/v1/webhook` | `GET {domain}/llms.txt` |
| **Drupal 8 / 9 / 10 / 11** | `POST {domain}/api/wsc/webhook` | `GET {domain}/llms.txt` |
| **Drupal 7** | `POST {domain}/api/wsc/webhook` | `GET {domain}/llms.txt` |
| **Joomla 4 / 5** | `POST {domain}/index.php?option=com_wsc_analytics&task=webhook.receive` | `GET {domain}/llms.txt` |
| **Shopware 6** | `POST {domain}/api/wsc/webhook` | `GET {domain}/llms.txt` |
| **Magento 2** | `POST {domain}/wsc/api/webhook` | `GET {domain}/llms.txt` |
| **Shopify (App Proxy)** | `POST {domain}/apps/wsc/webhook` | `GET {domain}/apps/wsc/llms.txt` |

### Inbound Webhook Headers & Verification
* **Header**: `X-WSC-Sig: <hex_hmac_sha256>`
* **Verification Algorithm**:
  ```php
  $calculated_sig = hash_hmac('sha256', $raw_request_body, $api_secret);
  if (!hash_equals($calculated_sig, $request_header_sig)) {
      // Reject: 401 Unauthorized
  }
  ```

### Canonical 7-Card Webhook Payload Body
```json
{
  "version": "2.2.3",
  "updated_at": 1763038000,
  "card1_traffic": {
    "data_7d": {
      "visitors": 1420,
      "pageviews": 5800,
      "avg_engaged_time": "1m 24s",
      "orders": 38,
      "revenue": 4910.50,
      "currency": "EUR",
      "channels": { "ai_pct": 18.5, "search_pct": 42.0, "social_pct": 14.5, "direct_pct": 20.0, "referral_pct": 5.0 }
    },
    "data_30d": {
      "visitors": 6100,
      "pageviews": 24500,
      "avg_engaged_time": "1m 32s",
      "orders": 162,
      "revenue": 21450.00,
      "currency": "EUR",
      "channels": { "ai_pct": 16.0, "search_pct": 45.0, "social_pct": 15.0, "direct_pct": 19.0, "referral_pct": 5.0 }
    }
  },
  "card2_insights": {
    "data_7d": {
      "summary": "AI search traffic increased by 24% after FAQ updates.",
      "actions": [
        { "type": "content", "title": "Expand Product FAQ", "description": "High conversion on FAQ pages.", "impact": "high" }
      ]
    }
  },
  "card3_bots": {
    "data_7d": {
      "human_pct": 88.5,
      "bot_pct": 11.5,
      "ai_rag_hits": 420,
      "ai_training_hits": 150,
      "search_hits": 1200
    }
  },
  "card_engagement": { "data_7d": { "avg_engaged_sec": 84, "ttfi_ms": 620, "scroll_75_pct": 48.2 } },
  "card_outbound": { "data_7d": { "total_clicks": 195, "top_categories": ["Reference", "Social"] } },
  "card_ecommerce": { "data_7d": { "product_views": 2100, "add_to_cart": 184, "checkout": 96, "orders": 38 } },
  "domain_health": { "summary": { "overall_health": "GOOD", "ssl_days_left": 64, "headers_score": "6/6" } },
  "pagespeed": { "rag_readiness": { "s_rag": 94, "t_rag_ms": 185 }, "human_ux": { "score": 92 } }
}
```

---

## 3. Local Client Tracker Endpoints

Each CMS instance exposes a first-party intake route to receive DOM beacons from frontend browser JavaScript:

* **WordPress**: `POST /wp-json/wsc/v1/track`
* **Drupal**: `POST /api/wsc/track`
* **Joomla**: `POST /index.php?option=com_wsc_analytics&task=track.event`
* **Shopware 6**: `POST /api/wsc/track`
* **Magento 2**: `POST /wsc/api/track`

#### Tracker Payload Structure
```json
{
  "events": [
    {
      "t": 1724930000,
      "e": "pageview",
      "p": "/products/smart-watch",
      "r": "https://google.com",
      "m": { "title": "Smart Watch Product Page" }
    }
  ]
}
```
The controller immediately extracts the client IP, User-Agent, and headers, combines them with the event into the queue row, and inserts it into `queue_temp.sqlite` within `< 1ms`.

---

## 4. Local Client Health & Diagnostics Endpoints

Each CMS instance exposes a health and diagnostics endpoint for uptime probes, container orchestration, and WordPress Site Health.

### 4.1 WordPress / WooCommerce Health Endpoint
* **Endpoint**: `GET /wp-json/wsc/v1/health`
* **Authentication**: Optional. Unauthenticated requests receive high-level status; authenticated requests (Admin session or `?token=<wsc_health_token>`) receive full system diagnostics.

#### Unauthenticated Response (`200 OK` / `503 Service Unavailable`)
```json
{
  "status": "healthy",
  "plugin": "WSC Search & Health",
  "version": "2.2.3",
  "standard_spec": "1.0",
  "healthy": true,
  "timestamp": 1763038000
}
```

#### Authenticated Diagnostics Response (`200 OK`)
```json
{
  "status": "healthy",
  "issues": [],
  "plugin": {
    "name": "WSC Search & Health",
    "version": "2.2.3",
    "standard_spec": "1.0",
    "text_domain": "wsc-search-health",
    "tracking_mode": "cookieless",
    "tracking_active": true,
    "dev_mode": false
  },
  "modules": {
    "active": ["wsc-schema", "wsc-meta-tools", "wsc-whitelist", "wsc-seo-score", "wsc-redirects"],
    "allowed": ["wsc-schema", "wsc-meta-tools", "wsc-whitelist", "wsc-seo-score", "wsc-redirects"]
  },
  "storage": {
    "upload_dir": "/var/www/html/wp-content/uploads/wsc-storage",
    "is_writable": true,
    "reports_db": { "exists": true, "size_bytes": 65536, "size_human": "64 KB" },
    "queue_db": { "exists": true, "size_bytes": 16384, "size_human": "16 KB" },
    "sqlite_version": "3.40.1"
  },
  "queue": {
    "pending_count": 0,
    "oldest_age_seconds": 0,
    "last_processed_time": 1725032000,
    "last_processed_date": "2026-08-30 15:06:40Z"
  },
  "cron": {
    "hook": "wsc_analytics_batch_export",
    "frequency_seconds": 300,
    "is_scheduled": true,
    "next_run_timestamp": 1725032300,
    "next_run_human": "3 minutes"
  },
  "gateway": {
    "mode": "hybrid",
    "connected": true,
    "project_id": "proj_a1b...",
    "encryption": "AES-256-CBC",
    "openssl_available": true,
    "last_reconnect": 1725000000,
    "last_dashboard_report": 1725028800
  },
  "bot_tracking": {
    "level": "search_ai",
    "hits_24h_total": 142,
    "hits_24h_ai": 38,
    "hits_24h_search": 104
  },
  "system": {
    "php_version": "8.2.18",
    "memory_limit": "256M",
    "memory_usage": "18.4 MB",
    "curl_available": true,
    "server_time": 1725032400,
    "server_utc": "2026-08-30 15:20:00Z"
  }
}
```

---

## 8. WSC Authority, Membership & MCP Endpoints (`www.worldsearchcouncil.org`)

### 8.1 License & Certification Verification

Verifies active WSC memberships, certified agency tier status, and Tribunal sanction state.

* **Endpoint**: `POST https://www.worldsearchcouncil.org/api/v1/license/verify` (Note: apex `https://worldsearchcouncil.org/*` permanently redirects via HTTP `308` preserving `POST` bodies)
* **Authentication**: None (License key in payload)
* **Content-Type**: `application/json`

#### Request Body
```json
{
  "license_key": "WSC-AGENCY-9A4B-8821",
  "site_url": "https://example.com",
  "site_title": "Example Brand",
  "project_uuid": "proj_a1b2c3d4e5f67890",
  "platform": "wordpress",
  "wsc_version": "2.2.3",
  "locale": "de_DE",
  "php_version": "8.2.18"
}
```

#### Response (`200 OK` - Valid)
```json
{
  "valid": true,
  "tier": "agency",
  "holder": "Acme Search Consulting",
  "expires_at": 1788192000,
  "features": [
    "co_branding",
    "directory_listing",
    "advanced_benchmarks"
  ]
}
```

---

### 8.2 Featured Partners Live-Feed

Provides a low-latency list of certified WSC Solution Partners for the dynamic 1-Column Bento Dashboard Card.

* **Endpoint**: `GET https://www.worldsearchcouncil.org/api/v1/partners/featured`
* **Query Parameters**:
  * `locale`: Target locale string (e.g. `de_DE`, `en_US`).
  * `limit`: Number of featured partners (default: `6`).

#### Response (`200 OK`)
```json
{
  "locale": "de_DE",
  "partners": [
    {
      "name": "Search Sentinel Labs",
      "slug": "search-sentinel-labs",
      "specialty": "GEO & AI Bot-Defense",
      "location": "Köln, DE",
      "rating": "5.0",
      "badge": "Founding Partner",
      "profile_url": "https://www.worldsearchcouncil.org/partner/search-sentinel-labs?ref=wp-dashboard"
    },
    {
      "name": "IndexWave Digital",
      "slug": "indexwave-digital",
      "specialty": "Schema 2.0 & Reality Matrix",
      "location": "Zürich, CH",
      "rating": "4.9",
      "badge": "Certified Partner",
      "profile_url": "https://www.worldsearchcouncil.org/partner/indexwave-digital?ref=wp-dashboard"
    }
  ]
}
```

---

### 8.3 Member Intelligence Model Context Protocol (`POST /api/v1/mcp`)

Provides a Streamable HTTP JSON-RPC 2.0 MCP server (`2024-11-05` & `2025-03-26`) for AI coding and research clients (Claude Desktop, Cursor, Google Antigravity, Windsurf).

* **Endpoint**: `POST https://www.worldsearchcouncil.org/api/v1/mcp`
* **Authentication**: `Authorization: Bearer wsc_mcp_<personal_token>` (generated & rotatable at `https://www.worldsearchcouncil.org/member/developer`)
* **Exposed Tools (`tools/list`, `tools/call`)**: `wsc_get_member_overview`, `wsc_list_connected_sites`, `wsc_get_site_subpages`, `wsc_scan_domain_sentinel`, `wsc_list_rfc_standards`, `wsc_read_rfc_spec`, `wsc_verify_credential`
* **Exposed Resources (`resources/list`, `resources/read`)**: `wsc://llms.txt` and `wsc://schemas/v1/*.schema.json`



