# Zynthio Stay: System Design and Function Catalogue

**Status:** Draft v1, 26 November 2026
**Companion to:** zynthio-pms-channel-manager-plan.md (product scope, roadmap, costs)

---

## 1. Design principles

1. **One organisation model.** Hotels are properties inside the existing Zynthio organisation and site hierarchy. Staff, roles, rotas, clock-ins and EPOS outlets already hang off a site; rooms, rates and folios hang off the same site.
2. **Ledger is the source of truth for money.** Every financial fact is an immutable journal entry. Screens, reports and exports are views over the journal. Nothing edits money in place.
3. **Events, not polling.** Domain events drive channel sync, housekeeping tasks, notifications, EPOS and reporting. Each consumer is idempotent and can replay.
4. **Configuration over code.** Rate logic, policies, taxes, templates, task rules and pricing rules are data edited in the admin console, versioned and auditable.
5. **Adapters at every edge.** Channels, payments, locks, accounting and messaging sit behind interfaces so a vendor can be swapped without touching the domain.
6. **Offline tolerant at the edge.** Tills and the housekeeping app queue locally and reconcile; the server resolves conflicts deterministically.
7. **Explainable automation.** Every automated price, task assignment or stop-sell records why it happened in plain English.

---

## 2. Component architecture

Modular monolith on the existing Zynthio backend, split into bounded contexts with their own schemas and a shared event bus. Services can be extracted later without changing contracts.

| Component | Responsibility | Talks to |
|---|---|---|
| Identity and Tenancy | Organisations, properties, users, roles, permissions, SSO, API keys | Everything |
| Inventory and Rates | Room types, rooms, rate plans, derived rates, restrictions, availability calendar | Reservations, Channel Sync, Pricing |
| Reservations | Booking lifecycle, assignment, groups and blocks, guest profiles, companies and agents | Inventory, Folio, Housekeeping, Messaging |
| Folio and Ledger | Folios, postings, journal, taxes, invoices, credit notes, night audit, AR | Payments, EPOS, Accounting adapters |
| Payments | Dojo adapter, tokens, pre-auth, capture, refunds, pay-by-link, virtual cards, reconciliation | Folio, Booking Engine, Front Desk |
| Channel Sync | ARI outbox, channel adapters (Channex now, Booking.com and Expedia direct later), mapping, inbound reservations, health | Inventory, Reservations, Pricing |
| Booking Engine | Public availability search, rate display, checkout, promo codes, upsells, metasearch feed | Inventory, Pricing, Payments, Reservations |
| Pricing (Yield) | Rules engine, promotions, comp set data ingestion, forecasting, recommendations | Inventory, Channel Sync, Reporting |
| Housekeeping and Assets | Room status, tasks, inspections, maintenance, lost and found, linen and stock counts | Reservations, Rota, Staff app |
| EPOS Bridge | Charge-to-room, package redemption, event tabs, reconciliation with the existing EPOS | Folio, Reservations, EPOS |
| Guest Journey | Online check-in, express check-out, in-stay web app, messaging templates and schedules | Reservations, Folio, Payments, Messaging |
| Messaging | Email via SendGrid, SMS, WhatsApp Business, push to the staff app, template rendering | All modules |
| Reporting | Nightly snapshots, KPI materialisation, exports, scheduled report delivery | All modules, existing KPI dashboard |
| Admin Console | Configuration UI for all of the above, audit log, feature flags, import and export, billing | All modules |
| Integrations Hub | Accounting, door locks, Wi-Fi, TV, review platforms, webhooks, public API | Folio, Reservations, Guest Journey |

**Runtime pieces:** web app (existing), native staff app (existing, iOS live, Android in progress) with the housekeeping module added, public booking engine and guest web app (new, server-rendered and fast on phones), background workers (ARI pushes, night audit, pricing, snapshots), scheduler, Postgres with row-level tenancy, Redis for queues and caches, object storage for documents and photos, SendGrid for email.

---

## 3. Data model (core entities)

