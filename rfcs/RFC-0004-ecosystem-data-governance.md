# WSC Ecosystem Data Governance & Cross-Platform Contract Standard

**Standard Version:** `1.7.0`  
**Governing Body:** World Search Council (WSC) Architecture Board  
**Target Systems:** WSC Gateway, WordPress/WooCommerce, Drupal 7, Drupal 8/9/10/11, Joomla 4/5, Shopify, Shopware 6, Magento 2 (Adobe Commerce).

---

## 1. Executive Summary & Governance Objectives

The World Search Council (WSC) ecosystem unites self-hosted CMS/Shops (using SQLite WAL local queues) with cloud-native SaaS platforms (Shopify Web Pixels & Metafields), centralized Ingestion & Processing Gateway (ClickHouse), and the Infrastructure Surveillance Sentinel.

To prevent schema drift, runtime crashes, Shopify Metafield serialization errors, and cross-platform divergence, this document establishes the **binding Data Governance Standard**.

### Core Governance Principles:
1. **Strict Contract First**: Every event, webhook, and key-value payload must conform to an immutable JSON Schema with SemVer compatibility rules.
2. **Deterministic Naming & Typing**: Mandatory `snake_case`, ISO standards (ISO-4217 currencies, ISO-8601 / Unix timestamps), and exact type definitions (e.g. numbers never stringified).
3. **Shopify Metafield Safety Guard**: Metafields have strict payload limits (max 2 MB for JSON metafields) and zero tolerance for malformed JSON. Every Gateway push is validated against the canonical schema before dispatch.
4. **Decoupled Local Stores with Shared Semantics**: Whether stored in SQLite `queue_temp.sqlite` / `reports_temp.sqlite` or Shopify Admin Metafields, the JSON object shape is 100% identical.

---

## 2. Ingestion Event Specification (Client & Server $\to$ Gateway)

All telemetry events collected by client trackers (`wsc-tracker.js`, Web Pixel) or server-side hooks (`CheckoutOrderPlacedEvent`, `sales_order_place_after`, `woocommerce_thankyou`) MUST follow this envelope:

### 2.1 Event Envelope Contract

```json
{
  "events": [
    {
      "t": 1756425600,
      "e": "purchase",
      "p": "/checkout/finish",
      "r": "https://www.google.com/",
      "m": { ... }
    }
  ]
}
```

| Field | Type | Required | Description / Rules |
| :--- | :--- | :--- | :--- |
| `t` | `integer` | **YES** | Unix timestamp in seconds (`Math.floor(Date.now() / 1000)`). |
| `e` | `string` | **YES** | Event identifier. Allowed enum: `pageview`, `purchase`, `bot_hit`, `click`, `download`, `time_on_page`, `view_item`, `add_to_cart`, `custom`. |
| `p` | `string` | **YES** | URL path (e.g., `/products/summer-jacket`). Max length: 2048 chars. |
| `r` | `string` | NO | Clean HTTP Referrer URL (or empty string `""`). |
| `m` | `object` | **YES** | Event-specific metadata map (cannot be null, defaults to `{}`). |

---

### 2.2 Event-Specific Metadata Schemas (`m`)

#### A. E-Commerce Order (`e = "purchase"`) — *100% Server-Side Immune*
```json
{
  "order_id": "100249",
  "order_number": "ORD-2026-9921",
  "amount_total": 149.95,
  "amount_net": 126.01,
  "shipping_total": 4.95,
  "tax_total": 23.94,
  "currency": "EUR",
  "items_count": 2,
  "items": [
    {
      "id": "item_901",
      "product_id": "SKU-8821",
      "label": "Organic Linen Shirt",
      "quantity": 1.0,
      "unit_price": 89.95,
      "total_price": 89.95
    },
    {
      "id": "item_902",
      "product_id": "SKU-3310",
      "label": "Cotton Socks",
      "quantity": 2.0,
      "unit_price": 30.00,
      "total_price": 60.00
    }
  ],
  "source": "server_side_shopware6"
}
```

