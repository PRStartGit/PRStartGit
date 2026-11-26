# Zynthio Stay: PMS + Channel Manager Plan

**Status:** Draft v1, 26 November 2026
**Owner:** Attila Juhasz (Zynthio Ltd)
**Scope:** A hotel property management system (PMS), channel manager, revenue engine, housekeeping/linen app and admin console, delivered as new modules inside the existing Zynthio platform.

---

## 1. Why now, and why Zynthio can win

Zynthio already runs the operational layer UK hospitality sites live on: rotas, HR and compliance, EPOS with till and kitchen screens, KPI dashboards, reputation tracking, a native staff app, and card payments through the Dojo partnership. Hotels buy a PMS, a channel manager, an EPOS, a payments provider, a rota tool and a housekeeping app from five or six vendors and then pay integrators to glue them together. The WithUS (University of Sheffield) opportunity is exactly that shape: 18 restaurants, 2 hotels, a conference venue and a production kitchen, one buyer.

The wedge is **one ledger across rooms, food and beverage, events and retail**, with staffing cost already inside the same system. No mid-market PMS has the rota and labour-budget side. No rota product has rooms. Zynthio can have both.

Target customer for the first 24 months: UK independent hotels, inns and pubs-with-rooms, 6 to 120 rooms, plus university and venue accommodation. Not resorts, not 300-room city hotels, not vacation rentals as a primary market (although the aggregator route supports them for free).

---

## 2. Product scope

### 2.1 Core PMS ("Rooms")

**Inventory and configuration**
- Properties, buildings, floors, room types, physical rooms, bed configurations, room features, connecting rooms, out-of-order and out-of-service with reason codes.
- Rate plans: BAR, non-refundable, advance purchase, packages (bed and breakfast, dinner inclusive), corporate and negotiated rates, derived rates (percentage or fixed offset from a parent), per-occupancy and per-length-of-stay pricing, child and extra-bed policies.
- Restrictions per room type, rate plan and date: min and max stay, closed to arrival, closed to departure, stop-sell, release period.
- Policies: cancellation, deposit and guarantee, no-show, check-in and check-out times, early and late fees, pets, smoking.
- Taxes and fees: UK VAT at 20 percent on accommodation, reduced or zero-rated items where applicable, city or tourism levies (the Scottish visitor levy is live from 2026 in several councils and must be a configurable per-night or percentage charge), service charge handling for tronc.

**Reservations**
- Individual, multi-room and group bookings, allotments and room blocks for events and conferences, waitlists.
- Reservation lifecycle: enquiry, tentative, confirmed, checked in, checked out, cancelled, no-show, with full audit history.
- Room assignment: manual, drag-and-drop on a tape chart, and auto-assign by preference and housekeeping status.
- Guest profiles: contact, preferences, nationality and ID capture, GDPR consent, marketing preferences, stay history, blacklist, loyalty tags, company and travel agent profiles with commission rules.
- Front desk: arrivals and departures boards, walk-ins, room moves, extensions, early departures, split and merge folios, key encoding hooks (Salto, Assa Abloy, dormakaba, Onity later).
- Communications: confirmation, pre-arrival, in-stay and post-stay email and SMS templates, online pre-check-in, digital registration card with e-signature, review request via the existing reputation module.

**Folios, payments and ledger**
- Folio per reservation with multiple windows (guest, company, third party), itemised postings with department, revenue category, tax code and user stamp.
- Posting sources: room and tax (nightly auto-post), EPOS room charges, extras, packages, manual postings, adjustments, refunds, rebates, allowances.
- Payments via Dojo: card present on terminal (pay at desk, pay at table posted to room), pre-authorisation at check-in with incremental auth and capture at check-out, tokenised card on file for no-show and incidentals, pay-by-link for deposits and remote balances, Apple Pay and Google Pay on the booking engine, cash, bank transfer, vouchers, virtual cards from OTAs charged on the correct date.
- Deposits, prepayments and credit limits with alerts.
- Ledgers: guest ledger, city ledger (accounts receivable for companies and agents), deposit ledger, advance deposit liability, with ageing and statements.
- Night audit: automated end of day with pre-checks (unposted room charges, pending arrivals, open shifts on EPOS), room and tax posting, rollover, daily revenue, manager's report, trial balance, variance alerts. Run as a scheduled job with manual override.
- Invoicing: UK-compliant VAT invoices and credit notes, sequential numbering per property, proforma, split by payer, PDF and e-mail.
- Accounting exports: Xero, Sage, QuickBooks mappings by revenue category and tax code; CSV fallback.