| Entity | Key fields | Notes |
|---|---|---|
| Property | org_id, site_id, name, address, timezone, currency, check-in/out times, tax jurisdiction, settings | One per hotel; links to the existing site |
| RoomType | property_id, code, name, description, max adults, max children, base occupancy, bed configs, amenities, sort order | Sold unit for channels |
| Room | room_type_id, number, floor, building, features, connecting_room_id, status (clean, dirty, inspected, OOO, OOS), notes | Physical unit |
| RatePlan | property_id, code, name, type (public, corporate, package, OTA), parent_rate_plan_id, derivation (percent or amount), currency, meal plan, cancellation_policy_id, deposit_policy_id, channels allowed, min and max occupancy pricing mode | Derived plans recompute from parent |
| DailyRate | rate_plan_id, room_type_id, date, price per occupancy, extra adult, extra child | Materialised per date for fast reads and pushes |
| Restriction | rate_plan_id or null, room_type_id, date, min_stay, max_stay, CTA, CTD, stop_sell, release_days | Nullable rate_plan means room-type level |
| Availability | room_type_id, date, physical, out_of_order, sold, blocked, available, overbooking_allowance | Derived; recomputed on every change and verified nightly |
| Policy | property_id, kind (cancellation, deposit, no-show, guarantee), rules JSON, text per language | Versioned; reservations pin the version they were sold under |
| TaxRule | property_id, code, name, rate, type (percent, fixed per night, per person), applies to charge codes, inclusive or exclusive, valid from and to | Handles VAT and visitor levies |
| ChargeCode | property_id, code, name, department, revenue_category, tax_rule_id, GL account mapping | Shared with EPOS item categories |
| GuestProfile | org-level id, names, contacts, nationality, ID document (encrypted), preferences, consents, tags, blacklist flag, stay stats | De-duplicated across properties with consent |
| Company and Agent | name, billing address, credit limit, payment terms, commission percent, negotiated rate plans, contacts | Feeds city ledger |
| Reservation | property_id, confirmation number, status, source, channel, channel_reservation_id, guest_id, company_id, arrival, departure, adults, children, room_type_id, room_id, rate_plan_id, policy versions, special requests, ETA, created_by, audit | Status machine in section 4 |
| ReservationNight | reservation_id, date, rate, tax, package components | Allows per-night pricing and splits |
| Group and Block | name, cut-off date, contract terms, allotments per room type per date, pickup, master folio | Events and conferences |
| Folio | reservation_id or account_id, window number, payer type, status (open, closed, settled), balance | Multiple windows per reservation |
| Posting | folio_id, charge_code_id, description, quantity, net, tax, gross, business_date, source (room, EPOS, manual, package, adjustment), source_ref, posted_by, reversal_of | Append only |
| JournalEntry | business_date, lines (account, debit, credit), source posting or payment, property_id | Double entry under every posting and payment |
| Payment | folio_id, method, amount, currency, status (authorised, captured, refunded, failed), gateway_ref, token_id, card last four, auth expiry, settlement batch | Dojo references kept for reconciliation |
| PaymentToken | guest_id or reservation_id, gateway token, scheme, expiry, consent record | Never stores PANs |
| Invoice and CreditNote | number sequence per property, payer, lines, VAT summary, PDF, issued_at, related invoice | Sequential and immutable |
| BusinessDay | property_id, date, status (open, auditing, closed), audit log, variances | Night audit control record |
| HousekeepingTask | room_id, type, due, priority, credits, assignee, status, inspection result, photos | Generated by rules |
| MaintenanceTicket | room or area, category, priority, photos, assignee, status, parts, cost | Links to OOO status |
| LinenItem and StockCount | item, location, par, counted, issued, collected, sent, returned, condemned, counted_by | Variance derived |
| ChannelMapping | channel, channel property id, Zynthio room_type to channel room id, rate_plan to channel rate id, markup, status | One per channel per property |
| SyncJob | kind (ARI, reservation), payload hash, attempts, status, last error, idempotency key | Outbox record |
| PricingRule | property_id, name, conditions JSON, actions JSON, priority, active window, approval mode | Evaluated in priority order |
| Recommendation | room_type_id, date, current rate, recommended rate, confidence, reasons, status (pending, approved, applied, rejected) | Explainable |
| CompSetRate | competitor, date, los, occupancy, rate, source, fetched_at | From licensed feed |
| Template | kind (email, SMS, document), language, subject, body, merge fields, version | Visual editor output |
| AuditLog | actor, action, entity, before, after, at, ip | Every config and money change |

---

## 4. Key flows

**4.1 Reservation state machine**
Enquiry → Tentative → Confirmed → Checked in → Checked out; side exits to Cancelled and No-show. Transitions emit events: reservation.created, modified, cancelled, checked_in, checked_out, no_show. Each transition checks availability, policy and folio state and writes an audit row.

