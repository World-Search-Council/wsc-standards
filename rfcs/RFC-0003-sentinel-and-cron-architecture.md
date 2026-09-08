# WSC Sentinel, Member Studio & Cron Self-Healing Architecture

**Standard:** WSC Search & Health System Architecture  
**Release:** v2.2.0 (2026-09-04)  
**Modules:** `wsc-tech-health`, `wsc-pagespeed`, `wsc-core`, `wsc-admin`  
**Status:** Approved & Implemented  

---

## 1. Executive Summary

Mit dem Release v2.2.0 wurden vier Kernbereiche des World Search Council (WSC) WordPress-Ökosystems grundlegend modernisiert und gehärtet:

1. **Dynamische „Domain & Speed Sentinel“-Card im Command Center**:
   Konsolidiert Domain-Infrastrukturdiagnosen (DNS, SSL, Mail, Header) und PageSpeed/RAG-Audits in einer einzigen intelligenten Bento-Grid-Karte mit einem genormten Dual-Marker-Thermometer.
2. **WSC Member vs. Free PageSpeed Studio**:
   Zweispaltiges interaktives Studio mit URL-Pfad-Eingabe und historischer Abrufliste, geschützt durch ein Lizenz-Gate (Free: nur Startseite, Tabelle mit Blur-Overlay; Member: beliebige interne Pfade).
3. **Proaktive Queue & Cron Self-Healing Engine**:
   Vollständige Selbstheilung gegen verwaiste oder fehlende WP-Cron-Ereignisse auf `init`, automatisches Notfall-Entleeren bei Queue-Stau (Admin Safety Valve) sowie 1-Click-Aktions-Buttons.
4. **Tech Health Scoring-Kalibrierung**:
   Beseitigung fälschlicher „Attention“-Warnungen für Standard-Websites (Score 75/100 wird nun als `GOOD`/`healthy` bewertet, IPv6-Neutralität, DMARC-Monitoring-Anerkennung).

---

## 2. Dynamic Domain & Speed Sentinel (Command Center Bento Grid)

### 2.1 Zustandsbasierte Umschaltung (State A vs. State B)

Die Karte passt sich automatisch dem Zustand der Domain-Infrastruktur an:

```
                          [ WSC_Tech_Health::get_dashboard_cards() ]
                                              │
                      ┌───────────────────────┴───────────────────────┐
                      ▼                                               ▼
         Score < 70 ODER SSL ungültig                    Score ≥ 70 UND SSL gültig
             [ STATE A: Diagnostik ]                       [ STATE B: Sentinel Studio ]
    ┌─────────────────────────────────────────┐     ┌─────────────────────────────────────────┐
    │ 🛡️ Domain & Tech Health                 │     │ 🛡️ Domain & Speed Sentinel              │
    │ Badge: 55/100 • Attention               │     │ Badge: 88 CWV • 74 RAG (Optimal)        │
    ├─────────────────────────────────────────┤     ├─────────────────────────────────────────┤
    │ 4-Säulen-Diagnostik:                    │     │ ✓ Domain & Tech Health: 75/100 (GOOD)   │
    │ 🌐 DNS/Net      ✉️ Mail Auth            │     │   DNS, SSL & Mail gesichert             │
    │ 🔒 SSL/TLS      🛡️ Sec-Headers          │     ├─────────────────────────────────────────┤
    ├─────────────────────────────────────────┤     │ Dual-Marker Thermometer Bar             │
    │ Aktionen: [Audit öffnen →] [Re-Scan]    │     │ 🤖 AI & RAG ready (obere Fahne ⬇️)       │
    └─────────────────────────────────────────┘     │ 👤 User UX        (untere Fahne ⬆️)      │
                      │                             ├─────────────────────────────────────────┤
                      ▼                             │ Submetriken: RAG-Latenz, LCP, TBT, CLS  │
            Card 5c bleibt aktiv                    ├─────────────────────────────────────────┤
            (SpeedSentinel separat)                 │ Aktionen: [SpeedSentinel Details →]     │
                                                    │           [Tech Details] [Re-Scan]      │
                                                    └─────────────────────────────────────────┘
                                                                      │
                                                                      ▼
                                                            Option A: Card 5c wird
                                                            automatisch unterdrückt
```

### 2.2 Dual-Marker Thermometer Studio

Beide Kernindikatoren werden auf **einem gemeinsamen 3-Zonen-Balken** visualisiert:

* **Zoneneinteilung**:
  * `🟢 0–33%`: Optimal / Super Turbo
  * `🟡 33–66%`: Risiko / Ausbaufähig
  * `🔴 66–100%`: Träge / Timeout-Gefahr