**Reporting**
- Occupancy, ADR, RevPAR, TRevPAR, GOPPAR when labour is included, pickup and pace, booking window, channel mix and cost of acquisition, cancellation and no-show rates, segment and source analysis, guest geography, forecast versus budget.
- Labour versus rooms revenue by hour and day using the existing rota and EPOS data (the differentiator).
- Housekeeping productivity, linen cost per occupied room, maintenance backlog.

### 2.2 EPOS integration (room charges and single ledger)

Zynthio EPOS is already live. The PMS adds:
- "Charge to room" tender on every till and handheld: search by room number or guest name, show only in-house guests, enforce credit limit and guest consent, optional PIN or signature capture, itemised post to folio with outlet and server.
- Reverse flow: folio lookup from the till, void or refund a room charge with manager approval.
- Package consumption: breakfast and dinner inclusive covers are redeemed on the till against the booking and not charged twice.
- Guest-facing: QR order to room, in-room dining with room charge, bar tabs closed to a room.
- Conference and events: function sheets, event folios, room hire, delegate packages, bar tabs by event, post to master account.
- Daily reconciliation: EPOS Z report versus PMS postings, with exceptions queue.
- Shared item, tax and department master data so revenue categories line up in one chart of accounts.

### 2.3 Channel manager ("Connect")

**Functions**
- Two-way sync of availability, rates and restrictions (ARI) per room type and rate plan to every connected channel, pushed on change and reconciled on a schedule.
- Reservation delivery: new, modified and cancelled bookings pulled into the PMS in near real time, mapped to the right room type, rate plan and guest profile, with OTA virtual card details stored in the tokenised vault.
- Channel mapping UI: map Zynthio room types and rate plans to channel products, detect unmapped products, parity and markup rules per channel (for example Booking.com plus 0 percent, Expedia plus 2 percent, direct minus 5 percent).
- Overbooking protection: last-room logic, per-channel allotments, stop-sell cascades when inventory hits zero, failed push retry queue with alerting.
- Health: per-channel sync status, last successful push, error log, latency, booking success rate (Booking.com and Expedia score partners on this).
- Direct booking engine: mobile-first, white-labelled per property, rate comparison widget against OTAs, promo codes, upsells, packages, Dojo checkout, metasearch feed (Google Hotel Ads free booking links first).
- Metasearch and GDS later: Google Hotel Ads, Trivago, TripAdvisor; GDS via a switch (Derbysoft or DHISCO) only when a corporate-heavy customer needs it.

**Connectivity strategy: build the channel manager, connect direct**

Zynthio builds and owns the whole channel manager: the availability and rate engine, channel mapping, the ARI outbox, inbound reservation processing and health monitoring. Channels connect through adapters behind one interface.

- **Direct from day one:** Booking.com Connectivity APIs and Expedia Group Lodging Connectivity APIs, both free to software partners. Register as a connectivity provider on both developer portals immediately; certification needs test requests against their sandboxes and, before go-live, at least one live property, so the first design partner is also the certification property. Google Hotel Ads free booking links via a price feed.
- **iCal two-way sync** for Airbnb, Vrbo and similar channels that do not offer API access to new partners. iCal carries availability only (blocked dates), so bookings from those channels are entered from the channel's notification email or imported, and rates are managed in the channel's own extranet. This is the honest limit while Airbnb's partner intake is closed.
- **Aggregator as optional adapter only.** A white-label aggregator (Channex or similar) can be plugged in later for long-tail channels (Agoda, Hostelworld, regional OTAs, GDS) if a customer needs them. No dependency on it for launch.