**4.2 OTA booking arrives**
Channel webhook → signature verified → payload stored raw → de-duplicated on channel_reservation_id → mapped to room type, rate plan and guest → availability decremented → reservation created with pinned policies → virtual card details tokenised → confirmation acknowledged to channel within the SLA → events fan out: ARI push for the affected dates, housekeeping forecast updated, guest pre-arrival schedule created, reporting snapshot marked dirty. Unmappable bookings land in an exceptions queue with a one-click fix that also repairs the mapping.

**4.3 Availability or rate changes**
Any write to DailyRate, Restriction, Room status or Reservation → Availability recomputed for affected room types and dates → ARI delta written to the outbox with an idempotency key → worker pushes to each connected channel with retry and backoff → channel acknowledgement recorded → health dashboard updated. A nightly full reconciliation pushes the complete 365-day ARI to catch drift.

**4.4 Charge to room from EPOS**
Till selects "Room charge" → searches in-house guests by room or name → PMS validates: guest in house, room charge allowed, credit limit → till captures PIN or signature → EPOS closes the check → posting created on the guest's folio with outlet, server, items and tax → journal entry written → till prints a receipt showing the room. Reversal from either side creates a reversing posting with approval.

**4.5 Night audit**
Scheduled at the property's audit time → pre-checks (unposted EPOS checks, arrivals not checked in, departures not checked out, open payment auths, unmapped channel bookings) → auto-resolve or hold for the auditor → room and tax posted for every in-house night → no-shows processed per policy → deposits due charged → business date rolled → daily reports generated and emailed → BusinessDay closed with a hash of the day's journal.

**4.6 Online check-in**
Scheduler sends the link at the configured offset → guest completes details, ID, consents, upsells, card pre-auth, signature → reservation flagged ready and room pre-assigned if clean or inspected → key code or digital key issued at the configured time → front desk confirms arrival with one tap, or an unattended arrival is logged by the lock or code use.

**4.7 Express check-out and refund**
Folio summary sent evening before → guest confirms → balance captured on the token → invoice issued and emailed → room set dirty, departure clean task created. Refunds run through the policy engine: compute refundable amount from pinned policy and dates → staff sees calculation → optional override with reason → reversing posting and credit note → gateway refund → event to messaging.

**4.8 Housekeeping task generation**
Nightly and on every reservation change: rules decide which rooms need departure cleans, stay-over service, linen change, deep clean → credits assigned per task type → assignment engine distributes by section, workload and who is on shift in the rota → tasks appear in the app, ordered by expected arrival time → supervisor inspects → room status flips → front desk sees it live.

**4.9 Pricing cycle**
Nightly and on triggers (pickup threshold, comp set move, event added): gather own pace and pickup, comp set rates, events, restrictions → evaluate rules in priority → produce recommendations with reasons → in advise mode notify, in approve mode wait for approval, in autopilot apply within guardrails → applied rates flow through 4.3.

---

## 5. Integration contracts

| Integration | Direction | Protocol | Notes |
|---|---|---|---|
| Channex (aggregator) | Both | REST plus webhooks | ARI push, reservations pull, mapping API; white-label |
| Booking.com Connectivity (phase 7) | Both | REST and XML endpoints | Rates and availability, reservations, PII review |
| Expedia Group (phase 7) | Both | REST | Product, AR, booking retrieval and notification, virtual cards |
| Google Hotel Ads | Out | Price feed and landing page | Free booking links first |
| Dojo | Both | Partner API and terminal integration | Pre-auth, capture, refund, tokens, pay-by-link, settlement files |
| Zynthio EPOS | Both | Internal API and events | Room charge, package redemption, reconciliation |
| Accounting (Xero, Sage, QuickBooks) | Out | Vendor APIs or CSV | Daily journal summary by GL account |
| Door locks (Salto, Assa Abloy, dormakaba, Onity) | Out | Vendor SDK or cloud API | Key encoding and mobile keys |
| Rate shopping feed | In | Vendor API | Daily and on-demand comp set rates |
| SendGrid | Out | API | Templates rendered by Zynthio, SendGrid delivers; webhooks for bounces |
| SMS and WhatsApp Business | Out | Vendor API | Opt-in managed per guest |
| Public API and webhooks | Both | REST, API keys, HMAC-signed webhooks | For customers and partners |

---

## 6. Multi-tenancy, security and compliance