*Rules:*
* `currency`: Must be uppercase 3-letter ISO-4217 string (`EUR`, `USD`, `GBP`, `CHF`).
* `amount_total` & `amount_net`: Float numbers with up to 2 decimal places.
* `source`: Enum identifying origin: `server_side_woocommerce`, `server_side_shopware6`, `server_side_magento2`, `server_side_shopify_webhook`, `client_pixel`.

#### B. Edge Bot Sensor Hit (`e = "bot_hit"`)
```json
{
  "bot_name": "ChatGPT-User",
  "bot_category": "ai",
  "bot_type": "rag",
  "ip": "20.171.207.0",
  "user_agent": "Mozilla/5.0 ... ChatGPT-User/1.0"
}
```

*Rules:*
* `bot_category`: Enum `["ai", "search", "seo_tool", "other"]`.
* `bot_type`: Enum `["rag", "training", "search", "seo_tool", "other"]`.
* `ip`: Client IP address for ephemeral geolocation & subnet lookup (never retained in cleartext in raw reports).

#### C. Active Time on Page (`e = "time_on_page"`)
```json
{
  "duration_ms": 145000,
  "final": true
}
```

---

## 3. The Canonical 3-Card Dashboard Contract (Gateway $\to$ CMS / Shopify)

Whether dispatched via HTTP Webhook to self-hosted CMS/Shops (`wsc_dashboard_reports` table) or written to Shopify Metafields (`wsc_analytics.dashboard_cards`), the JSON payload is **100% standardized**.

```json
{
  "version": "1.5.0",
  "updated_at": 1756425600,
  "project_id": "wsc_sw6_a94f1c",
  "card1": {
    "data_7d": {
      "visitors": 1420,
      "pageviews": 4810,
      "avg_engaged_time": "2m 14s",
      "orders": 38,
      "revenue": 3490.50,
      "currency": "EUR",
      "channels": {
        "ai_pct": 28.4,
        "search_pct": 45.1,
        "social_pct": 12.3,
        "direct_pct": 10.2,
        "referral_pct": 4.0
      }
    },
    "data_30d": {
      "visitors": 5890,
      "pageviews": 19400,
      "avg_engaged_time": "2m 08s",
      "orders": 154,
      "revenue": 14200.00,
      "currency": "EUR",
      "channels": {
        "ai_pct": 26.1,
        "search_pct": 46.8,
        "social_pct": 13.0,
        "direct_pct": 9.8,
        "referral_pct": 4.3
      }
    }
  },
  "card2": {
    "data_7d": {
      "summary": "AI search citations grew +24% WoW from Perplexity and ChatGPT.",
      "actions": [
        {
          "type": "opportunity",
          "title": "High Intent on /collections/summer",
          "description": "ChatGPT-User RAG queries are referencing product prices.",
          "impact": "high"
        }
      ]
    },
    "data_30d": {
      "summary": "Organic search remains the dominant baseline, while AI RAG traffic increased 3.2x over 30 days.",
      "actions": [
        {
          "type": "action",
          "title": "Maintain Product Schema",
          "description": "Keep in-stock and price specifications accurate for AI agents.",
          "impact": "medium"
        }
      ],
      "benchmarks": {
        "status": "active",
        "vertical": "/Shopping/Apparel",
        "vertical_label": "E-Commerce: Fashion & Apparel",
        "nodes_in_benchmark": 142,
        "k_threshold_satisfied": true,
        "metrics": {
          "ai_rag_share": {
            "site_val": 14.2,
            "industry_median": 8.5,
            "industry_p75": 12.1,
            "industry_p90": 17.4,
            "delta_pct": 67.1,
            "rating": "above_average",
            "badge": "🟢 +67.1% vs. Median"
          },
          "engaged_time": {
            "site_val": 42.0,
            "industry_median": 58.0,
            "industry_p75": 72.0,
            "industry_p90": 94.0,
            "delta_pct": -27.5,
            "rating": "below_average",
            "badge": "🔴 -27.5% vs. Median"
          }
        }
      }
    }
  },
  "card3": {
    "data_7d": {
      "human_pct": 89.2,
      "bot_pct": 10.8,
      "ai_rag_hits": 420,
      "ai_training_hits": 1180,
      "search_hits": 3200
    },
    "data_30d": {
      "human_pct": 87.5,
      "bot_pct": 12.5,
      "ai_rag_hits": 1850,
      "ai_training_hits": 4900,
      "search_hits": 13400,
      "benchmarks": {
        "status": "active",
        "vertical": "/Shopping/Apparel",
        "vertical_label": "E-Commerce: Fashion & Apparel",
        "nodes_in_benchmark": 142,
        "metrics": {
          "ai_coverage_pct": {
            "site_val": 81.4,
            "industry_median": 46.0,
            "industry_p90": 78.0,
            "delta_pct": 77.0,
            "rating": "top_10",
            "badge": "🏆 Top 10% Branchen-Pionier"
          }
        }
      }
    }
  }
}
```