Requirements observed for the direct route: Booking.com requires certification against their test endpoints, a self-assessment and a PII compliance review, and expects partners to keep up with API changes; partners start at Standard tier and move up (Preferred, Premier, Premier Plus from 2026) on connection quality and booking success rate. Expedia requires a Property Distribution Software system type, acceptance of the API licence, Availability and Rates API with Expedia Traveler Preference, Booking APIs with virtual card and Value Add Promotion support, and PCI compliance where applicable, with end-to-end certification by their EQC team; the programme runs on an annual cycle.

### 2.4 Revenue engine ("Yield")

**Rules**
- Rule builder with conditions (date range, day of week, lead time, occupancy percentage, pickup in last N days, length of stay, channel, room type, event calendar) and actions (set rate, adjust by percentage or amount, open or close restrictions, change min stay, toggle rate plans).
- Occupancy-based price ladders per room type (for example 0 to 40 percent at BAR minus 10, 70 to 85 percent at BAR plus 15, 85 plus at BAR plus 30).
- Event and seasonal calendars, with a UK bank holiday and local events feed (festivals, football fixtures, graduations for university customers).
- Guardrails: floor and ceiling per room type, maximum daily change, manual override with expiry, approve-before-publish mode.

**Promotions**
- Promo codes, early bird, last minute, length-of-stay discounts, mobile-only, member-only, secret deals, geo-targeted, package bundles, add-on upsells (late check-out, breakfast, parking, dog, spa) with inventory caps.
- Channel promotions mirrored to Booking.com Deals and Expedia promotions through the connectivity layer where the API allows.
- Gift vouchers and prepaid stays sold online, redeemed on folio.

**Competitor analysis and dynamic pricing**
- Rate shopping: a comp set of 3 to 10 competitors tracked for the next 90 to 365 days. Built as a framework with a provider interface from day one. Launch sources: manual entry and import of competitor rates from the property's own Booking.com extranet market insights. A licensed feed (OTA Insight, RateGain, Aggregate Intelligence or similar) is added later as one adapter. No scraping of Booking.com or Google: it breaches their terms, is blocked quickly, and would put the Booking.com connectivity partner application at risk.
- Market position dashboard: rank versus comp set per date, price index, parity violations, comp set occupancy proxy (sold-out signals).
- Demand signals: own pickup and pace, comp set movement, events, weather, search demand from Google Hotel Ads, day-of-week seasonality.
- Dynamic pricing model: start with transparent rules plus comp set anchors (explainable, hoteliers trust it). Phase two adds a forecasting model (gradient boosted or similar) per property trained on own history, producing a recommended rate per room type per date with a confidence band and plain-English reasons. Modes: advise, approve, autopilot with guardrails.
- Group and event quotes: displacement analysis showing what a block costs in forgone transient revenue.

### 2.5 Housekeeping and linen app ("Keep")

Delivered inside the existing Zynthio staff app (iOS live, Android in progress) as a role-based module, so housekeepers and maintenance already clocked in through Zynthio see their work.

**Housekeeping**
- Live room status board: dirty, clean, inspected, occupied, vacant, DND, out of order; auto-updated by check-in, check-out and night audit.
- Task generation: departure clean, stay-over service, turndown, deep clean cycle, linen change frequency per rate plan or stay length; priority by expected arrival time.
- Assignment: by housekeeper, floor or section, with credit minutes per task type for fair workload and productivity reporting; supervisor inspection checklists with photo evidence and fail reasons.
- Guest requests and extras (cot, towels, late service) from front desk straight to the housekeeper's phone.
- Lost and found with photos and storage location; maintenance tickets with photo, priority and parts, routed to maintenance or an external contractor; preventive maintenance schedules.
- Minibar and in-room consumables count with auto-post to folio.
- Offline-first: works in corridors with poor signal, syncs when back online. Push notifications and rota-aware (only assign to people on shift).