- Row-level security keyed on org_id and property_id; every query scoped by the authenticated principal.
- Roles: owner, group admin, property manager, front desk, night auditor, housekeeping supervisor, housekeeper, maintenance, revenue manager, finance, read-only, API client. Permissions are granular and assignable per property.
- Secrets in a vault; card data never touches Zynthio servers except as gateway tokens; OTA virtual card details encrypted at rest with field-level keys and purged after capture.
- Audit log on all configuration and financial actions, exportable per property.
- GDPR tooling: retention schedules, erasure requests that anonymise profiles while preserving the ledger, consent records, data export.
- Backups: point-in-time recovery plus daily off-site encrypted dumps, with a quarterly restore test.
- Observability: structured logs, tracing on every sync job, uptime checks, alerting on ARI failure rate, booking acknowledgement latency and audit failures.

---

## 7. Function catalogue

### 7.1 Admin and configuration

| Function | What it does |
|---|---|
| Organisation and property setup | Creates a hotel under the organisation, links it to the existing site, sets time zone, currency, check-in and check-out times, address and contact details |
| Users, roles and permissions | Invites staff, assigns roles per property, controls what each role can see and do, enforces two-factor and SSO |
| Room type and room builder | Defines sellable room types, their occupancy and amenities, and the physical rooms with floors, features and connecting rooms |
| Rate plan manager | Creates public, corporate, package and OTA rate plans, sets derivation from a parent, meal plans, occupancy pricing and which channels can sell them |
| Rate calendar | Grid view of prices and restrictions by room type, rate plan and date, with bulk edit by date range and day of week |
| Policy editor | Writes cancellation, deposit, no-show and guarantee policies as rules plus guest-facing text in each language |
| Tax and levy rules | Configures VAT and per-night or percentage levies, which charges they apply to, inclusive or exclusive pricing, and effective dates |
| Charge codes and chart of accounts | Defines every chargeable item, its department and revenue category, and the ledger account it posts to; shared with EPOS categories |
| Document and message templates | Visual editor for confirmations, invoices, registration cards, emails and SMS with merge fields, per language and brand |
| Feature flags and module entitlements | Turns modules on per property and ties them to billing |
| Training mode | Sandbox property with realistic data that resets nightly for staff practice |
| Data import and export | CSV import of rooms, rates, guests and future reservations from the old PMS; export of everything |
| Audit log viewer | Searches who changed what and when, across configuration and money |
| Integrations hub | Connects Dojo, accounting, locks, review sites and webhooks; manages API keys |
| Billing | Meters module usage per property and generates Zynthio's own invoices |
| Status and release notes | Shows system health and what changed in each release, in plain English |

### 7.2 Inventory, rates and availability

| Function | What it does |
|---|---|
| Availability engine | Calculates sellable rooms per type per date from physical rooms, out-of-order, sold and blocked, with an optional overbooking allowance |
| Restriction manager | Sets minimum and maximum stay, closed to arrival or departure, stop-sell and release periods at room-type or rate-plan level |
| Derived rates | Recomputes child rate plans automatically when a parent price changes |
| Out of order and out of service | Takes rooms off sale with a reason and return date, adjusting availability and housekeeping |
| Allotments and blocks | Holds inventory for groups, events and channels with cut-off dates and pickup tracking |
| Nightly availability verification | Recounts availability from first principles and flags any drift |

### 7.3 Reservations and front desk

| Function | What it does |
|---|---|
| Reservation create and modify | Books individual, multi-room and group stays with pinned policies, special requests and ETA |
| Tape chart | Drag-and-drop room assignment view by date, with status colours and conflicts highlighted |
| Auto-assignment | Allocates rooms by preferences, housekeeping status and upcoming maintenance |
| Arrivals and departures boards | Live lists for the day with pre-check-in status, balances and notes |
| Check-in and check-out | Captures ID, pre-authorises a card, issues keys, settles folios, prints or emails invoices |
| Walk-in and quick book | Fastest path to sell an available room at the desk |
| Room move, extension, early departure | Changes a stay in place and re-prices nights, folios and housekeeping |
| Waitlist | Holds requests for sold-out dates and notifies when rooms open |
| Guest profiles | Stores contact, preferences, ID, consents, stay history and notes, de-duplicated across the organisation |
| Company and agent accounts | Manages corporate and agent profiles, negotiated rates, credit limits and commissions |
| Groups and events | Builds room blocks, master accounts, rooming lists and function sheets for conferences |
| No-show and cancellation processing | Applies the pinned policy to charge or release automatically with override |
| Key encoding | Issues and revokes key cards or mobile keys through the lock adapter |
| Front desk cashiering | Cash drawer sessions, shift totals and paid-outs tied to the user and property |