* **Mathematische Normalisierung & Skalierung**:
  * **RAG-Latenz ($T_{RAG}$)**:
    - $\le 250\text{ms} \rightarrow 0\% \dots 33\%$
    - $250\text{ms} \dots 800\text{ms} \rightarrow 33\% \dots 66\%$
    - $> 800\text{ms} \rightarrow 66\% \dots 96\%$
  * **Core Web Vitals / Largest Contentful Paint (LCP)**:
    - $\le 2.5\text{s} \rightarrow 0\% \dots 33\%$
    - $2.5\text{s} \dots 4.0\text{s} \rightarrow 33\% \dots 66\%$
    - $> 4.0\text{s} \rightarrow 66\% \dots 96\%$
  * **Clamping**: Werte werden auf den Bereich $[8.0\%, 92.0\%]$ begrenzt, um Abschneiden der Fahnen an den Rändern zu verhindern.

* **Kollisionsfreie Nadel-Architektur**:
  * **Obere Nadel**: `🤖 AI & RAG ready` — Nadel sitzt bei `top: -4px`, Fahne bei `bottom: 20px` mit Pfeil nach unten (`border-top`).
  * **Untere Nadel**: `👤 User UX` — Nadel sitzt bei `top: -2px`, Fahne bei `top: 20px` mit Pfeil nach oben (`border-bottom`).
  * Selbst bei identischer Position (z. B. beide bei $28\%$) kollidieren die Textfahnen nicht.

* **WordPress `wp_kses`-Sicherheit**:
  Alle Formatierungs- und Positionierungsregeln (`display: flex`, `transform: translateX(-50%)`, etc.) sind in CSS-Klassen in `assets/css/wsc-dashboard.css` ausgelagert, da der interne WordPress-Style-Sanitizer Inline-Transforms und Flex-Eigenschaften standardmäßig filtert.

---

## 3. WSC Member vs. Free PageSpeed Studio

### 3.1 2-Spalten-Layout

Im Health Hub (`?page=wsc-health&tab=speed`) ist die Benutzeroberfläche in zwei dedizierte Bereiche unterteilt:

1. **Linke Spalte (Steuerung & Audit-Historie)**:
   * **URL-Eingabe-Formular**: Erlaubt die Eingabe relativer Pfade (z. B. `/blog/seo-tipps/`). Die Domain ist fest vorangestellt und kann nicht manipuliert werden.
   * **Audit-Historie-Tabelle**: Chronologische Auflistung vorheriger Abrufe mit Timestamp, URL-Pfad, Performance-Score, RAG-Latenz und Status.
2. **Rechte Spalte (Analyse & Visualisierung)**:
   * **Säule 1: 🤖 RAG-Readiness & KI-Crawler**: $T_{RAG}$ Thermometer, $T_{del}$, $R_{NoJS}$, TTFB, $E_{token}$.
   * **Säule 2: 👤 Menschliche UX & Core Web Vitals**: LCP Thermometer, TBT, CLS, FCP, Speed Index.
   * **Asset-Payload Aufschlüsselung**: HTML, CSS, JS, Bilder, Fonts.

### 3.2 Lizenz-Gate & Sicherheits-Restriktionen

| Feature | Free User (Keine Lizenz) | WSC Member (Verifizierte Lizenz) |
| :--- | :--- | :--- |
| **Erlaubte URLs** | Ausschließlich Startseite (`/`) | Beliebige interne Pfade (`/shop/`, `/kontakt/` etc.) |
| **URL-Formular** | Gesperrt / Deaktiviert | Vollständig editierbar mit Sofort-Validierung |
| **Domain-Einschränkung** | Fest an Site gebunden | Fest an Site gebunden (keine fremden Domains) |
| **Historien-Tabelle** | Mit `filter: blur(4px)` überlagert; Upgrade-Overlay | Vollständig interaktiv mit Re-Check-Links |
| **Upgrade-Link** | `?page=wsc-core-settings&tab=membership` | Nicht erforderlich / Badge `PRO AKTIV` |

---

## 4. Proaktive Queue & Cron Self-Healing Engine

### 4.1 Problemstellung

In Docker-Umgebungen (z. B. mit `ofelia` als Cron-Runner) oder bei Webhostern mit unregelmäßigem Traffic kann der WordPress-Pseudo-Cron verhungern:
* `wp_next_scheduled('wsc_analytics_batch_export')` liefert `false` (Event versehentlich gelöscht).
* Der Zeitstempel des nächsten Laufs liegt Stunden oder Tage in der Vergangenheit („Overdue / Stale Event“).
* Queue-Einträge stauen sich an (z. B. 264 Entries), ohne dass Fehler geloggt werden.

### 4.2 Dreistufige Selbstheilungs-Kaskade

