# WSC Standards & RFC Specifications

<p align="center">
  <strong>Official Specifications, Schemas, and Protocols of the World Search Council (WSC)</strong>
</p>

<p align="center">
  <a href="https://github.com/World-Search-Council/wsc-standards"><img src="https://img.shields.io/badge/RFC-Standard_Track-blue.svg?style=flat-square" alt="RFC Track"></a>
  <img src="https://img.shields.io/badge/Schemas-JSON_Schema_Draft_07-success.svg?style=flat-square" alt="JSON Schema">
  <img src="https://img.shields.io/badge/Status-Active-brightgreen.svg?style=flat-square" alt="Status">
  <img src="https://img.shields.io/badge/License-Apache_2.0_|_MIT-orange.svg?style=flat-square" alt="License">
</p>

---

## 🏛️ Overview

This repository houses the formal RFC specifications, JSON schemas, taxonomy definitions, and scoring protocols established by the **World Search Council (WSC)**.

These standards serve as the interoperability foundation across all WSC client extensions (WordPress, Shopware, Shopify, Magento, Drupal, Joomla) and edge ingestion backends.

---

## 📜 Published RFCs (Request for Comments)

| RFC ID | Title | Scope | Status |
| :--- | :--- | :--- | :---: |
| [**RFC-0001**](rfcs/RFC-0001-universal-analytics-standard.md) | Universal Analytics & Event Ingestion Standard | Canonical event schema, cookieless client rules, ephemeral salt hashing, and batch buffer protocols. | `Standard` |
| [**RFC-0002**](rfcs/RFC-0002-scoring-protocol-spec.md) | Search & Technical Health Scoring Protocol | Normalized 0–100 heuristic models for SEO, Core Web Vitals, Schema.org validation, and SSL lifecycle. | `Standard` |
| [**RFC-0003**](rfcs/RFC-0003-sentinel-and-cron-architecture.md) | Sentinel & Asynchronous Worker Architecture | Cloud mesh architecture, headless worker dispatch, and transient execution schedules. | `Standard` |
| [**RFC-0004**](rfcs/RFC-0004-ecosystem-data-governance.md) | Ecosystem Data Governance & Privacy | Data sovereignty, zero third-party leakage, AES-256 payload encryption, and HMAC authentication. | `Standard` |
| [**RFC-0005**](rfcs/RFC-0005-cross-platform-matrix.md) | Cross-Platform Architecture Matrix | Implementation equivalence guidelines across WordPress, Shopware 6, Shopify, Magento 2, and Drupal. | `Standard` |
| [**RFC-0006**](rfcs/RFC-0006-api-endpoints-reference.md) | Canonical API & Webhook Endpoints Reference | REST and RPC contracts between edge clients and the WSC Gateway. | `Standard` |

---

## 📐 Canonical JSON Schemas

Formal JSON Schemas are published under [`schemas/v1/`](schemas/v1/):

- [`event-envelope.schema.json`](schemas/v1/event-envelope.schema.json) — Defines client event structure, micro-batch payloads, and cryptographic envelopes.
- [`domain-health.schema.json`](schemas/v1/domain-health.schema.json) — Technical health diagnostic output schema.
- [`dashboard-3cards.schema.json`](schemas/v1/dashboard-3cards.schema.json) — Schema for the 3-Card Smart Dashboard metrics and recommendations.
- [`pagespeed-audit.schema.json`](schemas/v1/pagespeed-audit.schema.json) — Core Web Vitals (LCP, CLS, INP, TTFB) diagnostic payload.

---

## 🤝 Contributing & Submitting RFCs

1. Fork this repository.
2. Draft your proposal in `rfcs/RFC-XXXX-<name>.md` following the template.
3. Open a Pull Request with the tag `[RFC Proposal]`.
4. The Technical Committee and community discuss and review the proposal.

---

## 📄 License

The WSC Standards and Schemas are dual-licensed under the [Apache License 2.0](LICENSE) and [MIT License](LICENSE).