**Linen and stock management**
- Linen items and par levels per room type and per floor pantry.
- Counts: clean issued, dirty collected, sent to laundry, returned, rejected or condemned; variance reports and shrinkage tracking; in-house laundry cycles or external laundry with delivery note reconciliation and invoice matching.
- Cost per occupied room, reorder alerts, supplier management; reuse the EPOS stock module where sensible.
- Guest amenities and cleaning chemicals stock with periodic counts.

### 2.6 Admin console and configuration

Everything above is driven from a fully manageable admin site, consistent with the existing Zynthio organisation and site model.

- Multi-tenant hierarchy: organisation, brand, property, outlet; users with roles and granular permissions (front desk, night audit, housekeeping, revenue, finance, group admin), SSO for larger customers, two-factor, session and device management.
- Configuration without code: room types and rooms, rate plans and derivation, policies, taxes, charge codes and chart of accounts, folio templates, document templates (confirmations, invoices, registration cards) with a visual editor and merge fields, email and SMS templates, languages and currencies, check-in and check-out times, housekeeping task types and credits, linen items and pars, channel mappings, pricing rules, comp sets.
- Feature flags per property, module entitlements tied to billing, sandbox or training mode with sample data.
- Audit log of every configuration change and every financial posting, exportable.
- Data tools: CSV import of rooms, rates, guests and future reservations for migration from the incumbent PMS; export everything; scheduled report delivery.
- Integrations hub: Dojo, accounting (Xero, Sage, QuickBooks), door locks, guest Wi-Fi, TV and phone, review platforms, Google Business Profile, webhooks and a public REST API with API keys and rate limits.
- Support: in-app ticketing already exists; add impersonation with consent for support staff, status page and changelog.
- Billing: per-room per-month subscription per module, usage metering for channel connections, invoices generated by the platform itself.

### 2.7 Guest journey: online check-in, check-out, payments and refunds

**Online check-in (pre-arrival)**
- Triggered automatically 24 to 72 hours before arrival by email, SMS or WhatsApp, with a secure link (no app download). Works for direct and OTA bookings.
- Steps: confirm details, add co-guests, capture ID and nationality where required, accept terms and house rules, choose arrival time, pick upsells (early check-in, breakfast, parking, room upgrade), add a card for incidentals with a Dojo pre-authorisation, e-sign the registration card.
- Outcome: reservation flagged "ready", room pre-assigned if housekeeping status allows, digital key or door code issued where locks support it, front desk sees a one-tap confirm. Kiosk mode on a tablet at reception for the same flow.
- Staff benefit: arrivals board shows who is pre-checked, so a 10-room inn can run unattended late arrivals with a code and a key safe.

**Express check-out**
- Night before departure: folio summary sent to guest, with a "check out now" link. Guest reviews charges, disputes a line (opens a ticket for front desk), adds a tip to tronc where offered, chooses invoice recipient (self or company), and confirms. Card on file is captured for the balance; a VAT invoice PDF is emailed.
- Room flips to dirty automatically and a departure clean is queued in the housekeeping app. Late check-out can be bought in the same flow if availability allows.
- Front desk still has a full manual check-out with cash, split payments and company billing to city ledger.

**Payments**
- Full lifecycle on Dojo: pre-auth at booking or check-in, incremental authorisation when the folio grows past the hold, capture at check-out, card-on-file charges for no-shows and post-departure incidentals with guest-consented terms, pay-by-link for deposits and group balances, terminal payments at desk and on handhelds, Apple Pay and Google Pay on the booking engine, OTA virtual cards charged on the permitted date.
- Every payment is a ledger entry linked to its folio, with the acquirer reference, so settlement files from Dojo reconcile automatically against the bank.
- Deposits: fixed, percentage or first-night rules per rate plan, with automatic collection dates and reminders.