```
                     [ JEDER HTTP-REQUEST / WP-CRON / CLI ]
                                       │
                               Hook: 'init'
                                       ▼
                  [ WSC_Core::ensure_cron_scheduled() ]
                                       │
                     ┌─────────────────┴─────────────────┐
                     ▼                                   ▼
          Event existiert nicht               Event überfällig (> 30 Min)
                     │                                   │
                     └─────────────────┬─────────────────┘
                                       ▼
                    wp_reschedule_event( time(), ... )
                     Sofortige Re-Aktivierung auf JETZT!
                                       │
                                       ▼
                     [ ADMIN-REQUEST SHUTDOWN VALVE ]
                                       │
                     WSC_Core::admin_shutdown_safety_valve()
                                       │
               ┌───────────────────────┴───────────────────────┐
               ▼                                               ▼
     Queue > 50 Einträge                             Queue <= 50 Einträge
     UND letzter Lauf > 15 Min                       ODER letzter Lauf <= 15 Min
               │                                               │
               ▼                                               ▼
     process_batch_export()                               Keine Aktion
     (Notfall-Entleerung vor Seitenende)
```

### 4.3 Interaktive 1-Click Steuerung & WP-CLI

* **Admin UI (`?page=wsc-core-settings&tab=queue_health`)**:
  * `[⚡ Queue jetzt senden]`: AJAX-Endpoint zum manuellen Leeren der SQLite-Queue an das Gateway.
  * `[🔄 Cron neu starten]`: Repariert den Cron-Schedule und setzt das nächste Event auf `time()`.
* **WP-CLI Befehl**:
  ```bash
  wp wsc cron --repair
  ```
  Prüft den Planer, hebt blockierte Locks auf und terminiert den nächsten Lauf neu.

---

## 5. Tech Health Scoring-Protokoll (Kalibrierung)

### 5.1 Anpassung der Bewertungsmetriken

| Prüfpunkt | Altes Verhalten | Neues Verhalten (v2.2.0) | Begründung |
| :--- | :--- | :--- | :--- |
| **DNS: IPv6 (AAAA)** | Fehlen führte zu Punkteabzug | Volle 20/20 Punkte, wenn IPv4 und redundante Nameserver vorhanden sind | Viele Hosting-Umgebungen und Firewalls nutzen stabil reines IPv4; kein Qualitätsmangel für SEO. |
| **Mail: DMARC `p=none`** | 0 Punkte (nur `p=quarantine`/`p=reject`) | 8 von 10 Punkten | `p=none` ist der von RFCs empfohlene Monitoring-Modus für sichere Mail-Zustellung. |
| **Scoring-Klassifikation** | Score 75 = `Attention` (Gelb) | Score ≥ 70 = `GOOD` / `healthy` (Grün `#10B981`) | 75/100 stellt ein absolut solides Setup dar (z. B. wenn lediglich HSTS oder IPv6 fehlen). |
| **X-Frame-Options** | Zeigte `✓ Array` bei Listenwerten | String-Sanitizer extrahiert den konkreten Directive-Wert | Behebung eines Darstellungsfehlers. |
| **Naming / Source** | „Scan Engine: Local PHP Engine“ / „Gateway Playwright Cluster“ | Einheitlich **`Source: WSC API`** | Konsistentes Wording für Benutzer und Audit-Logs. |

---

## 6. Asynchrone Scan- & Abfrage-Architektur (Sentinel 2.2.1)

### 6.1 Problemstellung & Motivation
Die Prüfung autoritativer Nameserver, DNSSEC-Validierungsketten, SPF-Includes, DMARC-Richtlinien und TLS-Zertifikatsketten (via CZ.NIC `dns-crawler` und Gateway-Worker) benötigt netzwerkbedingt zwischen 2 und 8 Sekunden. 
* Synchrones Ausführen blockierte bisher den PHP-FPM-Worker und führte bei langsamen DNS-Nameservern zu HTTP 504 Timeouts.
* Ein unvollständiger oder fehlender lokaler Cache zwang `get_latest_report()` zu einem synchronen Scan beim Seitenaufruf, was die Ladezeit des WordPress-Adminbereichs spürbar verlangsamte.
* SMTP-Port-25-Connect-Versuche führten in Cloud-Umgebungen (Hetzner, AWS, GCP, DigitalOcean) regelmäßig zu Abuse-Sperren durch Hosting-Provider.

