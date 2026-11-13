# WSC Standards & RFC Specifications (`wsc-standards`)

<p align="center">
  <strong>Official RFC Specifications, JSON Schemas & Interoperability Protocols of the World Search Council (WSC)</strong>
</p>

<p align="center">
  <a href="https://www.worldsearchcouncil.org/standards"><img src="https://img.shields.io/badge/Live_Hub-worldsearchcouncil.org%2Fstandards-0F172A?style=flat-square" alt="Live Standards Hub"></a>
  <a href="https://github.com/World-Search-Council/wsc-standards"><img src="https://img.shields.io/badge/RFC-Standard_Track-2563EB.svg?style=flat-square" alt="RFC Track"></a>
  <img src="https://img.shields.io/badge/Schemas-JSON_Schema_Draft_07-10B981.svg?style=flat-square" alt="JSON Schema">
  <img src="https://img.shields.io/badge/Status-Active-brightgreen.svg?style=flat-square" alt="Status">
  <img src="https://img.shields.io/badge/License-Apache_2.0_|_MIT-orange.svg?style=flat-square" alt="License">
</p>

---

## 🏛️ Overview

This repository houses the formal **Request for Comments (RFC)** specifications, canonical JSON Schemas, bot taxonomy definitions, and scoring protocols ratified by the **[World Search Council (WSC)](https://www.worldsearchcouncil.org)**.

All ratified specifications are also published interactively on the live **[WSC Standards & Working Group Hub (`www.worldsearchcouncil.org/standards`)](https://www.worldsearchcouncil.org/standards)**, where WSC Members participate in Working Groups (`WG-01..WG-05`), submit formal section amendments, and cast ratification ballots.

---

## 📜 Published Core RFCs

| RFC ID | Title | Scope | Live Spec | Status |
| :--- | :--- | :--- | :---: | :---: |
| [**RFC-0001**](rfcs/RFC-0001-universal-analytics-standard.md) | Universal Analytics & Event Ingestion Standard | Canonical event schema, cookieless client rules, rotating daily salt hashing, and SQLite WAL micro-batching. | [Read Live](https://www.worldsearchcouncil.org/standards/RFC-0001) | `Standard` |
| [**RFC-0002**](rfcs/RFC-0002-scoring-protocol-spec.md) | Search & Technical Health Scoring Protocol | Normalized 0–100 heuristic models for SEO, Dual-Thermometer RAG/CWV (`S_RAG`, `T_RAG`), Schema.org, and TLS lifecycle. | [Read Live](https://www.worldsearchcouncil.org/standards/RFC-0002) | `Standard` |
| [**RFC-0003**](rfcs/RFC-0003-sentinel-and-cron-architecture.md) | Sentinel & Asynchronous Worker Architecture | Cloud RabbitMQ + DLX worker mesh, Playwright headless dispatch, and Port-25-free authoritative DNS scanning. | [Read Live](https://www.worldsearchcouncil.org/standards/RFC-0003) | `Standard` |
| [**RFC-0004**](rfcs/RFC-0004-ecosystem-data-governance.md) | Ecosystem Data Governance & Privacy | Zero-PII sovereignty, AES-256-CBC payload encryption, HMAC-SHA256 (`X-WSC-Sig`), and $k \ge 25$ benchmark anonymity. | [Read Live](https://www.worldsearchcouncil.org/standards/RFC-0004) | `Standard` |
| [**RFC-0005**](rfcs/RFC-0005-cross-platform-matrix.md) | Cross-Platform Architecture Matrix | Canonical 7-Card Bento feature parity across WordPress, Shopware 6, Shopify, Magento 2, Drupal 8–11, Drupal 7, and Joomla 4/5. | [Read Live](https://www.worldsearchcouncil.org/standards/RFC-0005) | `Standard` |
| [**RFC-0006**](rfcs/RFC-0006-api-endpoints-reference.md) | Canonical API & Webhook Endpoints Reference | REST, webhook, licensing, and Model Context Protocol (`/api/v1/mcp`) contracts across edge clients, Gateway, and Portal. | [Read Live](https://www.worldsearchcouncil.org/standards/RFC-0006) | `Standard` |

---

## 📐 Canonical JSON Schemas (`schemas/v1/`)

Formal Draft-07 JSON Schemas are maintained in [`schemas/v1/`](schemas/v1/) and served over HTTPS with `Content-Type: application/schema+json` at `https://www.worldsearchcouncil.org/schemas/v1/`:

| Schema File | Canonical `$id` / Live URL | Description |
| :--- | :--- | :--- |
| [`event-envelope.schema.json`](schemas/v1/event-envelope.schema.json) | [`/schemas/v1/event-envelope.schema.json`](https://www.worldsearchcouncil.org/schemas/v1/event-envelope.schema.json) | Client event structure, micro-batch payloads, and cryptographic envelopes. |
| [`domain-health.schema.json`](schemas/v1/domain-health.schema.json) | [`/schemas/v1/domain-health.schema.json`](https://www.worldsearchcouncil.org/schemas/v1/domain-health.schema.json) | Port-25-free Domain Sentinel diagnostic output (DNSSEC, SSL, Mail DNS, 6/6 Security Headers, Drift). |
| [`pagespeed-audit.schema.json`](schemas/v1/pagespeed-audit.schema.json) | [`/schemas/v1/pagespeed-audit.schema.json`](https://www.worldsearchcouncil.org/schemas/v1/pagespeed-audit.schema.json) | SpeedSentinel 2.0 Dual-Thermometer payload (🤖 RAG Readiness + 👤 Human Core Web Vitals). |
| [`dashboard-3cards.schema.json`](schemas/v1/dashboard-3cards.schema.json) | [`/schemas/v1/dashboard-3cards.schema.json`](https://www.worldsearchcouncil.org/schemas/v1/dashboard-3cards.schema.json) | Core dashboard report contract extended across the 7-Card Bento Suite (`card1_traffic`..`domain_health`). |

---

## 🤝 Contributing & Submitting RFC Proposals

1. Fork this repository and draft your proposal in `rfcs/RFC-XXXX-<slug>.md`.
2. Open a Pull Request prefixed with `[RFC Proposal]`, or submit section amendments via the **[WSC Member Governance Console](https://www.worldsearchcouncil.org/member/governance)**.
3. Proposals undergo technical review in the responsible Charter Working Group (`WG-01..WG-05`) prior to Assembly ratification.

---

## 📄 License

WSC Standards and JSON Schemas are dual-licensed under the [Apache License 2.0](LICENSE) and [MIT License](LICENSE).