### 7.4 Folio, payments and ledger

| Function | What it does |
|---|---|
| Folio windows | Splits charges across guest, company and third-party payers within one reservation |
| Posting | Adds room, tax, extras, package and manual charges with department and tax code |
| Transfers and splits | Moves postings between folios and windows with an audit trail |
| Adjustments and allowances | Corrects charges via reversing entries with reason codes and approval |
| Pre-authorisation and incremental auth | Holds funds at booking or check-in and increases the hold as the folio grows |
| Capture and settlement | Charges the held amount at check-out and batches settlements |
| Card on file | Tokenises cards with consent for no-shows, incidentals and post-departure charges |
| Pay-by-link | Sends secure payment requests for deposits, group balances and remote payments |
| Terminal payments | Takes card-present payments at the desk and on handhelds, posted straight to the folio |
| Wallets | Accepts Apple Pay and Google Pay on the booking engine and pay-by-link |
| Virtual card handling | Stores OTA virtual cards securely and charges them on the permitted date |
| Deposits | Calculates, collects and tracks deposits per policy with reminders |
| Refund engine | Computes the refundable amount from policy and dates, supports partial and alternative-method refunds, issues credit notes |
| Chargeback evidence | Assembles folio, signed registration, check-in logs and activity into a dispute pack |
| Invoices and credit notes | Produces sequential UK VAT invoices and credit notes per payer, as PDF and email |
| Guest, city and deposit ledgers | Tracks in-house balances, receivables from companies and agents, and prepaid liabilities with ageing and statements |
| Night audit | Posts room and tax, processes no-shows, rolls the business date and produces daily reports with variance checks |
| Journal and trial balance | Double-entry record of every posting and payment, with trial balance and account drill-down |
| Accounting export | Sends daily journal summaries to Xero, Sage or QuickBooks, or as CSV |
| Payment reconciliation | Matches Dojo settlement files to ledger payments and flags differences |

### 7.5 EPOS bridge

| Function | What it does |
|---|---|
| Charge to room | Lets any till or handheld post a check to an in-house guest's folio after validation and PIN or signature |
| Package redemption | Marks inclusive breakfast and dinner covers as consumed without charging twice |
| Room charge reversal | Voids or refunds a room charge from till or PMS with approval |
| Event tabs | Runs bar and catering tabs against event master accounts |
| Table bookings for guests | Books the property's restaurant from the guest web app and links covers to the stay |
| EPOS to PMS reconciliation | Compares till totals to folio postings daily and queues exceptions |
| Shared master data | Keeps items, categories, taxes and departments aligned between EPOS and PMS |

### 7.6 Channel manager and booking engine

| Function | What it does |
|---|---|
| Channel connection wizard | Connects a property to each OTA through the aggregator with guided steps |
| Mapping manager | Maps room types and rate plans to channel products, detects unmapped items, sets per-channel markups |
| ARI push | Sends availability, rates and restrictions on every change, with retries and a nightly full refresh |
| Reservation delivery | Receives new, modified and cancelled OTA bookings and creates or updates reservations automatically |
| Overbooking protection | Applies last-room logic, per-channel allotments and stop-sell cascades when inventory reaches zero |
| Exceptions queue | Surfaces bookings that could not be mapped or payments that failed, with one-click fixes |
| Channel health | Shows sync status, latency, error rate and booking acknowledgement time per channel |
| Parity monitor | Compares rates across channels and direct and flags violations |
| Booking engine | Mobile-first direct booking site per property with availability search, rate display, upsells, promo codes and Dojo checkout |
| Rate comparison widget | Shows direct versus OTA prices on the booking engine |
| Metasearch feed | Publishes prices to Google Hotel Ads free booking links, later Trivago and TripAdvisor |
| Direct connections (phase 7) | Native Booking.com and Expedia adapters replacing the aggregator for those channels |
| GDS (on demand) | Connects to Amadeus, Sabre and Travelport via a switch for corporate business |

### 7.7 Revenue engine (Yield)

