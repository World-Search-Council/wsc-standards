# WSC Search & Health Scoring Protocol (v1.0) Specification

**Standard:** WSC Search & Health Scoring Protocol  
**Version:** 1.0.0  
**Publisher:** World Search Council (WSC) Standardisation Committee  
**Status:** Approved Standard  

---

## 1. Executive Overview

Traditional SEO tools evaluate websites via static, monolithic checklists. An agency can fill in basic meta tags, write image alt attributes, and present a client with an artificial „95% SEO Score“ — even if zero search crawlers index the site and zero real human visitors convert.

The **WSC Search & Health Scoring Protocol (v1.0)** establishes a dynamic, uncompromised, and vendor-neutral evaluation framework built upon two foundational architectural principles:

1. **The 3+1 Dimensions Architecture**: 1 Infrastructure Foundation (*Technical Health*) and 3 Goal-Directed Channels (*AI & GEO*, *Social & Engagement*, *Organic Search*).
2. **The 3-Phase Reality Cascade (0–100 Points)**:
   - **Phase 1: Readiness & Static Audits (0–50 Pkt)**: Configuration, On-Page, Schema.org, and LLM readiness.
   - **Phase 2: Bot-Crawl & Signal Proof (51–70 Pkt)**: Verification of real AI and Search engine crawlers without 4xx/5xx errors.
   - **Phase 3: Real User Traffic & Retention Gate (71–100 Pkt)**: Hard-gated top tier unlocked exclusively through sustained real human visits over rolling time windows (7 / 30 / 90 days).

---

## 2. The 3+1 Dimensions Taxonomy

```
                            [ 🏛️ OVERALL SITE SCORE ]
                                        ▲
        ┌───────────────────────────────┼───────────────────────────────┐
        ▼                               ▼                               ▼
[ 🤖 AI & GEO (0–100) ]     [ 📱 Social (0–100) ]     [ 🔍 Organic Search (0–100) ]
- /llms.txt & RAG-Structure - OpenGraph & X-Cards      - Meta-Templates & IndexNow
- Schema.org Entities       - Social Share Previews    - LOVE Internal Link Graph
- AI-Crawler Hits (51–70)   - Social Scrapers (51–70)  - Search Bot Hits (51–70)
- AI Traffic (71–100)       - Social Traffic (71–100)  - Organic Traffic (71–100)
        ▲                               ▲                               ▲
        └───────────────────────────────┴───────────────────────────────┘
                                        ▲
                                        │ (Hard Gate / Blocker-Cap: min(Score, 40))
                        [ 🛡️ FUNDAMENT: Technical Health ]
                        - SSL Certificate & Expiration Countdown
                        - DNSSEC, DMARC, SPF, Nameserver Redundancy
                        - HTTP Security Headers (HSTS, CSP, X-Frame)
                        - Server Response TTFB & Uptime Probe
```

### 2.1 Foundation: Technical Health
Technical infrastructure determines whether search engines and AI agents can reach the domain.
* **Scope**: Global / Site-wide (`scope => 'site'`).
* **Blocker Impact**: If any technical metric flagged as `is_blocker => true` enters `status === 'critical'` (e.g. expired SSL, missing MX/DMARC, or DNS failure), the **global score and all 3 channels are capped at $\min(S, 40)$**.

### 2.2 Target Channel: AI & GEO Readiness
Measures optimization for Large Language Models (LLMs), RAG systems (Retrieval-Augmented Generation), and AI search engines (ChatGPT, Perplexity, Claude, Gemini).
* **Audits (0–50)**: Schema.org structured data depth, `/llms.txt` and `/llms-full.txt` standard generation, information density, semantic chunk clarity.
* **Crawl Proof (51–70)**: Verified crawl activity by GPTBot, ClaudeBot, PerplexityBot, Applebot without server timeouts.
* **Traffic Proof (71–100)**: Direct referral traffic from AI search engines and conversational interfaces.

### 2.3 Target Channel: Social & Engagement
Measures snippet visibility and conversion potential across social media platforms (LinkedIn, Facebook, X, Pinterest).
* **Audits (0–50)**: OpenGraph (`og:image`, `og:title`, `og:description`), Twitter Card metadata (`summary_large_image`), image dimensions (1200x630px).
* **Crawl Proof (51–70)**: Verified crawl activity by FacebookExternalHit, LinkedInBot, TwitterBot.
* **Traffic Proof (71–100)**: Verified social media inbound visitors and engagement.

### 2.4 Target Channel: Organic Search
Measures traditional search engine discoverability, on-page architecture, and crawl equity.
* **Audits (0–50)**: Dynamic title/meta templates, single `<h1>` hierarchy, image `alt` coverage, internal link distribution (LOVE engine), XML sitemaps, IndexNow keys.
* **Crawl Proof (51–70)**: Verified crawl hits from Googlebot, Bingbot, YandexBot, SeznamBot.
* **Traffic Proof (71–100)**: Organic search visits and session retention.