### 6.2 Lösungsarchitektur & Workflow
```
[ WordPress Admin UI ]
         │
         │ 1. POST /wp-json/wsc/v1/sentinel/scan (AJAX, non-blocking)
         ▼
[ WSC_Tech_Health ] ──(HMAC Signed POST /v1/sentinel/scan { sync: false })──► [ WSC Gateway 2.0 ]
         │                                                                             │
         │ ◄── HTTP 202 Accepted { status: 'queued', task_id: '...' } ─────────────────┘
         │                                                                             │
         │ Setzt Transient `wsc_tech_health_scan_pending` (120s TTL)                   │
         │                                                                             ▼
[ UI Live-Progress Banner ]                                                   [ RabbitMQ ]
         │                                                                    Queue: `wsc_sentinel_tasks`
         │ 2. Polling: GET /wp-json/wsc/v1/sentinel/report                             │
         │    (Intervall: 2s mit Step-Feedback)                                        ▼
         │                                                                   [ Dedicated Sentinel Worker ]
         │                                                                   - CZ.NIC dns-crawler
         │                                                                   - Strikte DNS-Record-Prüfung
         │                                                                   - KEIN SMTP Port 25 Connect!
         │                                                                   - DeepDiff Drift Tracking
         │                                                                             │
         ▼                                                                             ▼
[ Webhook Ingestion ] ◄── Signed POST /wp-json/wsc/v1/insights { report_type: 'domain_health' }
WSC_Core::store_insights()
- Normalisiert & speichert Report
- Löscht Transient `wsc_tech_health_scan_pending`
- Aktualisiert SQLite & WP Options
         │
         ▼
[ UI Auto-Reload & Status Update ]
```

### 6.3 Sicherheitsrichtlinien & Mail-Checks
* **Kein Port 25 Connect**: Zur Vermeidung von Provider-Abuse-Meldungen und Firewall-Drops werden Mail-Prüfungen ausschließlich auf DNS-Ebene durchgeführt:
  - MX-Records (Prioritäten, Hostnamen, Redundanz)
  - TXT-Records (`v=spf1 ...`, Rekursionstiefe $\le 10$, Include-Syntax)
  - DMARC-Records (`_dmarc.domain.tld`, Policy `p=reject`/`quarantine`/`none`, Alignment)
  - Bekannte Mail-Provider-Erkennung (Google Workspace, Microsoft 365, Proton, Mailgun, SendGrid etc.)

### 6.4 Non-Blocking Page-Rendering
* `get_latest_report(bool $allow_blocking_scan = false)`:
  1. Liefert zuerst den persistenten Options- oder SQLite-Report.
  2. Falls kein lokaler Report existiert, erfolgt eine ultraschnelle Abfrage des Gateway-Redis-Caches (`/v1/sentinel/health/{project_id}`, Timeout $\le 2\text{s}$).
  3. Falls noch kein Scan vorliegt, wird ein solider Baseline-Report zurückgegeben und ein Hintergrund-Event (`wsc_weekly_tech_health_scan`) für den nächsten Lauf geplant. Die Seite lädt ohne jegliche Verzögerung (< 50ms).

### 6.5 Frontend Controller (`assets/js/wsc-dashboard.js`)
* `initSentinelStudio(restBase, restNonce)`:
  - Deaktiviert den Scan-Button, zeigt den `#wsc-sentinel-loading`-Banner und triggert den REST-Endpunkt.
  - Zeigt zyklisch 4 Live-Schritte an:
    1. *„Querying authoritative nameservers and DNSSEC records...“*
    2. *„Evaluating SPF includes, DMARC policies and MX routing...“*
    3. *„Validating SSL/TLS certificates and HSTS security headers...“*
    4. *„Analyzing infrastructure drift and changes...“*
  - Reagiert nahtlos auf fertige Webhooks oder pollt den Gateway-Redis-Cache.
  - Führt bei Abschluss einen sauberen Seiten-Reload durch.

---

## 7. Verifikations-Checkliste

- [x] **Docker Volume Sync**: `rsync -av` zwischen Quell-Repo und Container-Volume durchgeführt.
- [x] **PHP Syntax Linter**: `docker exec wp_app php -l` für alle geänderten Core- und Dashboard-Dateien bestanden.
- [x] **Bento-Grid Suppression**: Option A verifiziert (Card 5c wird bei `merged_speed => true` ausgeblendet).
- [x] **CSS-Isolierung**: Alle Positionierungsregeln der Thermometer-Nadeln werden sauber über CSS gerendert.
- [x] **Cron Self-Healing Test**: `wp wsc cron --repair` und automatischer Re-Schedule im Container validiert.
- [x] **Asynchrone Sentinel Pipeline**: Non-blocking `POST /wp-json/wsc/v1/sentinel/scan` mit HTTP 202, RabbitMQ-Queuing und Webhook-Ingestion validiert.
- [x] **Port 25 Sicherheit**: Verifiziert, dass keine SMTP Connect-Checks auf Port 25 stattfinden.
- [x] **Multilanguage Sync**: POT, PO und MO (DE/ES) synchronisiert und kompiliert (1.939 Strings).