| Function | What it does |
|---|---|
| Rules builder | Lets a manager write pricing and restriction rules from conditions and actions without code |
| Occupancy ladders | Adjusts prices by occupancy band per room type |
| Event calendar | Imports bank holidays and local events and lets staff add their own to drive demand rules |
| Guardrails | Enforces floors, ceilings and maximum daily change on any automated price |
| Promotions | Creates promo codes, early bird, last minute, length-of-stay, mobile and member deals with caps |
| Upsells and add-ons | Sells extras at booking, check-in and in stay with inventory limits |
| Gift vouchers | Sells and redeems vouchers against folios |
| Channel promotions | Mirrors eligible deals to OTA promotion programmes |
| Rate shopping | Pulls competitor rates daily from a licensed feed for a configured comp set |
| Market position dashboard | Ranks the property against competitors by date, with parity and sold-out signals |
| Demand signals | Combines pickup, pace, comp set movement, events and search demand into a daily demand index |
| Recommendations | Produces a suggested rate per room type per date with confidence and reasons |
| Advise, approve, autopilot | Chooses whether recommendations notify, wait for approval or apply automatically |
| Displacement analysis | Shows what a group block costs in forgone transient revenue before quoting |
| Forecasting | Projects occupancy and revenue by day from history and pace |

### 7.8 Housekeeping, maintenance and linen

| Function | What it does |
|---|---|
| Room status board | Live view of clean, dirty, inspected, occupied, vacant, DND and out-of-order rooms |
| Task generation | Creates departure cleans, stay-over services, linen changes and deep cleans from reservation and rule data |
| Task assignment | Distributes tasks by section, workload credits and who is on shift in the rota |
| Housekeeper mobile view | Shows each person their rooms in arrival order, with notes and guest requests, offline-capable |
| Inspection checklists | Supervisor checks with photos and fail reasons before a room is released |
| Guest requests | Routes extras and service requests from front desk or guest app to the right housekeeper |
| Lost and found | Logs items with photos and storage location, matches to guests |
| Maintenance tickets | Raises, prioritises and tracks repairs with photos, parts and cost, and links to out-of-order |
| Preventive maintenance | Schedules recurring checks per room or asset |
| Minibar and consumables count | Records in-room usage and posts it to the folio |
| Linen pars and counts | Sets par levels and records clean issued, dirty collected, sent, returned and condemned |
| Laundry reconciliation | Matches external laundry delivery notes and invoices to counts |
| Stock counts | Periodic counts of amenities and chemicals with variance and reorder alerts |
| Productivity reports | Minutes per room, credits per shift, inspection pass rate, linen cost per occupied room |

### 7.9 Guest journey and messaging

| Function | What it does |
|---|---|
| Message scheduler | Sends confirmation, pre-arrival, in-stay and post-stay messages at configured offsets by email, SMS or WhatsApp |
| Online check-in | Collects details, ID, consents, upsells, card pre-authorisation and signature before arrival |
| Kiosk mode | Runs the same check-in on a reception tablet |
| Digital key and codes | Issues keys or door codes at the right time through the lock adapter |
| Guest web app | Lets guests order to room, request items, report issues, view their folio, extend and book tables |
| Express check-out | Lets guests review charges, choose invoice recipient, pay and leave without the desk |
| Self-service cancellation | Cancels within policy on the booking engine with automatic refund |
| Review request | Invites guests to review after stay through the existing reputation module |
| Re-booking offers | Sends direct-booking promo codes to past guests |

### 7.10 Reporting and analytics

| Function | What it does |
|---|---|
| Daily manager report | Occupancy, ADR, RevPAR, revenue by department, payments and balances for the business day |
| Pace and pickup | Bookings on the books versus same time last year and recent pickup by day |
| Channel mix and acquisition cost | Revenue and commission by channel, direct share |
| Segment and source | Performance by market segment, company, agent and geography |
| Cancellation and no-show analysis | Rates by channel, lead time and rate plan |
| Labour versus revenue | Hourly rooms and F&B revenue against rota cost from the existing rota module |
| Forecast versus budget | Compares projections and actuals to targets by month |
| Housekeeping and linen KPIs | Productivity, inspection and linen cost metrics |
| Scheduled reports | Emails chosen reports to chosen people on a schedule |
| Exports and API | CSV exports and API access to every report dataset |

### 7.11 Training and support

| Function | What it does |
|---|---|
| In-app guidance | Contextual help, first-run checklists and guided tours per role |
| Help centre | Searchable user guides per module, versioned with the software |
| Video library | Short task videos and full walkthroughs, playable in the staff app |
| Role courses and certification | Quizzes per role with completion recorded on the employee profile in Zynthio HR |
| Onboarding playbook | Structured 30-day migration and go-live checklist per property |
| Support tickets | Existing escalation flow from staff to organisation admin to Zynthio, with consented impersonation for support |