**Refunds, chargebacks and disputes**
- Refund types: full, partial, item-level, goodwill credit, and refund to a different method when the original card is unavailable (bank transfer or voucher) with manager approval and audit trail.
- Policy engine computes the refundable amount from the cancellation policy and dates, shows the staff member the calculation, and allows an override with reason code; overrides are reported weekly.
- Refunds post as reversing ledger entries and generate a credit note that references the original invoice (required for VAT).
- Chargeback handling: evidence pack generated from the folio, signed registration card, online check-in acceptance log, and door-lock or Wi-Fi activity where available; status tracked to resolution.
- Guest-initiated: self-service cancellation on the booking engine within policy, with instant refund; OTA cancellations arrive via the channel manager and trigger the same engine.

**In-stay**
- Guest web app: order to room with room charge, request housekeeping items, report issues, chat with front desk, view folio live, extend stay, book tables in the property's own restaurant through the EPOS.
- Post-stay: review request via the reputation module, re-booking offer with a direct-booking promo code.

### 2.8 Training, documentation and user guides

Shipped alongside each phase, not after. Hotels churn on bad onboarding more than on missing features.

- **In-product guidance:** contextual help on every screen, first-run checklists per role (owner, front desk, night auditor, housekeeper, revenue manager), guided tours for new modules, empty-state explanations, and a searchable help centre embedded in the admin console.
- **User guides:** a complete written manual per module (Rooms, Connect, Keep, Yield, Admin, Guest journey), kept in the repository as Markdown so it versions with the software and publishes to a public docs site on every release. Each guide has task-based how-tos ("take a deposit", "handle a no-show", "map a new rate plan to Booking.com", "run the night audit", "do a linen count"), screenshots regenerated automatically, and a troubleshooting section.
- **Video library:** 2 to 5 minute screencasts per task, recorded against the training-mode property; a 45-minute "new hotel" walkthrough and a 20-minute "new housekeeper" video playable inside the staff app.
- **Training mode:** every organisation gets a sandbox property with realistic sample data that resets nightly, so staff practise check-ins, refunds and audits without touching live books.
- **Role-based courses and certification:** short courses with quizzes for front desk, night audit, housekeeping and revenue; completion recorded on the employee profile in Zynthio HR, so a manager can see who is trained before putting them on a rota. Certification badges can be a requirement for a shift type.
- **Onboarding playbook:** a structured 30-day migration and go-live plan per property: data import, parallel run, channel cut-over sequence (close the old channel manager, open Zynthio, verify ARI on each OTA within one hour), first night audit with Zynthio support on a call, week-one check-in.
- **Internal enablement:** support staff runbooks, escalation paths (already in the ticketing flow), release notes written for hoteliers in plain English, and a quarterly webinar.
- **Accessibility and language:** guides in plain English at reading age 12, with Polish, Romanian and Spanish translations of the housekeeping guides given the UK workforce.

Documentation and training are estimated at 10 to 15 percent of build effort (included in the figures in section 5) plus a part-time technical writer or trainer from phase 3.

### 2.9 Compliance and security
- GDPR and UK GDPR: data retention schedules, right-to-erasure tooling, DPA template for customers, Booking.com PII review readiness.
- PCI DSS: never store PANs; use Dojo tokenisation and a vault for OTA virtual cards, aim for SAQ-A or SAQ-A-EP scope; annual penetration test.
- UK guest registration: capture of nationality and passport details for non-UK guests as required by the Immigration (Hotel Records) Order.
- Accessibility (WCAG 2.1 AA) on booking engine; Cyber Essentials Plus for public-sector tenders such as universities.

---

## 3. Architecture

Fit it to the existing Zynthio platform rather than build a parallel stack. Specific technology names below assume the current web platform plus the native app; adjust to the actual stack.