---

## 3. The 3-Phase Reality Cascade Mathematics

A site score $S \in [0, 100]$ is computed through a cumulative, gated 3-phase formula:

$$S_{\text{total}} = S_{\text{readiness}} + S_{\text{crawl\_proof}} + \left( 30 \times U_{\text{time}} \times \text{Gate}_{\text{active}} \right)$$

### 3.1 Phase 1: Readiness & Audit ($S_{\text{readiness}} \in [0, 50]$)
Calculated from all active declared static audit rules within dimension $D$:

$$S_{\text{readiness}} = \frac{50 \times \sum_{i \in \text{ActiveRules}} (v_i \cdot w_i)}{100 \times \sum_{i \in \text{ActiveRules}} w_i}$$

Where:
* $v_i \in [0, 100]$ is the measured metric raw score.
* $w_i \in [0, 100]$ is the active weighting factor.
* Deactivated rules ($w_i = 0$ or `enabled = false`) are completely excluded from numerator and denominator, preserving mathematical scaling.

### 3.2 Phase 2: Bot-Crawl & Signal Proof ($S_{\text{crawl\_proof}} \in [0, 20]$)
Rewards domains where bots successfully fetch and parse content without 4xx/5xx errors:

$$S_{\text{crawl\_proof}} = 20 \times \min\left(1.0, \frac{\text{BotHits}_{\text{24h}}}{\text{Threshold}_{\text{bots}}}\right) \times \left(1.0 - \text{ErrorRate}_{\text{4xx/5xx}}\right)$$

### 3.3 Phase 3: Real User Traffic & Retention Gate ($S_{\text{user}} \in [0, 30]$)
The upper tier ($71–100$) remains **strictly locked** ($\text{Gate}_{\text{active}} = 0$) until verified real human visitor sessions are recorded by the WSC Tracker:

$$\text{Gate}_{\text{active}} = \begin{cases} 1, & \text{if } \text{HumanSessions}_{\text{30d}} \ge \text{Threshold}_{\text{traffic}} \\ 0, & \text{otherwise} \end{cases}$$

#### Temporal Decay & Consistency Factor ($U_{\text{time}} \in [0, 1.0]$):
To prevent artificial traffic spikes from gaming the score, $U_{\text{time}}$ requires sustained human traffic:
* **Week 1 sustained**: $U_{\text{time}} = 0.33$
* **Month 1 sustained**: $U_{\text{time}} = 0.67$
* **Quarter sustained (90 days)**: $U_{\text{time}} = 1.00$
* **Decay Mechanism**: If real user traffic ceases, $U_{\text{time}}$ gradually decays to $0$, returning the score to Phase 2 ($\le 70$).

---

## 4. Hook Specifications & Data Contracts

### 4.1 Filter: `wsc_register_scoring_rules`
Executed to discover and register all available rules from sub-modules and 3rd-party providers.

```php
apply_filters( 'wsc_register_scoring_rules', array $rules ): array
```

#### Rule Schema:
```php
$rules['unique_rule_id'] = [
    'module'         => (string) 'wsc-ai-ready',       // Module slug
    'dimension'      => (string) 'ai',                 // 'technical' | 'ai' | 'social' | 'organic'
    'phase'          => (int)    1,                    // 1 = Readiness, 2 = Bot Proof, 3 = Traffic
    'scope'          => (string) 'page',               // 'site' | 'page'
    'label'          => (string) __( 'Rule Name', 'text-domain' ),
    'description'    => (string) __( 'Rule explanation...', 'text-domain' ),
    'default_weight' => (int)    80,                   // 0 to 100
    'is_blocker'     => (bool)   false,                // If true & critical => caps total score to 40
    'async'          => (bool)   false,                // Heavy/network check flag
];
```

### 4.2 Filter: `wsc_register_scoring_metrics`
Executed to collect evaluated measurements for declared rules.

```php
apply_filters( 'wsc_register_scoring_metrics', array $metrics, ?int $post_id ): array
```

#### Metric Schema:
```php
$metrics[] = [
    'rule_id'      => (string) 'unique_rule_id',
    'raw_value'    => (int)    100,                  // 0 to 100
    'status'       => (string) 'good',               // 'good' | 'attention' | 'critical'
    'message'      => (string) __( 'Explanatory status message', 'text-domain' ),
    'action_url'   => (string) admin_url( '...' ),   // Direct link to fix
    'action_label' => (string) __( 'Fix Issue', 'text-domain' ),
];
```

---

## 5. Storage Governance & Sparse Overrides

To prevent submodule updates from being clobbered by database artifacts, WordPress option `wsc_scoring_overrides` stores **only user modifications (deltas)**:

```json
{
  "meta_og_image": {
    "enabled": false
  },
  "tech_health_dmarc": {
    "weight": 25
  }
}
```

At runtime, `WSC_SEO_Score` executes an `array_replace_recursive()` over the hook-registered defaults and stored overrides.