---

## 4. Platform-Specific Naming & Storage Matrix

| Platform | Storage Target | Key / Namespace | Sync Frequency | Security / Auth |
| :--- | :--- | :--- | :--- | :--- |
| **Shopify** | Shop Metafields | `wsc_analytics.dashboard_cards`<br>`wsc_analytics.sync_status` | Hourly Cron Push | GraphQL Admin API Access Token |
| **Shopify (Products)** | Product Metafields | `wsc_seo.performance`<br>`wsc_seo.ai_citation_score` | Daily Cron Push | GraphQL `metafieldsSet` |
| **WordPress / WooCommerce** | SQLite WAL | `wsc_dashboard_reports`<br>`queue_temp.sqlite` | 5 Min. Micro-Batch | AES-256-CBC + `X-WSC-Sig` (HMAC-SHA256) |
| **Drupal (7, 8-11)** | SQLite WAL | `wsc_dashboard_reports`<br>`queue_temp.sqlite` | 5 Min. Micro-Batch | AES-256-CBC + `X-WSC-Sig` (HMAC-SHA256) |
| **Joomla (4 & 5)** | SQLite WAL | `wsc_dashboard_reports`<br>`queue_temp.sqlite` | 5 Min. Micro-Batch | AES-256-CBC + `X-WSC-Sig` (HMAC-SHA256) |
| **Shopware 6** | SQLite WAL | `wsc_dashboard_reports`<br>`queue_temp.sqlite` | 5 Min. ScheduledTask | AES-256-CBC + `X-WSC-Sig` (HMAC-SHA256) |
| **Magento 2** | SQLite WAL | `wsc_dashboard_reports`<br>`queue_temp.sqlite` | 5 Min. Crontab | AES-256-CBC + `X-WSC-Sig` (HMAC-SHA256) |

---

## 5. Automated Governance Enforcement & Error Handling

To guarantee 100% adherence:

1. **Gateway Ingestion Verification (`FastAPI` + `Pydantic`)**:
   * Malformed event structures receive HTTP 422 with a structured error response:
     ```json
     {"ok": false, "error": "validation_error", "field": "events[0].e", "msg": "Invalid event type"}
     ```
2. **Shopify GraphQL Metafield Pre-Flight Validation**:
   * Before pushing to `metafieldsSet`, the Gateway runs a strict JSON schema validator.
   * If any numeric field contains `NaN`, `null`, or invalid string types, the pusher aborts and logs an alert instead of corrupting the merchant's Shopify Metafield.
3. **Frontend UI Graceful Degradation**:
   * All Admin UI components (Polaris, Vue.js, Twig, Blade, PHP templates) must implement null-coalescing defaults (`cards?.card1?.data_7d?.visitors ?? 0`).
   * UI components must display empty states without crashing when initial data is synchronizing.
4. **SemVer Backward Compatibility**:
   * Future schema changes (e.g. adding new metrics) must only introduce optional additive fields. Breaking changes require incrementing the major version.