- **Domain services (modular monolith first):** Inventory and Rates, Reservations, Folio and Ledger, Payments, Housekeeping, Channel Sync, Pricing, Admin. Each with its own schema boundary so services can be split later.
- **Ledger as an append-only journal:** every posting is an immutable double-entry record; corrections are reversing entries. This makes night audit, VAT and accountant exports defensible.
- **Event bus:** reservation created, room status changed, folio posted, inventory changed. Channel sync, housekeeping task generation, EPOS and notifications subscribe to events; nothing polls the database.
- **Channel sync service:** outbox pattern for ARI pushes with idempotency keys, exponential retry, dead-letter queue and operator view; inbound reservations via webhooks with signature verification, de-duplication, and a mapping engine. Adapter interface so Channex today and direct Booking.com or Expedia adapters later share the same contract.
- **Pricing service:** rules engine evaluated nightly and on triggers; forecasting jobs run in a worker with model artefacts per property; recommendations stored with explanations.
- **Mobile:** housekeeping module in the existing native app, offline queue with conflict resolution; a lightweight web fallback.
- **Reporting:** nightly snapshots into a reporting schema or warehouse; materialised KPIs for dashboards.
- **Stack:** Python with Django and Django REST Framework for Stay (models, migrations, viewsets, serializers, permissions, Celery workers), Angular feature modules in the existing front end, the existing native app for housekeeping. Integrated with Zynthio organisations, sites, users, rotas and EPOS from the first migration.
- **Infrastructure:** run on the existing Zynthio hosting and SendGrid email, UK-hosted (data residency matters to universities), managed Postgres with point-in-time recovery, object storage for documents and photos, queue (Redis or SQS-class), scheduled jobs, observability (logs, traces, uptime), daily encrypted backups off-site (the existing Drive backup routine shows this habit already exists), 99.9 percent target with a published status page.

---

## 4. Phased roadmap

| Phase | Weeks | Outcome |
|---|---|---|
| 0. Discovery and design | 1 to 4 | Workshops with 3 to 5 design-partner hotels (WithUS hotels first), domain model, UX prototypes, Channex sandbox account, Dojo pre-auth and tokenisation flows confirmed, PCI scope agreed. |
| 1. Core PMS MVP | 5 to 20 | Rooms, rate plans, restrictions, reservations, tape chart, guest profiles, folios, Dojo payments and refunds, night audit, VAT invoices, basic reports, admin console, Rooms and Admin user guides and training mode. Run live at one design-partner hotel from week 16. |
| 2. EPOS room charges and single ledger | 15 to 24 | Charge-to-room on tills, package redemption, events folios, EPOS to PMS reconciliation, accounting exports. |
| 3. Channel manager | 18 to 32 | Own channel manager core, direct Booking.com and Expedia adapters in sandbox, iCal sync, mapping UI, ARI push and reservation pull, overbooking protection, booking engine with Dojo checkout, online check-in and express check-out, Google free booking links, Connect guide and channel cut-over playbook. |
| 4. Housekeeping and linen app | 22 to 34 | Room status board, task generation and assignment, inspections, maintenance, lost and found, linen counts and laundry reconciliation, offline mode, Android parity. |
| 5. Revenue engine v1 | 28 to 40 | Rules builder, promotions, vouchers, comp set rate shopping via data feed, market position dashboard, advise and approve modes. |
| 6. Hardening and go-to-market | 34 to 44 | Pentest, Cyber Essentials Plus, migration tooling, onboarding playbook, pricing page, 10 to 20 paying properties. |
| 7. Certification go-live and dynamic pricing v2 | 45 to 78 | Booking.com and Expedia production certification with the first live property, licensed rate feed adapter, forecasting model with autopilot, metasearch and GDS as demanded. |

Months 1 to 10 deliver a sellable product. Months 11 to 18 complete OTA certification and add the premium revenue tier.

---

## 5. Costs

All figures exclude VAT and are estimates to be validated with quotes. GBP unless stated.

### 5.1 Build cost

Zynthio builds this in-house with the founder and AI-assisted engineering, as the rest of the platform was built. Cash build cost is therefore founder time plus AI tooling, not salaries. Where a specialist is worth a short engagement (ledger design review by a hotel accountant, a payments security review), budget 2k to 5k each.

For reference, hiring the work out would cost 180k to 280k for a lean two-engineer build over 14 to 16 months, or 320k to 450k for a team of three to four over 10 months, at UK contractor rates of 450 to 650 a day.

### 5.2 Third-party API and compliance costs

The OTA APIs are free to software partners; the OTAs earn commission from the hotel. The cost of going direct is certification time, not fees.

| Item | Cost |
|---|---|
| Booking.com Connectivity API | 0; certification, self-assessment and PII review, 3 to 6 months of work |
| Expedia Group Connectivity API | 0; ARI, booking, virtual card and promotion support plus EQC certification, 3 to 5 months |
| Airbnb API | Not available to new partners; reach it via the aggregator |
| Google Hotel Ads free booking links | 0; Hotel Center account and price feed |
| GDS via a switch (Derbysoft or similar) | Monthly fee plus per-booking fee; only when a corporate customer requires it |
| Optional aggregator for long-tail channels (Channex or similar) | Only if ever needed: about 130 USD a month plus 7 USD per connected hotel; not required for launch |
| Rate shopping data feed (Yield module) | 0 at launch (manual and extranet import); a licensed feed later at 15 to 40 per property per month, sold on to the hotel as an add-on |
| Dojo | 0 to Zynthio; hotel pays transaction fees, Zynthio earns partner revenue share |
| Penetration test | 6k to 12k per year, from first paying hotels |
| PCI DSS SAQ and ASV scans | 1k to 3k per year |
| Cyber Essentials Plus | 2k to 4k, needed for university and public-sector tenders |
| Legal: terms, DPA, OTA partner agreements | 5k to 10k |
| Design partner incentives (free year for 3 to 5 hotels) | 10k to 20k forgone revenue |
| Video production tooling | 2k to 5k |

Cash outlay to launch the channel manager is close to zero: the Channex sandbox is free and billing starts only when a hotel connects its first channel.

### 5.3 Running costs (monthly, at 50 and 300 properties)

| Item | 50 properties | 300 properties |
|---|---|---|
| OTA connectivity (Booking.com, Expedia, Google, iCal) | 0 | 0 |
| Optional aggregator for long-tail channels | 0 unless enabled | 0 unless enabled |
| Rate shopping data feed (later, 15 to 40 per property) | 0 at launch, 750 to 2,000 with a feed | 0 at launch, 4,500 to 12,000 with a feed |
| Hosting, database, storage, queues, monitoring | Existing Zynthio infrastructure; incremental 0 to 300 | Incremental 500 to 1,500 for database and storage growth |
| Transactional email | Existing SendGrid account; 0 to 50 incremental | 100 to 300 if volume outgrows the current plan |
| SMS and WhatsApp (guest messaging) | 50 to 150 | 400 to 900 |
| App store fees, misc SaaS | 0 (existing developer accounts) | 100 |
| **Total third-party cost** | **50 to 500 at launch; 800 to 2,500 with a rate feed** | **600 to 2,800 at launch; 5,000 to 15,000 with a rate feed** |

Support and engineering time is founder time at launch; budget a first support hire at around 40 properties and a second engineer at around 100.

Dojo transaction fees are paid by the hotel, not Zynthio; Zynthio earns partner revenue share on volume, which can be material once rooms revenue flows through Dojo.

### 5.4 Pricing model for Zynthio and unit economics

Market reference points: flat-fee channel managers commonly cost independents 50 to 150 USD per month, SiteMinder entry plans list at 85 to 119 USD per month, and bundled PMS plus channel manager suites run 180 to 500 USD per month.

Proposed per-property monthly pricing (per room, with floors):

| Module | Price | 20-room example |
|---|---|---|
| Rooms (PMS, payments, ledger, admin) | 4 per room, minimum 79 | 80 |
| Connect (channel manager, booking engine) | 2 per room, minimum 49 | 49 |
| Keep (housekeeping and linen) | 1.50 per room, minimum 29 | 30 |
| Yield (rules, promos, rate shopping, dynamic pricing) | 3 per room, minimum 69 | 69 |
| Full bundle | 9 per room, minimum 199 | 199 |

A 20-room inn on the bundle pays about 2,400 a year, under half the cost of a typical PMS, channel manager and RMS stack, while Zynthio's variable cost for that property (aggregator and rate feed) is roughly 25 to 45 a month. Gross margin on software is above 75 percent before support.

With no salaried build cost, the modules are cash-positive from the first few properties: a 20-room bundle at 199 a month carries almost no variable third-party cost until a licensed rate feed is added. Hosting and email already exist and OTA connectivity is free, so the only fixed overhead is compliance; two or three properties cover it. The WithUS deal alone would cover two hotels plus the venue; the university's 18 restaurants are already EPOS revenue.

---

## 6. Risks and mitigations

- **Payments and ledger correctness.** Hotel accountants will leave over a penny of variance. Mitigation: immutable journal, reconciliation reports from day one, a chartered-accountant review of the ledger design, parallel run with the incumbent system at the first hotel.
- **Overbookings during channel sync failures.** Mitigation: outbox and retry, stop-sell cascade, synthetic monitoring that books and cancels a test room, per-channel allotment buffers configurable by the hotel.
- **OTA programme dependency.** Airbnb has closed new partner intake; Booking.com and Expedia change criteria annually and certification needs a live property. Mitigation: register now, build against sandboxes from day one, iCal for closed channels, aggregator adapter kept as a fallback.
- **Rate shopping legality.** Scraping breaches OTA terms and would endanger the Booking.com partner application. Mitigation: manual and extranet-import sources now, licensed feed later, no crawling.
- **Team bandwidth.** The founder is running sites and opening a pub. Mitigation: design partners who co-own requirements, a tech lead who owns delivery, strict phase gates.
- **Hardware.** Hotels expect key cards, printers and terminals to just work; the Jarltech relationship covers tills and printers, and lock vendors need separate agreements.
- **Public sector procurement.** Universities will ask for Cyber Essentials Plus, a DPA, accessibility statements and an exit plan. Build these into phase 6.

---

## 7. Immediate next steps (next 30 days)

1. Confirm the two WithUS hotels as design partners and schedule discovery workshops.
2. Register on the Booking.com Connectivity and Expedia Group developer portals, obtain sandbox credentials and run the first ARI push and reservation pull against their test properties.
3. Confirm with Dojo: pre-authorisation, incremental auth, capture, tokenised card on file and pay-by-link availability on the partner API, and the PCI scope they certify.
4. Build the comp-set framework with manual and extranet-import sources; shortlist licensed providers for later.
5. Write the domain model for Inventory, Reservations and Ledger, and get it reviewed by an accountant who has run hotel books.
6. Using the Dojo partner API sandbox and the OTA sandboxes, build the first vertical slice (room type, rate, availability, one booking pulled and paid) within the first fortnight.
7. Confirm the Zynthio backend framework and set up the Stay Django apps and Angular feature modules against the existing organisation, site and user models.

---

## Sources

- Booking.com connectivity documentation: https://developers.booking.com/connectivity/docs and https://developers.booking.com/connectivity/docs/going_live
- Booking.com partner tiers (vendor announcements, 2026 Premier Plus): https://www.hostaway.com/blog/booking-com-premier-plus-2026/ and https://hospitable.com/booking-com-connectivity-partners
- Expedia Group Connectivity Partner Program: https://developers.expediagroup.com/supply/lodging/cpp/overview and https://developers.expediagroup.com/supply/lodging/updates
- Airbnb API access status: https://www.airbnb.com/terms/api and https://airbnb.com/d/preferred-software-partner
- Channex white-label pricing: https://channex.io/pricing.md and https://channex.io/
- Channel manager market pricing: https://www.capterra.com/p/123133/SiteMinder/pricing/ , https://www.stayntouch.com/articles/?p=311 , https://www.roommaster.com/blog/siteminder-pricing
