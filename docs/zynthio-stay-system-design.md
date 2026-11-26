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
| Channel Sync | Own channel manager: ARI outbox, direct Booking.com and Expedia adapters, Google Hotel Ads feed, iCal sync, optional aggregator adapter, mapping, inbound reservations, health | Inventory, Reservations, Pricing |
| Booking Engine | Public availability search, rate display, checkout, promo codes, upsells, metasearch feed | Inventory, Pricing, Payments, Reservations |
| Pricing (Yield) | Rules engine, promotions, comp set data ingestion, forecasting, recommendations | Inventory, Channel Sync, Reporting |
| Housekeeping and Assets | Room status, tasks, inspections, maintenance, lost and found, linen and stock counts | Reservations, Rota, Staff app |
| EPOS Bridge | Charge-to-room, package redemption, event tabs, reconciliation with the existing EPOS | Folio, Reservations, EPOS |
| Guest Journey | Online check-in, express check-out, in-stay web app, messaging templates and schedules | Reservations, Folio, Payments, Messaging |
| Messaging | Email via SendGrid, SMS, WhatsApp Business, push to the staff app, template rendering | All modules |
| Reporting | Nightly snapshots, KPI materialisation, exports, scheduled report delivery | All modules, existing KPI dashboard |
| Admin Console | Configuration UI for all of the above, audit log, feature flags, import and export, billing | All modules |
| Integrations Hub | Accounting, door locks, Wi-Fi, TV, review platforms, webhooks, public API | Folio, Reservations, Guest Journey |

**Stack:** Django and Django REST Framework apps for each bounded context (models with migrations, viewsets as controllers, serializers, DRF permissions, Celery workers and beat for jobs), Angular feature modules in the existing Zynthio front end, the existing native app for housekeeping. Shared Postgres with the existing Zynthio schema; Stay tables reference Zynthio organisation, site and user rows by foreign key from the first migration.

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
| Booking.com Connectivity | Both | REST and XML endpoints | Rates and availability, reservations, content; sandbox from phase 3, production after certification |
| Expedia Group Lodging Connectivity | Both | REST | Product, availability and rates, booking retrieval and notification, virtual cards; sandbox from phase 3 |
| iCal channels (Airbnb, Vrbo) | Both | iCal import and export | Availability only; bookings imported from notifications |
| Aggregator (optional, Channex or similar) | Both | REST plus webhooks | Long-tail channels only if a customer needs them |
| Google Hotel Ads | Out | Price feed and landing page | Free booking links first |
| Dojo | Both | Partner API and terminal integration | Pre-auth, capture, refund, tokens, pay-by-link, settlement files |
| Zynthio EPOS | Both | Internal API and events | Room charge, package redemption, reconciliation |
| Accounting (Xero, Sage, QuickBooks) | Out | Vendor APIs or CSV | Daily journal summary by GL account |
| Door locks (Salto, Assa Abloy, dormakaba, Onity) | Out | Vendor SDK or cloud API | Key encoding and mobile keys |
| Rate shopping sources | In | Provider interface: manual entry, extranet import, licensed vendor API later | Daily and on-demand comp set rates |
| SendGrid | Out | API | Templates rendered by Zynthio, SendGrid delivers; webhooks for bounces |
| SMS and WhatsApp Business | Out | Vendor API | Opt-in managed per guest |
| Public API and webhooks | Both | REST, API keys, HMAC-signed webhooks | For customers and partners |

---

## 6. Multi-tenancy, security and compliance

- Row-level security keyed on org_id and property_id; every query scoped by the authenticated principal.
- Roles: Zynthio super admin (platform level, above all organisations: manages tenants, entitlements, feature flags, support impersonation with consent, and enables optional modules such as owner accounting), owner, group admin, property manager, front desk, night auditor, housekeeping supervisor, housekeeper, maintenance, revenue manager, finance, read-only, API client. Permissions are granular and assignable per property.
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
| Channel connection wizard | Connects a property to each OTA directly (API credentials or iCal URLs) with guided steps and a test push |
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
| Channel adapters | Native Booking.com and Expedia adapters, Google Hotel Ads feed, iCal sync; optional aggregator adapter for long-tail channels |
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
| Rate shopping | Collects competitor rates for a configured comp set from manual entry, extranet imports and, later, a licensed feed through one provider interface |
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

---

## 8. Configuration specification

This section defines every setting the admin console exposes, what type it is, its default, and the rule it drives. It is the contract between the admin UI, the domain model and the channel adapters.

### 8.1 Property settings

| Setting | Type | Default | Rule or behaviour |
|---|---|---|---|
| Legal name, trading name | text | from site | Appears on invoices and registration cards |
| Address, phone, email, website | text | from site | Pushed to channels and shown on confirmations |
| Time zone | IANA zone | Europe/London | All business dates, audit time and message offsets use it |
| Currency | ISO 4217 | GBP | Single property currency; channel adapters convert where the channel demands |
| Check-in time, check-out time | time | 15:00, 11:00 | Drives early and late fee thresholds, housekeeping deadlines, key activation |
| Night audit time | time | 04:00 | When the business date rolls; must be after check-out and before check-in |
| Business date mode | enum | rolls at audit | Alternative: rolls at midnight |
| Star rating, property type | enum | none | Required by Booking.com and Expedia content |
| Default child age brackets | list of ranges | infant 0 to 2, child 3 to 12 | Property-wide definition used by room types and rate plans unless overridden |
| Count infants toward occupancy | boolean | false | Infants in cots do not consume a bed space when false |
| Max cots per room default | integer | 1 | Overridable per room type and room |
| Overbooking allowance | integer per room type | 0 | Extra rooms availability may show above physical count; applied in availability engine |
| Tape chart colours | map status to colour | preset | Front desk only |
| Invoice numbering | prefix plus sequence | ZS-{YYYY}-{000001} | Sequential per property, never reused, credit notes have their own sequence |
| Registration card requirements | checklist | name, nationality, passport for non-UK | Which fields online check-in and the desk must capture |
| Data retention | days per data class | 7 years finance, 3 years guest | Erasure jobs respect it |
| Languages | list | en-GB | Templates and booking engine languages |
| Brand | logo, colours, fonts, email sender | Zynthio defaults | Booking engine, guest app, documents and emails |

### 8.2 Room type settings

A room type is the unit sold on every channel. Everything a guest sees before arrival comes from here.

| Setting | Type | Default | Rule or behaviour |
|---|---|---|---|
| Code | short text, unique per property | required | Used in channel mapping and reports |
| Name | text per language | required | Shown on booking engine and pushed to channels |
| Short and long description | rich text per language | empty | Pushed to channels that accept content |
| Photos | ordered list with captions | empty | First photo is hero; minimum sizes enforced for OTA content rules |
| Category | enum (room, suite, apartment, dormitory bed, meeting room, pitch, parking) | room | Dormitory beds sell per bed; meeting rooms and pitches reuse the same inventory engine |
| Size | square metres | empty | Content only |
| Floor range, view, accessibility, smoking | attributes | empty | Filterable in auto-assign and booking engine |
| Bed configuration | list of (bed type, count), with alternatives | required | Alternatives let one room type offer twin or double; selected at booking and shown to housekeeping |
| Standard occupancy | integer | 2 | The occupancy the base rate is priced for |
| Max adults | integer | 2 | Hard cap for adults |
| Max children | integer | 0 | Hard cap for children, by property child brackets |
| Max total persons | integer | max adults | Overall cap; adults plus children cannot exceed it, infants excluded unless the property counts them |
| Min occupancy | integer | 1 | Prevents a family suite being sold to one person if the hotel wants that |
| Extra bed allowed, extra bed count | boolean, integer | false, 0 | Extra beds extend max total up to the physical limit and trigger a housekeeping task |
| Cot allowed, max cots | boolean, integer | true, 1 | Cots do not consume a bed space; create a housekeeping task |
| Child age brackets override | list of ranges | inherit | Lets a family room define its own brackets |
| Occupancy pricing mode | enum (per room, per occupancy, per person) | per occupancy | Per room: one price regardless of guests. Per occupancy: price per number of adults with child supplements. Per person: price times persons, used for hostels and dormitories |
| Amenities | multi-select from the master list | empty | Mapped to channel amenity codes |
| Virtual room type | boolean plus component list | false | Sells a combination of physical room types (two connecting rooms as a family suite); availability is the minimum of components and selling one consumes all components |
| Shared inventory group | optional group id | none | Room types that draw from the same physical pool (same room sold as double or twin); selling one reduces the other |
| Default housekeeping credits | minutes per task type | 30 departure, 15 stay-over | Used by the assignment engine |
| Linen par per room | list of (linen item, quantity) | empty | Drives counts and laundry forecasting |
| Channel defaults | boolean per channel | all on | Whether new rate plans on this room type are offered to each channel |
| Sort order, active, sellable online | integer, boolean, boolean | end, true, true | Inactive room types keep history but cannot be booked |

### 8.3 Room settings

| Setting | Type | Default | Rule or behaviour |
|---|---|---|---|
| Number or name | text, unique per property | required | What housekeeping and guests see |
| Room type | reference | required | A room belongs to exactly one room type; change requires no future bookings or confirmation |
| Building, floor, wing, section | text | empty | Housekeeping sections and auto-assign proximity |
| Occupancy override | max adults, max children, max total | inherit | A room in a type that normally sleeps 2 can be marked to sleep 3 |
| Bed configuration override | list | inherit | For the one room in the type with a different layout |
| Features | multi-select | empty | Balcony, sea view, accessible, ground floor, quiet, connecting |
| Connecting room | reference | none | Used for virtual room types and family assignments |
| Status | enum (clean, dirty, inspected, pickup, occupied, vacant) | clean | Set by housekeeping and reservation events |
| Out of order, out of service | range with reason | none | OOO removes from availability; OOS keeps availability but blocks assignment |
| Door lock reference | text | empty | Lock adapter target |
| Smoking, pet friendly | boolean | from type | Assignment filters |
| Notes | text | empty | Shown to front desk and housekeeping |

### 8.4 Occupancy and pricing matrix

How a price is formed for a given room type, rate plan, date and party.

| Element | Definition |
|---|---|
| Party | Adults, children by bracket, infants, extra beds requested, cots requested |
| Validation | Adults <= max adults; children <= max children; adults plus children <= max total (plus extra beds if allowed); adults >= min occupancy; infants <= max cots |
| Base price | Price for standard occupancy on the rate plan for that date |
| Per-occupancy prices | Optional explicit price for 1, 2, 3 ... adults; if absent, base plus or minus adult supplement per difference from standard occupancy |
| Single-use discount | Negative supplement when adults is below standard occupancy |
| Child supplements | Per bracket, per night, per child; may be zero or free up to N children |
| Extra bed and cot charges | Per night supplements, posted as separate charge codes |
| Meal plan | Included in rate or priced per person per night by bracket |
| Length-of-stay pricing | Optional price per night by stay length band on the rate plan |
| Taxes | Applied per charge code after supplements |
| Rounding | Property rounding rule, default nearest 0.01, optional nearest 1 or 5 |
| Channel representation | Channels that accept only per-occupancy prices receive a table of prices for 1 to max adults; children supplements are flattened where the channel cannot express them |

### 8.5 Rate plan settings

| Setting | Type | Default | Rule or behaviour |
|---|---|---|---|
| Code, name per language | text | required | Channel mapping key and guest display |
| Type | enum (public, private, corporate, agent, package, OTA-only, group) | public | Private plans need a code or login; corporate plans attach to companies |
| Room types | multi-select | all | Which room types the plan sells |
| Parent plan and derivation | reference, percent or amount, direction | none | Derived plans recompute when the parent changes; derivation can be per room type and per day of week |
| Currency | inherit | property | Multi-currency display only on the booking engine |
| Pricing mode override | enum | inherit from room type | A package may price per person even if the room type prices per room |
| Price table | per room type per date | required for non-derived | Edited in the rate calendar; bulk by range and weekday |
| Occupancy supplements | adult, single-use, child per bracket, extra bed, cot | inherit | Overrides property-level supplements |
| Meal plan | enum (room only, breakfast, half board, full board, all inclusive) | room only | Drives package components and EPOS redemption |
| Package components | list of (charge code, quantity, per night or per stay, per person or per room, value) | empty | Revenue is split to components on posting for accurate department reporting |
| Cancellation policy | reference | property default | Pinned to the reservation at booking |
| Deposit policy | reference | property default | Fixed, percent, first night, full prepay; due at booking or N days before |
| Guarantee policy | reference | card required | None, card required, pre-authorisation amount |
| No-show policy | reference | first night | Charged at night audit |
| Booking window | min and max days before arrival | none | Advance purchase and last minute |
| Length of stay | min and max nights | none | Can be per date via restrictions |
| Stay dates and sell dates | ranges | open | When the plan is valid and when it can be sold |
| Days of week sellable | set | all | Weekend-only packages |
| Channels | multi-select with per-channel markup | all public | OTA-only plans hidden from direct |
| Loyalty or member only | boolean | false | Booking engine login or code |
| Promo code required | boolean, codes | false | Links to the promotions module |
| Payment timing | enum (pay at hotel, pay now, deposit) | pay at hotel | Booking engine checkout behaviour |
| Commissionable | boolean, percent | true, 0 | For agent and corporate plans |
| Yield controls | enum (open, closed, managed by rules) | open | Whether the revenue engine may change this plan |
| Display order, active | integer, boolean | end, true | |

### 8.6 Restrictions

| Setting | Scope | Behaviour |
|---|---|---|
| Stop sell | room type or rate plan, per date | Removes from sale on all channels; cascades automatically at zero availability |
| Closed to arrival | per date | No stays may start on the date |
| Closed to departure | per date | No stays may end on the date |
| Min stay through, min stay arrival | per date | Minimum nights for stays including or starting on the date |
| Max stay | per date | Maximum nights |
| Release period | days before arrival | Plan closes within N days of arrival |
| Max advance booking | days | Plan closes beyond N days |
| Allotment | per channel per room type per date | Caps rooms a channel may sell; defaults to shared pool |

### 8.7 Policies

| Policy | Settings | Behaviour |
|---|---|---|
| Cancellation | Free until N days or hours before check-in time; then charge fixed, percent, first night or full; non-refundable flag; text per language | Refund engine computes refundable amount from the version pinned at booking |
| Deposit | Amount type (fixed, percent, nights), due (at booking, N days before, at check-in), collection method (card on file, pay-by-link), auto-charge | Scheduler collects and reminds |
| Guarantee | None, card details required, pre-authorisation amount or first night | Booking engine and OTA bookings apply it |
| No-show | Charge amount type, auto-process at audit, release room at audit | Night audit executes |
| Early check-in and late check-out | Time thresholds, fee or free, availability dependent | Guest app upsell and desk |
| Children and extra beds | Free up to N children per bracket, extra bed fee, cot fee | Pricing matrix |
| Pets | Allowed, fee per night or stay, max count | Booking engine filter and posting |
| House rules | Free text per language, must be accepted | Online check-in acceptance recorded |

### 8.8 Taxes, levies and charge codes

| Setting | Type | Behaviour |
|---|---|---|
| Tax rule | name, rate or fixed amount, basis (percent of net, per night, per person per night, per stay), inclusive or exclusive, applies to charge codes, exemptions (children, long stay over 28 nights, business), effective dates | Calculated per posting; UK VAT on accommodation at 20 percent with the 28-day reduced-value rule supported; visitor levies as per-night or percent with exemptions |
| Charge code | code, name, department (rooms, F&B, events, spa, retail, other), revenue category, tax rule, GL account, allows negative, allows manual posting, EPOS category link | Every posting references one; EPOS items map to a charge code so one chart of accounts covers rooms and food |
| Service charge and tronc | percent, optional, applies to departments, tronc flag | Posted as a separate code, excluded from VAT where tronc rules apply |
| Commission codes | agent and channel commission percent, calculated on net or gross | Feeds cost of acquisition and agent statements |

### 8.9 Guest profile fields and consents

| Field group | Fields | Rule |
|---|---|---|
| Identity | title, first and last names, date of birth, nationality, language | Required set configurable per property |
| Contact | email, phone, address | Email or phone required for messaging |
| Travel document | type, number, issuing country, expiry; encrypted | Required for non-UK guests under UK hotel records rules; purged per retention |
| Preferences | room, pillow, floor, dietary, allergies, newspaper | Shown at assignment and to housekeeping |
| Consents | marketing email, SMS, WhatsApp, profile retention, card on file | Timestamped with source; messaging respects them |
| Flags | VIP level, blacklist with reason, do not disturb, repeat guest | Alerts at booking and check-in |
| Company links | employer, booker, payer | City ledger and corporate rates |

### 8.10 Housekeeping configuration

| Setting | Type | Behaviour |
|---|---|---|
| Task types | name, default credits, checklist, requires inspection, triggers (departure, stay-over day N, linen change every N nights, deep clean every N days, on request) | Generation rules |
| Sections | named groups of rooms | Assignment and reporting |
| Shift templates | start, end, break, linked to rota roles | Only staff on shift are assignable |
| Status rules | which status a room goes to on check-out, on task completion, on inspection pass | Front desk visibility |
| Linen items | name, category, par per room type, par per pantry, cost, laundry supplier, lifecycle washes | Counts and condemnation |
| Stock items | amenities and chemicals with pars and reorder points | Counts and alerts |
| Inspection scoring | pass threshold, photo required on fail | Quality reporting |
| Maintenance categories | plumbing, electrical, furniture, IT, exterior; SLA hours per priority | Ticket routing and KPIs |

### 8.11 Channel configuration

| Setting | Type | Behaviour |
|---|---|---|
| Channel account | channel, property id on channel, credentials (direct API keys or iCal URLs, stored encrypted), status | One per channel per property |
| Room type mapping | Zynthio room type to channel room id, with occupancy check | Warns if channel max occupancy differs from Zynthio max |
| Rate plan mapping | Zynthio rate plan to channel rate id, markup percent or amount, rounding | Markups apply at push |
| Occupancy mapping | how per-occupancy and child prices are flattened for the channel | Per channel capability |
| Inventory mode | shared pool or allotment | Per room type per channel |
| Restrictions to push | which restriction types the channel accepts | Per channel capability |
| Reservation defaults | default guest language, default source and segment, auto-confirm, virtual card handling | Inbound mapping |
| Content sync | push descriptions, photos and amenities | Where channel allows |
| Sync schedule | full refresh time, retry policy, alert thresholds | Health dashboard |

### 8.12 Revenue engine configuration

| Setting | Type | Behaviour |
|---|---|---|
| Comp set | list of competitors with name, source (manual, extranet import, licensed feed), identifier per source, weight | Market position and recommendations |
| Price floors and ceilings | per room type, per season | Guardrails |
| Max daily change | percent | Guardrail |
| Mode | advise, approve, autopilot | Per property, with per-rate-plan override |
| Rules | conditions (dates, weekdays, lead time, occupancy band, pickup, LOS, channel, event tags), actions (set, adjust, open, close, min stay), priority, active window | Evaluated in order, first match or cumulative as configured |
| Promotions | type, code, discount, dates, LOS, channels, caps, stackable | Booking engine and channel promotions |
| Events | name, dates, demand impact, source | Rules input |
| Forecast settings | history window, seasonality, confidence threshold for autopilot | Model behaviour |

### 8.13 Messaging and documents

| Setting | Type | Behaviour |
|---|---|---|
| Message schedule | trigger (booking, N days before arrival, arrival day, N days after check-in, departure eve, departure, N days after), channel (email, SMS, WhatsApp), template, audience filter (source, rate plan, segment) | Scheduler sends via SendGrid or messaging adapter |
| Templates | subject, body, merge fields, language, brand | Visual editor with preview using a sample reservation |
| Documents | invoice, credit note, registration card, confirmation, folio statement; layout, legal footer, VAT number | PDF rendering |
| Sender identities | from name and address, reply-to, SendGrid sub-user per property | Deliverability and branding |

### 8.14 Roles and permissions (default matrix)

| Capability | Owner | Property manager | Front desk | Night auditor | Housekeeping supervisor | Housekeeper | Revenue manager | Finance |
|---|---|---|---|---|---|---|---|---|
| Configure property, room types, rates | yes | yes | no | no | no | no | rates only | no |
| Book, modify, cancel | yes | yes | yes | yes | no | no | no | no |
| Check in and out, take payment | yes | yes | yes | yes | no | no | no | no |
| Refund above threshold | yes | yes | no | no | no | no | no | yes |
| Run night audit | yes | yes | no | yes | no | no | no | no |
| Housekeeping board and tasks | yes | yes | view | view | yes | own tasks | no | no |
| Linen and stock counts | yes | yes | no | no | yes | count only | no | view |
| Pricing rules and promotions | yes | yes | no | no | no | no | yes | no |
| Ledger, exports, accounting | yes | view | no | view | no | no | view | yes |
| Users and roles | yes | property only | no | no | no | no | no | no |
| Audit log | yes | yes | no | no | no | no | no | yes |

Thresholds (refund limit, adjustment limit, discount limit) are per role and per property.

### 8.15 Extra charges, fees and incidentals

Every fee is a configurable **charge item** with a charge code behind it, so it posts, taxes, reports and refunds like any other revenue. Fees are either **automatic** (a trigger and a calculation decide when and how much) or **manual** (staff or the guest add it). Nothing is hard-coded.

**Charge item definition**

| Setting | Type | Default | Rule or behaviour |
|---|---|---|---|
| Code, name per language | text | required | Shown on folio, invoice and guest app |
| Charge code | reference | required | Department, revenue category, tax rule and GL account come from it |
| Pricing basis | enum (fixed, per hour, per night, per person, per person per night, percent of room rate, percent of folio, per unit, tiered, free) | fixed | Tiered uses bands (first hour 10, then 5 per hour) |
| Amount or rate table | money or table | required | Can vary by room type, rate plan, season and day of week |
| Max amount, min amount | money | none | Caps a per-hour fee at a night's rate, for example |
| Tax treatment | inherit from charge code or override | inherit | Damage recharges may be outside the scope of VAT; configurable per item |
| Trigger | enum (manual, early check-in, late check-out, no-show, late cancellation, extra guest, extra bed, cot, pet, parking, resort or facility fee, minibar, damage, lost key, smoking, excess cleaning, unreturned item, late payment, card surcharge where lawful, custom rule) | manual | Automatic triggers evaluated by the fee engine |
| Trigger parameters | per trigger | see below | Thresholds, grace periods, rounding |
| Availability check | boolean | false | Early check-in and late check-out only offer if the room is free and clean or the next arrival allows |
| Approval required | none, supervisor, manager | none | Damage and discretionary fees usually need approval |
| Evidence required | boolean, photo count | false | Damage requires photos and a note before posting |
| Guest consent required | boolean | false | Posts only after the guest accepts in the guest app, at check-in or by signed terms |
| Waivable | boolean, waiver reasons | true | Waiver is a reversing posting with a reason code; waivers are reported |
| Refund rule | enum (refundable, non-refundable, pro rata) | refundable | Used by the refund engine |
| Sellable as upsell | boolean, channels (booking engine, online check-in, guest app, desk) | false | Appears as an add-on with its own inventory cap where relevant |
| Inventory cap | integer per date | none | Limits parking bays, cots, late check-outs per day |
| Charge timing | at posting, at check-out, at night audit, N days after departure | at posting | Post-departure charges use card on file with the consent record |
| Notify guest | boolean, template | true for post-departure | Email or SMS with itemised reason and photos where applicable |
| Dispute window | days | 14 | Guest can dispute from the notification; posting is held or flagged per setting |
| Active, display order | boolean, integer | true, end | |

**Automatic fee triggers and their parameters**

| Trigger | Parameters | Calculation |
|---|---|---|
| Early check-in | Earliest allowed time, grace minutes, free if room ready and occupancy below X percent, bands (before 10:00 fee A, 10:00 to 13:00 fee B) | Compares actual or requested check-in time to property check-in time |
| Late check-out | Latest allowed time, grace minutes, bands by hour, cap at full night, free for loyalty tier or rate plan | Compares actual or requested departure to check-out time; automatic posting when the room is still occupied past grace unless pre-sold |
| No-show | From the no-show policy on the rate plan | Night audit posts it; the reservation moves to no-show |
| Late cancellation | From the cancellation policy | Refund engine computes and posts the penalty |
| Extra guest over standard occupancy | Adult supplement, child supplement per bracket | Pricing matrix posts per night |
| Extra bed, cot | Per night or per stay fee | Posted when added; creates housekeeping task |
| Pet | Per night or per stay, max pets, deposit option | Posted at booking or check-in |
| Parking | Per night, per stay, hourly; bay inventory | Upsell with cap |
| Facility or resort fee | Per night or per stay, mandatory, disclosed on booking engine and channels where required | Auto-posted nightly |
| Minibar and consumables | From housekeeping count | Posted per item at count |
| Lost key or card | Fixed | Manual with one tap at desk |
| Smoking in room, excess cleaning | Fixed or hourly, evidence required | Manual from housekeeping inspection with photos |
| Unreturned item (robe, towel, charger) | Price per item from the linen or stock catalogue | Housekeeping flags missing items at departure inspection |
| Late payment on city ledger | Percent per month or fixed, grace days | Finance posts to the company account |
| Card surcharge | Only where lawful; UK consumer cards are prohibited, so default off | Finance control |
| Custom rule | Condition builder (same engine as pricing rules) plus a charge item | Hoteliers can create fees without Zynthio |

**Damages and incident charges**

| Element | Behaviour |
|---|---|
| Incident record | Created from the housekeeping inspection, maintenance ticket or front desk: room, date, description, category (damage, missing item, excess cleaning, smoking, noise, breach of house rules), photos, staff witness, estimated cost |
| Cost basis | Price list per category and item (replacement cost of a lamp, repaint per wall, professional clean), or a quote attached from maintenance, or actual invoice |
| Approval | Manager approval required above a per-property threshold; approval is logged with the evidence |
| Guest notification | Itemised notice with photos sent before or at posting per the charge timing setting, with the dispute window and the signed terms referenced |
| Posting | Posts to the guest folio or a post-departure incidentals folio; charged to the card on file with the consent record attached |
| Deposits | Optional damage deposit configurable per rate plan or guest segment: pre-authorise and release at check-out, or collect and refund |
| Disputes and chargebacks | Dispute opens a ticket; evidence pack assembled from the incident record, registration card, house rules acceptance and check-in logs |
| Insurance and recovery | Mark incident as claimed, record recovery amount, write-off remainder with reason |
| Reporting | Incidents by category, room and month; recovery rate; average cost per incident; repeat rooms flag maintenance issues |

**Where fees surface**

| Surface | What the guest or staff see |
|---|---|
| Booking engine | Mandatory fees shown in the price breakdown before payment; optional add-ons as upsells with inventory |
| Online check-in | Early check-in, late check-out, parking, breakfast and other upsells priced live against availability |
| Guest app | Request late check-out, add parking, view fees already posted, dispute a charge |
| Front desk | One-tap fee buttons configured per property, with amount preview, waiver and approval flow |
| Housekeeping app | Flag damage, missing items, smoking or excess cleaning with photos; the fee engine prices it |
| Night audit | Auto-posts no-shows, facility fees and unpaid late check-outs; lists fees awaiting approval |
| Channels | Mandatory fees declared to OTAs that require fee disclosure; optional fees excluded from pushed rates |
| Reports | Fee revenue by item, waiver rate and value by user, dispute rate, incident recovery |

### 8.16 Property onboarding wizard

A new property is created through a wizard, not a blank admin form. Every step saves and can be revisited; the property cannot go live until required steps are complete. The super admin or an organisation admin starts it.

| Step | Captures | Required to go live |
|---|---|---|
| 1. Property basics | Legal and trading names, address, phone, email, website, time zone, currency, star rating, property type, link to existing Zynthio site or create one | Yes |
| 2. Legal and tax | Company registration number, VAT number and VAT registration status, invoice numbering prefix and starting number, credit note prefix, legal footer text, visitor levy applicability, 28-day VAT rule on or off | Yes |
| 3. Times and audit | Check-in and check-out times, audit mode (automatic roll or manual), audit time, business date mode | Yes |
| 4. Occupancy defaults | Child age brackets, count infants, max cots, default supplements | Yes |
| 5. Room types and rooms | Guided entry or CSV import of room types with occupancy and beds, then rooms per type; validation of totals | Yes |
| 6. Rate plans and prices | Create BAR plus at least one derived plan; seasonal price entry or CSV import; restrictions | Yes |
| 7. Policies | Cancellation, deposit, guarantee, no-show, children, pets, house rules; templates offered | Yes |
| 8. Taxes and charge codes | VAT rules confirmed, standard charge codes created from a template, GL mapping optional | Yes |
| 9. Payments | Dojo merchant connection and terminal pairing, pay-by-link test, card-on-file consent text | Yes |
| 10. Fees and extras | Choose from a template list (early check-in, late check-out, parking, pet, cot) and set prices | No |
| 11. Housekeeping | Task types and credits from template, sections, linen items and pars | No |
| 12. Channels | Booking.com and Expedia property ids and credentials, iCal URLs, mapping with a test push | No, but recommended before go-live |
| 13. Booking engine | Brand colours, logo, domain or subdomain, languages, currencies, terms link, analytics ids | No |
| 14. Messaging | Sender identity, template review, message schedule from template | No |
| 15. Users | Invite staff by role, link existing Zynthio employees | Yes, at least one property manager |
| 16. Data migration | Import future reservations, guests and companies from the incumbent PMS via CSV templates; parallel-run checklist | No |
| 17. Go-live check | Automated checklist: required steps complete, test booking through the booking engine, test payment and refund, test ARI push, fire sheet renders, audit runs on the training copy | Yes |

The wizard also creates the training-mode copy of the property on completion.

---

## 9. Events and conference module ("Gather")

Covers meetings, conferences, weddings, private dining, function room hire and any event that combines space, catering, equipment and bedrooms. Shares the inventory engine with rooms (a function room is a bookable unit with a calendar), the folio and ledger with the PMS, and the kitchen and bar with the EPOS.

### 9.1 Venue and space configuration

| Setting | Type | Default | Rule or behaviour |
|---|---|---|---|
| Space code, name, description, photos, floor plan | text, media | required | Shown on proposals and the events enquiry page |
| Space type | enum (meeting room, boardroom, conference hall, ballroom, private dining, bar area, terrace, marquee site, whole venue) | meeting room | Filters and reporting |
| Capacities by layout | map layout to max pax (theatre, classroom, boardroom, U-shape, cabaret, banquet, reception, dinner dance) | required | Validates attendee count against chosen layout |
| Area, ceiling height, natural light, access | attributes | empty | Proposal content and filters |
| Fixed equipment | list (screen, projector, PA, lectern, stage, dance floor, Wi-Fi) | empty | Included items on function sheet |
| Combinable spaces | list of (component spaces) | none | A ballroom bookable whole or as two halves; booking the whole blocks the parts and the reverse |
| Setup and breakdown time | minutes before and after | 60, 60 | Blocks the space calendar beyond the event hours |
| Operating hours and licence hours | per weekday | property hours | Validation and late licence fee trigger |
| Noise and curfew rules | time, text | none | Function sheet and contract clauses |
| Minimum spend by day part and weekday | money | none | Enforced on proposals, waivable with approval |
| Room hire tariff | see 9.4 | required | |
| Housekeeping and porter tasks | task types and credits per layout change | preset | Drives setup tasks in the housekeeping app |
| Active, display order, bookable online | boolean, integer, boolean | true, end, false | Online enquiry versus instant booking for small meeting rooms |

### 9.2 Enquiry to contract pipeline

| Stage | What happens | Data captured |
|---|---|---|
| Enquiry | From web form, phone, email or walk-in; auto-acknowledged; assigned to an events coordinator | Organiser, company, event type, dates with flexibility, pax range, budget, spaces of interest, bedrooms needed, source |
| Availability check | Space calendar, bedroom availability, kitchen capacity (covers per service from the EPOS production module) and staffing (rota) checked together | Conflicts shown with alternatives |
| Proposal | Branded PDF and web proposal built from spaces, packages, menus, bedrooms and extras with pricing; multiple options allowed; expiry date | Versioned; opens tracked |
| Provisional hold | Space and bedrooms held with a release date; automatic reminders; auto-release or escalate | Hold expiry, decision date |
| Contract | Terms, deposit schedule, cancellation terms, minimum numbers, attrition and final numbers deadline, e-signature | Signed PDF stored; policies pinned |
| Deposit and payment schedule | Instalments by date or milestone, pay-by-link, card on file | Scheduler collects, reminds and escalates |
| Planning | Function sheet built and revised with the organiser; final numbers, dietary, timings, AV, layout, signage, accessibility | Change log |
| Delivery | Function sheet distributed to kitchen, bar, housekeeping, reception, AV; day-of checklist | Actuals recorded |
| Billing | Master account consolidated with EPOS tabs, bedrooms and extras; invoice; balance collection | Final invoice, credit notes |
| Follow-up | Feedback request, repeat booking offer, pipeline for next year's date | Organiser history |

Pipeline views: kanban by stage, calendar by space, forecast by month with weighted probability, coordinator workload. Lost enquiries capture a reason and competitor.

### 9.3 Function sheet (banquet event order)

One function sheet per event, with a section per space and day. Generated from the booking data and edited in place; every version is stored and the delta is shown.

| Section | Content |
|---|---|
| Header | Event name, organiser and on-site contact, company, coordinator, status, version, date issued, final numbers deadline |
| Timings | Access, setup complete, arrival, each session or course, breaks, carriages, breakdown, per space |
| Layout | Layout per space and time with pax, top table, stage, dance floor, registration desk, cloakroom; diagram attached |
| Catering | Menu per service with course detail, service style (plated, buffet, bowl food, canapés), numbers per menu choice, dietary and allergen matrix per guest or table, children's and crew meals, tea and coffee breaks with times and quantities |
| Beverage | Drinks packages, bar type (cash, account, pre-paid tab with limit), wine per table, toast drinks, licence times, corkage |
| Equipment and AV | Items, quantities, source (in-house or hired), supplier, delivery and collection times, technician on site |
| Accommodation | Bedroom block summary, rooming list status, arrival pattern, VIP rooms, upgrades, breakfast times |
| Staffing | Roles and headcount per time slot (front of house, kitchen, porters, AV, security), linked to rota so the shifts are published from the sheet |
| Billing instructions | What goes to master account, what guests pay individually, deposit received, balance due, PO number, who signs |
| Special requirements | Accessibility, signage, branding, deliveries, parking, security, entertainment, children, photography |
| Sign-off | Organiser approval with e-signature, internal departmental acknowledgement |

Distribution: PDF and live link to kitchen, bar, housekeeping, reception and AV; kitchen receives a production view (covers by course and time with allergens) that feeds the EPOS kitchen screens. Changes after the lock date raise a change notice with cost impact.

### 9.4 Room hire pricing

| Setting | Type | Behaviour |
|---|---|---|
| Tariff structure | per hour, half day, full day, evening, 24-hour, per event, per weekend | Multiple tariffs per space; the engine picks the cheapest valid combination or the one the coordinator selects |
| Day parts | configurable windows (morning, afternoon, evening) with prices | Weekday and season variants |
| Seasonal and weekday pricing | calendar of price bands | Saturdays in summer for weddings |
| Minimum spend offset | boolean, percent | Room hire waived or reduced when food and beverage spend exceeds a threshold |
| Setup and breakdown charging | free, hourly, flat | Beyond included time |
| Overrun charges | per hour or part hour, grace minutes | Posted from actual end time recorded on the day |
| Late licence fee | fixed or hourly | Triggered by licence hours |
| Combinable space pricing | whole price versus sum of parts | Rule when a combined space is sold |
| Discounts | charity, repeat client, agency, midweek | With approval limits |
| Agency commission | percent on hire, on total, on specific components | Agent statements |
| VAT | standard rated; configurable for exempt hirers where applicable | Tax rule per charge code |
| Deposit and cancellation | tiered by lead time (for example 25 percent at contract, 50 percent at 90 days, 100 percent at 30 days) | Policy engine |

### 9.5 Delegate packages and catering

| Element | Settings | Behaviour |
|---|---|---|
| Delegate package | name, day delegate or 24-hour, inclusions (room hire, arrival coffee, mid-morning break, lunch type, afternoon break, water, stationery, AV, Wi-Fi, bedroom, dinner, breakfast), price per delegate, minimum numbers, price bands by volume, upgrade options | Posts as components to the correct departments (hire, food, beverage, rooms) for accurate reporting; 24-hour packages create bedroom reservations in the block |
| Menus | courses, dishes with allergens and dietary tags, price per head, minimum numbers, seasonal availability, kitchen prep time, linked EPOS items | Allergen matrix auto-built per event; items flow to kitchen screens |
| Drinks packages | per head or per bottle, duration, inclusions | Posts to beverage |
| Pre-orders | organiser or guests submit menu choices per attendee online with dietary notes; table plan assignment | Kitchen production counts and place cards |
| Attrition | contracted minimum, final numbers deadline, charge for shortfall | Billing engine |
| Extras catalogue | AV hire, flipcharts, staging, flowers, cake, entertainment, security, cloakroom, parking, printing | From the extra charges engine (8.15) with supplier cost and margin |
| Supplier and hire management | supplier records, purchase orders from the function sheet, delivery schedule, cost reconciliation against invoices | Event profitability |
| Minimum numbers and guarantees | per menu and package | Enforced at contract and final numbers |

### 9.6 Bedroom blocks for events

| Function | Behaviour |
|---|---|
| Block creation | Allotment per room type per night with a negotiated rate plan, cut-off date and release rules (release all, release unsold, convert to transient) |
| Rooming list | Upload or online form; names, sharing, arrival and departure, special requests; creates reservations in the block |
| Individual pickup | Guests book into the block with a code on the booking engine; pickup tracked against allotment; wash factor applied to forecasting |
| Billing split | Rooms to master account, incidentals to guest, or any mix per block |
| Cut-off handling | Reminders to organiser, auto-release at cut-off with notice |
| Reporting | Pickup versus block, attrition exposure, block revenue |

### 9.7 Billing and ledger for events

| Function | Behaviour |
|---|---|
| Master account | One event folio with windows for organiser, company, sponsor and individual payers |
| EPOS link | Bar and catering tabs on the till post to the event account by event code; cash bar sales attributed to the event for profitability without posting to the account |
| Deposit schedule | Instalments collected by pay-by-link or card on file; receipts and VAT treatment for deposits handled correctly |
| Actuals versus contract | Final numbers, overruns, extras and attrition applied automatically after the event; coordinator reviews before invoicing |
| Invoicing | Proforma before the event, final VAT invoice after, split invoices per payer, PO references, credit notes |
| Profitability | Revenue by component against supplier costs, staffing cost from the rota and kitchen cost from EPOS recipes; margin per event and per event type |
| Commission | Agency statements and payment tracking |

### 9.8 Events function catalogue

| Function | What it does |
|---|---|
| Space calendar | Shows every space by day with holds, confirmed events, setup and breakdown blocks, and combinable space logic |
| Enquiry capture | Takes enquiries from web, phone or email into a pipeline with auto-acknowledgement and assignment |
| Availability finder | Checks spaces, bedrooms, kitchen capacity and staffing together and offers alternatives |
| Proposal builder | Assembles branded proposals from spaces, packages, menus, bedrooms and extras with expiry and tracking |
| Provisional holds | Holds inventory with release dates, reminders and automatic release |
| Contract and e-signature | Generates contracts from templates with pinned terms and collects signatures |
| Payment schedules | Creates instalments and collects them automatically with reminders |
| Function sheet editor | Builds, versions and distributes the banquet event order with departmental views |
| Pre-order portal | Lets organisers and attendees choose menus and declare dietary needs online |
| Allergen matrix | Compiles allergens per guest, table and course for the kitchen |
| Table planner | Drag-and-drop seating plan linked to pre-orders and place cards |
| Staffing from sheet | Publishes required shifts to the rota module and checks labour cost against the event budget |
| Supplier orders | Raises purchase orders for hired items and reconciles supplier invoices |
| Bedroom blocks | Manages allotments, rooming lists, pickup and release for event bedrooms |
| Event master account | Consolidates all charges with payer windows and EPOS tabs |
| Actuals reconciliation | Applies final numbers, overruns and attrition before invoicing |
| Event invoicing | Issues proforma, final and split VAT invoices with PO references |
| Event profitability | Reports margin per event with revenue, supplier, labour and food costs |
| Pipeline and forecast | Kanban, weighted forecast, conversion and lost-reason reporting |
| Online meeting room booking | Instant booking and payment for small meeting rooms with day delegate rates |
| Organiser portal | Gives the organiser their proposal, contract, function sheet, payments, rooming list and pre-orders in one place |
| Post-event follow-up | Feedback request, repeat booking offer and anniversary reminders |

---

## 10. Operational sheets and reporting

Operational sheets are the printed or on-screen lists a hotel runs its day from. Reports are the analysis. Both are generated from the same data, both are schedulable, printable, exportable (PDF, CSV, Excel) and visible on any device, and every sheet has a configurable column set, sort, filter and grouping saved per property and per user.

### 10.1 Daily operational sheets

| Sheet | Who uses it | Columns and content | Generated | Notes |
|---|---|---|---|---|
| Arrivals | Front desk, porters | Room, room type, guest name, pax (adults, children, infants), ETA, nights, rate plan, source and channel, balance and deposit status, pre-check-in status, VIP level, preferences, special requests, traces, cot or extra bed, pets, car registration, company, flags (blacklist, repeat guest) | Live, plus a printable snapshot at any time | Groups by ETA band or room type; shows unassigned rooms in red; one-tap check-in |
| Departures | Front desk, housekeeping | Room, guest, pax, checkout time requested, late check-out sold, balance due, payment method on file, invoice recipient, next arrival into the room and their ETA, express check-out status | Live | Flags rooms with a same-day arrival for priority cleaning |
| In-house guests | All departments | Room, guest, pax, arrival and departure dates, rate plan, meal plan, balance, credit limit status, DND, VIP, allergies and dietary, traces, room charge allowed | Live | Basis for fire sheet, breakfast and dinner sheets |
| Fire and emergency roll call | Duty manager, fire marshal | Room, guest names for every registered occupant, pax, mobility or assistance needs, allergies not needed, checked-in time, staff on shift from rota with roles, contractors signed in, events in progress with pax per space | Live, auto-printed at a configured time (for example 23:00) and on demand; also available offline on the duty manager's phone | Registered occupants come from registration cards, so online check-in must capture every guest name; the sheet is the legal roll call |
| Breakfast sheet | Kitchen, restaurant | Room, guest, pax by adult and child, meal plan (included, pre-paid, pay on day), breakfast time requested, dietary and allergens, table preference, in-room breakfast orders, event delegates with breakfast included | Night audit output, refreshed live | Totals by time slot drive kitchen prep; checked off on a tablet as guests arrive; no-shows and chargeable extras post to folios |
| Dinner sheet | Kitchen, restaurant | Room, guest, pax, package dinner allowance (value or courses), table reservation time, dietary, pre-orders, events dinners by space | Night audit output, refreshed live | Links to the restaurant's table bookings in EPOS; allowance redemption on the till |
| Housekeeping sheet (per housekeeper) | Housekeepers | Section, rooms in priority order, task type, credits, current status, expected arrival time, occupancy and pax, linen change due, cot or extra bed to set up, guest requests, DND, do-not-move items, maintenance notes | Generated after night audit from the assignment engine, live on the app, printable | Supervisor version shows all housekeepers and progress; inspection results write back |
| Room status report | Housekeeping, front desk | Every room with status, housekeeping status, assigned housekeeper, inspection, OOO and OOS, discrepancies (sleep-outs, skips: front desk says occupied, housekeeping says vacant) | Live | Discrepancy report drives a physical check |
| Linen and laundry sheet | Housekeeping | Clean issued, dirty collected and sent per item, returned and rejected, counts by pantry, par shortfalls, condemned | Daily | Reconciles to laundry delivery notes |
| Maintenance sheet | Maintenance | Open tickets by priority and room, due today, OOO returns due, preventive tasks due | Live | |
| VIP and special attention | Management | VIPs arriving, in house and departing, with preferences, amenities to place, notes | Daily | |
| Traces and alerts | Front desk, all | Date-stamped actions attached to reservations or rooms (call guest about deposit, place flowers, confirm taxi), department, due, done by | Live | Overdue traces escalate |
| Deposits due and taken | Front desk, finance | Reservations with deposit due today or overdue, amount, policy, collection method, reminder status; deposits taken today with method and receipt | Daily and live | One-click pay-by-link resend; links to deposit ledger |
| Balances and credit limits | Front desk, finance | In-house folios over credit limit, high balances, cards on file expiring, auths expiring | Live | Night audit pre-check |
| No-shows and cancellations | Front desk, revenue | Yesterday's no-shows with policy applied and charge status; cancellations with penalty applied, channel and reason | Night audit output | |
| Cashier and shift report | Front desk, finance | Payments by method, cash drawer opening, paid-outs, closing count, variance, refunds, per user and shift | On shift close | Drawer count entered on screen; variance needs a reason |
| Night audit pack | Night auditor, GM | Manager's report (occupancy, ADR, RevPAR, revenue by department, payments, ledger balances), trial balance, room and tax posting log, rate check exceptions (rate differs from plan), unposted EPOS checks, open folios of departed guests, zero-rate rooms, complimentary and house use, overbooking and oversell check, audit variances, business date roll confirmation | Night audit | Emailed to a configurable list |
| Rate and rooms check | Night auditor, revenue | Reservations where the posted rate differs from the rate plan, manual rates, discounts over threshold, upgrades given free | Night audit | |
| Pickup report | Revenue, GM | Rooms and revenue booked yesterday by arrival month and channel, cancellations netted | Daily | |
| Forecast sheet | GM, kitchen, housekeeping | Next 14 and 30 days: rooms sold, arrivals, departures, pax in house, breakfasts and dinners expected, events pax, staffing suggestion from the rota module | Daily | This is the sheet the rota is built from |
| Guest messages and parcels | Front desk | Messages taken, parcels received and handed over, with signatures and timestamps | Live | |
| Wake-up calls and requests | Front desk, night | Room, time, status | Live | |
| Car park sheet | Front desk, security | Registrations, bays, permits, in-house or event, overstay | Live | |
| Lost property register | Housekeeping, front desk | Item, photo, where found, date, stored at, claimed by, returned or disposed | Live | Retention period per property |
| Events today | All departments | Function sheets for today with timings, spaces, pax, catering counts and staff, in one view | Daily | |
| Group arrivals and rooming | Front desk, porters | Group name, rooming list, pre-assigned rooms, keys cut, luggage tags, master account | Per group | Group check-in mode issues all keys at once |

### 10.2 Reporting catalogue

All reports support date ranges, comparison periods (last year, budget, forecast), property and group level roll-up, drill-down to the reservation or posting, scheduling and export.

**Performance**

| Report | Measures and breakdowns |
|---|---|
| Daily, monthly and yearly performance | Occupancy, rooms sold, ADR, RevPAR, TRevPAR, revenue by department, by day and month, versus last year and budget |
| Pace and on-the-books | Rooms, revenue and ADR on the books for future dates versus same point last year |
| Pickup | Daily change in rooms and revenue by arrival date, channel and segment |
| Segment, source and channel mix | Revenue, ADR, length of stay and lead time by market segment, source, channel, company and agent |
| Cost of acquisition | Commission and channel fees by channel against revenue; direct share |
| Rate plan performance | Revenue and nights by rate plan; package uptake |
| Length of stay and lead time distribution | Histograms by month |
| Day of week and seasonality | Occupancy and ADR by weekday and month |
| Cancellations and no-shows | Counts, value, rate, by channel, lead time and policy |
| Upsell and ancillary | Revenue by upsell item, conversion by surface (booking engine, online check-in, desk) |
| Guest geography and demographics | Nationality, origin, repeat rate |
| Reviews and reputation | Scores by platform over time against occupancy, from the existing reputation module |

**Revenue management**

| Report | Measures |
|---|---|
| Market position | Rank and price index against comp set by date |
| Rate parity | Violations by channel and date |
| Recommendation performance | Accepted, rejected and applied recommendations and the revenue outcome |
| Displacement and group evaluation | Transient revenue displaced by blocks |
| Forecast accuracy | Forecast versus actual occupancy and revenue by lead time |

**Finance**

| Report | Measures |
|---|---|
| Manager's daily report | The night audit summary |
| Revenue journal | Postings by charge code, department and tax code |
| Tax report | VAT by rate and levy collected, with exemptions |
| Payments and settlements | By method, terminal, user and settlement batch; reconciliation to Dojo |
| Guest ledger | In-house balances and ageing |
| City ledger and debtors | Company and agent balances, ageing, statements, credit limit breaches |
| Deposit ledger | Advance deposits held, applied and refunded; liability at period end |
| Refunds and adjustments | By user, reason code and value |
| Waivers and discounts | By user and reason |
| Commission payable | By agent and channel |
| Invoice register | Sequential list with status |
| Trial balance and GL export | Daily and period summaries by account |
| Profit and loss by department | Revenue less direct costs where EPOS recipe cost and rota labour are available |

**Operations**

| Report | Measures |
|---|---|
| Housekeeping productivity | Minutes and credits per room, rooms per shift, inspection pass rate, by housekeeper |
| Room status discrepancies | Sleep-outs and skips by day |
| Linen usage and cost | Items per occupied room, shrinkage, laundry cost |
| Maintenance | Tickets opened and closed, time to resolve, by category and room; OOO nights lost |
| Incidents and damages | By category, recovery rate |
| Lost property | Open and returned |
| Guest requests | Volume and response time by type |
| Labour versus revenue | Rota hours and cost by hour against rooms and F&B revenue, from the existing rota module |
| Breakfast and dinner covers | Expected versus served, no-shows, chargeable extras |

**Events**

| Report | Measures |
|---|---|
| Pipeline and conversion | Enquiries by stage, conversion rate, lost reasons, coordinator |
| Space utilisation | Occupied hours per space by day part |
| Event profitability | Revenue, costs and margin by event and type |
| Bedroom block pickup | Pickup against allotment and attrition exposure |

**Compliance and audit**

| Report | Measures |
|---|---|
| Registration compliance | Guests without a completed registration card or required ID |
| Audit log | Configuration and financial changes by user |
| User activity | Logins, failed logins, permission changes |
| GDPR | Retention jobs run, erasure requests, consent status |
| Channel sync health | Push failures, latency, acknowledgement time by channel |

### 10.3 Report builder

A custom report builder lets a manager pick a dataset (reservations, postings, payments, rooms, tasks, events), choose columns, filters and grouping, add calculated fields, save the report, share it with roles, and schedule it. Saved reports can be pinned to the existing KPI dashboard as tiles.

---

## 11. Smaller functions that are easy to miss

| Function | What it does |
|---|---|
| Traces | Date-stamped to-dos attached to a reservation, guest or room, routed to a department, with escalation when overdue |
| Guest messages and parcels | Logs messages and deliveries for in-house and arriving guests with hand-over signature |
| Wake-up calls | Scheduled per room with completion status |
| Group check-in | Checks in a rooming list in one action, issues all keys, prints luggage tags and welcome letters |
| Rate override and discount approvals | Threshold-based approvals with reason codes, reported nightly |
| Complimentary and house use | Zero-rate rooms with reason and approver, reported separately from paid occupancy |
| Day use and hourly rooms | Same-day in and out with its own rate plans and housekeeping task |
| Sleep-out and skip handling | Discrepancy workflow between front desk and housekeeping status |
| Guest ID verification | Document capture with optional automated checks, stored encrypted, purged on retention |
| Multi-property central view | Group-level arrivals, availability, rates and reports across properties; central reservations with cross-selling when one property is full |
| Cross-property guest profiles | One profile and preferences across the group with consent |
| Shared booking engine for a group | One site selling all properties with a map and compare view |
| Rate and availability API for the hotel's own website | Embeddable widget and REST endpoints |
| Tour operator and allotment contracts | Contracted allotments with release dates and net rates, manifest import |
| Gift vouchers and prepaid stays | Sold online, tracked as liabilities, redeemed on folio |
| Loyalty and member pricing | Tags, tiers and member-only rates on the booking engine |
| Guest Wi-Fi and TV integration | Pushes guest name and check-out to Wi-Fi and TV systems and clears on departure |
| Door lock integration | Key cards, mobile keys and PIN codes issued and revoked by reservation |
| Telephone and call accounting | Posts call charges where phones still exist |
| Parking and EV charging | Bay inventory, permits and charges |
| Shuttle and taxi bookings | Requests and traces for transport |
| Early departure and stay extension pricing | Re-prices nights and applies early departure fees per policy |
| Room move with folio and key handling | Moves charges and reissues keys in one flow |
| Dirty room sale protection | Prevents selling a dirty or OOO room without an override |
| Overbooking resolution | Shows oversold dates, suggests walk candidates and partner hotels, records walk costs |
| Walk-in rate suggestions | Shows the revenue engine's current recommendation at the desk |
| Guest feedback and complaints log | Records issues, compensation given and resolution, feeds reputation and incident reports |
| Document archive | Signed registration cards, contracts, invoices and ID documents retrievable per reservation |
| Data export and portability | Full export per property for audits or exit |
| Offline mode for front desk | Read-only arrivals, in-house and fire sheet cached on the desk device if the connection drops |
| Printer and hardware management | Receipt and document printers, key encoders and terminals per device, through the existing Jarltech hardware range |

---

## 12. Market comparison and remaining gaps

Vendors reviewed (from their own sites and review listings, November 2026): Mews, Cloudbeds, Guestline Rezlynx (Access Group), eviivo Suite, RoomRaccoon, Stayntouch, Oracle OPERA Cloud, Apaleo, Little Hotelier (SiteMinder), Clock PMS, Chekin. Vendor claims are marketing and not independently verified.

### 12.1 What the market now treats as baseline

| Capability | Who leads on it | Zynthio Stay status |
|---|---|---|
| Embedded payments with tokenisation, pay-by-link, wallets, chargeback tooling | Mews, Cloudbeds, RoomRaccoon, Guestline GuestPay | Covered via Dojo (sections 2 and 8.15) |
| Online check-in with ID capture and e-signature, kiosk mode | Mews Kiosk, Cloudbeds, Stayntouch, Chekin | Covered |
| Unified guest inbox across OTA messaging, email, SMS and WhatsApp | eviivo, Cloudbeds (Whistle), Mews | **Gap, added below** |
| AI assistant for guest questions with human escalation | eviivo, Cloudbeds, Mews | **Gap, added below** |
| Bookable non-room spaces and services by the hour (parking, coworking, bikes, meeting rooms) | Mews Spaces | Partly covered (meeting rooms, parking); **generalised below** |
| Upsell menus pre-arrival and in stay | RoomRaccoon RaccoonUpsell, Mews | Covered |
| Pricing rules on occupancy, restrictions and comp set | Cloudbeds, RoomRaccoon RaccoonRev, Mews | Covered and deeper (section 2.4 of the plan) |
| Guest profile merge, de-duplication and segmentation for marketing | Cloudbeds, Mews | Partly covered; **CRM campaigns added below** |
| Open API, webhooks and an app marketplace | Mews, Apaleo, Cloudbeds | API and webhooks covered; **developer portal added below** |
| Multi-property central management | Mews, Cloudbeds, OPERA Cloud, Guestline | Covered in section 11 |
| Conference and banqueting | Guestline, OPERA Cloud | Covered and deeper (section 9) |
| Two-way EPOS with room posting | Guestline EPoS, Mews POS | Covered, and Zynthio owns the EPOS |
| Owner and trust accounting for managed units | eviivo Owner Manager | **Gap, optional module below** |
| Continuous audit without a night audit step | Mews, Apaleo | **Design choice added below** |
| Hostel bed-level inventory | Cloudbeds, Mews | Covered via per-bed room type category |
| Labour and rota integrated with the PMS | None of the above | Zynthio's differentiator |

### 12.2 Gaps found and decisions

| Gap | What to add | Decision |
|---|---|---|
| Unified guest inbox | One inbox per property pulling Booking.com and Expedia messaging APIs (via the aggregator), Airbnb threads, email, SMS, WhatsApp Business and the guest web app; conversation linked to the reservation; templates, assignment to staff, SLA timers, read receipts; replies go back on the channel the guest used | Add to Guest Journey, phase 3 |
| AI guest assistant | Answers common questions (check-in time, parking, breakfast, directions, Wi-Fi) from the property's knowledge base, in the guest's language, on the website, WhatsApp and the guest app; drafts replies for staff in the inbox; hands over to a human on uncertainty, complaints or payment topics; every answer logged; opt-in per property | Add, phase 5; built on the Claude API with the property knowledge base and strict escalation rules |
| Bookable resources beyond rooms | Generalise the inventory engine to any resource with a calendar: meeting rooms, parking bays, EV chargers, coworking desks, bikes, spa treatment slots with therapist and room, tennis court, kayak; hourly, daily or slot pricing; sold on booking engine, guest app and desk; posts to folio or standalone sale | Add as "Resources" in Inventory, phase 4 |
| Spa and activity appointments | Treatment menu, duration, therapist skills and rota, treatment rooms, prep and cleanup buffers, deposits, in-stay booking from the guest app, commission to therapists | Phase 6 via Resources |
| CRM and marketing campaigns | Segments built from profile and stay data (lapsed guests, weekend leisure, corporate bookers, high spenders), campaign builder for email and SMS with consent enforcement, win-back and anniversary automations, attribution to bookings; integration with Klaviyo or Mailchimp for hotels that already use them | Add to Guest Journey, phase 5, using SendGrid |
| Abandoned booking recovery | Capture email at step one of the booking engine, send a recovery message with the held price, track conversion | Add to Booking Engine, phase 3 |
| Booking engine analytics and conversion tracking | GA4 and Meta pixel events, funnel report, metasearch cost per booking, A/B test of rate display | Add, phase 3 |
| Paid metasearch management | Google Hotel Ads commission or CPC campaigns, Trivago, TripAdvisor bidding with budget caps and ROI reporting | Phase 7 |
| Guest self-service modification | Change dates, room type and occupancy, add nights or extras, within policy and availability, on the guest portal; re-prices and collects difference | Add, phase 3 |
| Digital compendium | QR in room opens the property guide: Wi-Fi, menus, local recommendations, house rules, services to order; replaces printed folders | Add to Guest app, phase 4 |
| In-stay satisfaction pulse | One-question survey the morning after arrival so problems are fixed before checkout; routes to duty manager | Add, phase 4 |
| Digital tipping | QR or in-app tips to housekeeping and staff, routed to tronc in the rota module | Add, phase 4 |
| Loyalty programme | Points or tier scheme, member rates, recognition at check-in, redemption on folio; cross-property for groups | Phase 6 |
| B2B portals | Travel agent portal with net rates and allotments; corporate booking portal with negotiated rates, traveller profiles, PO and cost centre capture, consolidated monthly invoice | Phase 6 |
| Group booking landing page | A per-event page where attendees book into the block at the negotiated rate, with pickup visible to the organiser | Add to Events, phase 5 |
| Owner and trust accounting | For serviced apartments and let rooms owned by third parties: owner statements, management fee rules, trust ledger, owner portal and payouts | Optional module, only if a customer needs it |
| Continuous audit | Allow properties to opt out of a manual night audit: business date rolls automatically, postings are continuous, the audit pack is still generated and exceptions are queued for the morning | Design both from the start; default to automatic roll with an optional manual gate |
| Developer portal and marketplace | Public API docs, sandbox tenants, API keys and scopes, webhook catalogue with retries, partner listing in the integrations hub | Phase 6 |
| Multi-currency display | Booking engine shows prices in the guest's currency with a disclosed rate; settlement stays in GBP | Add, phase 3 |
| Long-stay and serviced apartment billing | Weekly or monthly periodic invoicing, utilities and cleaning schedules, 28-day VAT rule (already covered), tenancy-style documents | Phase 6 |
| Energy and IoT | Occupancy status to heating and lighting controls, door sensors feeding housekeeping status | Phase 7, via integrations hub |
| BI connector | Scheduled export to a warehouse or Power BI and Looker Studio connectors for groups that want their own dashboards | Phase 6 |
| Accessibility | WCAG 2.2 AA on booking engine, guest app and staff web; large-text mode on the housekeeping app | Build in from phase 1 |
| Welsh and other UK languages | Booking engine and guest documents in Welsh for Welsh properties; Gaelic optional | Phase 4, templates only |
| Procurement for F&B | Already in scope for the EPOS stock module; link supplier invoices to event profitability | Confirm against the existing EPOS |
| Health, safety and compliance logs | Fire alarm tests, legionella, food safety checks: already part of Zynthio digital compliance; add hotel-specific templates (fire sheet printed, pool checks, lift inspections) | Templates only |

### 12.3 Where Zynthio Stay should deliberately be different

- **Labour inside the PMS.** No competitor shows rota cost against rooms and F&B revenue by hour, or builds staffing suggestions from arrivals, departures, covers and events. This is the headline.
- **One ledger across rooms, F&B, events and retail**, because Zynthio owns the EPOS. Competitors integrate with third-party tills and lose fidelity.
- **Housekeeping, linen and compliance in the staff app staff already clock into**, instead of a separate app.
- **Pricing that explains itself.** Every automated price change carries plain-English reasons; autopilot is opt-in behind guardrails.
- **Priced for pubs with rooms and small independents** at a per-room rate with low minimums, where competitors' minimums start at a few hundred pounds a month once payments and channel manager are added.
- **UK first.** VAT 28-day rule, visitor levies, Immigration (Hotel Records) Order registration fields, tronc, Cyber Essentials Plus for public-sector buyers, Dojo rates.

---

## 13. Booking engine search algorithm

The same algorithm serves the public booking engine, the guest portal (modify booking), the front desk quick book, the channel ARI push (which precomputes the per-occupancy table for each room type, rate plan and date) and the public availability API. One implementation, several callers.

### 13.1 Inputs

| Input | Notes |
|---|---|
| property_id (or list for group search) | |
| arrival, departure | Dates in property time zone; nights = departure minus arrival; 0 nights means day use and is only allowed if the property enables it |
| parties | One or more rooms requested, each with adults, children with ages, infants, requested extra beds and cots |
| promo_code, member flag, company or agent code | Unlock private rate plans |
| channel context | booking engine, guest portal, desk, API client; decides which rate plans are visible and which markups apply |
| currency and language | Display only |
| flexible dates | Optional plus or minus N days for a calendar view |

### 13.2 Steps

1. **Validate the request.** Arrival not in the past (desk may override for same-day late night), nights within property max, each party within global caps (adults 1 to 10, children 0 to 10). Reject with a specific error code.
2. **Build the date set.** Stay nights = each night from arrival to departure minus one. Fetch for the property: room types (active, sellable in this context), rate plans (active, sellable in this context, matching codes), DailyRate rows, Restriction rows, Availability rows for the stay nights, policies and tax rules in force on the arrival date, promotions active on the sell date.
3. **Filter room types by occupancy per party.** For each party and room type, apply the pricing matrix validation (8.4): adults <= max adults; children <= max children; adults plus children <= max total plus allowed extra beds; adults >= min occupancy; infants <= max cots when the property does not count infants. Children are counted by the applicable bracket; children above the oldest bracket count as adults. Keep room types that pass. If none pass for a party, compute a suggestion: the smallest set of room types that can hold the party split across rooms (greedy by capacity), and return it as "split your party".
4. **Filter by availability.** For each surviving room type, availability on every stay night must be >= the number of parties assigned to that room type in the current combination. Virtual room types use the minimum of their components. Shared inventory groups reduce together. Allotments apply when the channel context has one. Availability includes the configured overbooking allowance only for channels where the property allows it. Rooms out of order are already excluded; rooms out of service are available for sale.
5. **Filter rate plans per room type.** Rate plan sells this room type; sell dates include today; stay dates include every night; booking window satisfied (lead time between min and max days); length of stay within plan min and max; weekday sellable for the arrival day (or for every night, configurable); channel context allowed; private plan unlocked by code or membership; company or agent plan attached to the supplied code.
6. **Apply restrictions.** For each room type and rate plan: no stop sell on any night at room-type level or rate-plan level; closed to arrival not set on the arrival night; closed to departure not set on the departure date; min stay arrival satisfied by the arrival night's value; min stay through satisfied by the maximum of all nights' values; max stay satisfied by the minimum of all nights' values; release period: arrival must be at least N days away.
7. **Price each candidate.** For each room type, rate plan and party:
   - For each night, resolve the DailyRate (explicit row, else derived from the parent plan, else the plan has no price for that night and the candidate is dropped).
   - Determine the per-occupancy price: explicit price for the adult count if present, else base price for standard occupancy plus adult supplement times (adults minus standard), with single-use discount when adults are below standard.
   - Add child supplements per bracket per child, honouring "free up to N".
   - Add extra bed and cot supplements as separate lines.
   - Apply length-of-stay band pricing if the plan defines it.
   - Apply package components: inclusive components do not change the price but are recorded; priced components are added per the component rules.
   - Apply promotions in configured order: percentage then fixed, stackable or exclusive, with caps; record the applied promotion.
   - Apply channel markup for the channel context (booking engine normally 0 or a direct discount).
   - Apply tax rules per charge code: room, supplements, packages and mandatory fees each carry their own tax; inclusive or exclusive display per property setting.
   - Add mandatory fees from the fee engine (facility fee, visitor levy) as disclosed lines.
   - Round per the property rule at the line level, then sum.
   - Produce: nightly breakdown, total, taxes, fees, deposit due now (from the deposit policy), cancellation text (from the pinned policy), payment timing, inclusions.
8. **Rank.** Default order: rate plan display order, then total ascending within a room type; room types by display order. Options: cheapest first, recommended (configurable weighting of margin and conversion), by room type. Mark the cheapest flexible and cheapest non-refundable option per room type for the comparison UI.
9. **Upsell suggestions.** For each room type shown, compute the next room type up by display order that is available and show the price difference. Attach eligible add-ons from the fee engine with inventory checks.
10. **Hold.** When the guest proceeds, create a soft hold on the chosen room type and rate plan for the stay nights for a configurable number of minutes (default 15), decrementing availability for other searches. Holds expire automatically and are not pushed to channels unless the property enables hold pushes.
11. **Re-price at checkout.** Re-run steps 5 to 7 for the chosen candidate immediately before payment; if the total changed, show the difference and ask the guest to confirm. Never charge a stale price.
12. **Commit.** Create the reservation with pinned policy versions, nightly prices and components, guest profile, payment record and promotion usage, release the hold, and emit reservation.created, which triggers the ARI delta push.

### 13.3 Flexible dates and calendar

For plus or minus N days, run steps 2 to 7 once per candidate arrival date with the same nights, in parallel, and return the cheapest available total per date for a calendar strip. Cache per property, nights, party signature and date for 60 seconds; invalidate on any ARI change for the affected dates.

### 13.4 Multi-room and group search

For several parties, compute per-party candidate lists, then assemble combinations where availability per room type per night covers the count of parties assigned to it. Cap combinations shown (for example the three cheapest and the one with all parties in the same room type). Groups above a configurable size (default 8 rooms) are routed to an enquiry form that feeds the events pipeline.

### 13.5 Performance targets

| Measure | Target |
|---|---|
| Single search, 20 room types, 10 rate plans, 7 nights | under 300 ms server time |
| Calendar strip of 15 dates | under 800 ms |
| Availability cache consistency after a booking | under 2 seconds to channels, immediate on the booking engine |
| Hold expiry | within 60 seconds of the deadline |

### 13.6 Edge rules

- A rate plan that is derived from a closed parent is closed.
- A child whose age equals the upper bound of a bracket belongs to that bracket.
- Infants never consume availability or a bed space unless the property counts them.
- When the departure date is closed to departure but the arrival night is closed to arrival for the next day, the stay is still valid if no night in the stay is stop-sold.
- Day use (0 nights) prices from a dedicated day-use rate plan and creates a same-day housekeeping task.
- Prices shown in a foreign currency carry the rate and timestamp used; the reservation stores GBP.
- A promo code that fails validation returns a specific reason (expired, not valid for these dates, minimum stay not met, usage cap reached) rather than silently ignoring it.
- The desk context can bypass booking window, weekday and release rules with an override permission, and the override is logged.

---

## 14. Acceptance tests

Format: ID, Given, When, Then. Every test is automated at the service layer unless marked UI. A module is accepted for release when all its tests pass on a seeded test property and a hotelier from a design-partner property has signed off the UI tests.

### 14.1 Inventory and rates

| ID | Given | When | Then |
|---|---|---|---|
| INV-01 | A room type with 10 rooms, 1 out of order on date D | Availability is read for D | 9 available |
| INV-02 | A room type with 10 rooms and overbooking allowance 1 for the booking engine | Availability is read for the booking engine | 11 available; for a channel without allowance, 10 |
| INV-03 | A virtual family suite made of rooms 101 and 102 | 101 is sold on D | Family suite availability on D is 0 |
| INV-04 | Twin and double share one inventory group of 5 rooms | 3 twins sold on D | Double availability on D is 2 |
| INV-05 | Rate plan B derived at minus 10 percent from plan A | A's price changes from 100 to 120 on D | B's price on D becomes 108 within one second and an ARI delta for B is queued |
| INV-06 | Stop sell set on room type on D at room-type level | Any rate plan is searched including D | No results for that room type |
| INV-07 | Min stay through 3 on D | Search for 2 nights including D | Rejected; search for 3 nights including D | Accepted |
| INV-08 | A room is set OOS (not OOO) | Availability read | Unchanged; auto-assign skips the room |
| INV-09 | Nightly verification job runs | Availability has drifted by a manual DB change | Drift is detected, corrected and alerted |
| INV-10 | Bulk rate edit for Fridays and Saturdays in July | Applied | Only those dates change; audit log records before and after values |

### 14.2 Occupancy and pricing

| ID | Given | When | Then |
|---|---|---|---|
| PRC-01 | Room type standard occupancy 2, max adults 3, adult supplement 20, base 100 | Priced for 3 adults | 120 per night |
| PRC-02 | Same, single-use discount 15 | Priced for 1 adult | 85 |
| PRC-03 | Child brackets 0 to 2 infant, 3 to 12 child; child supplement 15; free up to 1 child | 2 adults, 2 children aged 5 and 9 | 100 plus 15 = 115 |
| PRC-04 | Max children 1 | 2 adults and 2 children aged 5 and 9 | Room type excluded; split suggestion returned |
| PRC-05 | Child aged 12 with bracket 3 to 12 | Priced | Counted as a child, not an adult |
| PRC-06 | Child aged 13 | Priced | Counted as an adult |
| PRC-07 | Infant with property not counting infants, max cots 1 | 2 adults and 1 infant in a max total 2 room | Allowed; cot task created; cot fee posted if configured |
| PRC-08 | Explicit per-occupancy prices 1 adult 90, 2 adults 100, 3 adults 130 | Priced for 3 adults | 130, supplements ignored |
| PRC-09 | Per-person pricing mode, 25 per person | 4 persons | 100 |
| PRC-10 | VAT 20 percent exclusive display, room 100, visitor levy 2 per night | Priced for 1 night | Room 100, VAT 20, levy 2, total 122, with lines itemised |
| PRC-11 | Stay of 30 nights with the 28-day VAT rule enabled | Priced | Nights 29 and 30 taxed on the reduced value per HMRC rule and shown on the invoice |
| PRC-12 | Rounding rule nearest 1 | Price computes to 107.4 | 107 |
| PRC-13 | Channel push for Booking.com for a room type with max adults 3 | ARI generated | Prices for 1, 2 and 3 adults supplied; children flattened per channel capability |

### 14.3 Booking engine search

| ID | Given | When | Then |
|---|---|---|---|
| BE-01 | Arrival yesterday | Search | Error "arrival in the past" |
| BE-02 | Plan with booking window min 2 days | Search for tomorrow | Plan absent; other plans present |
| BE-03 | Plan sellable Fridays only | Search arriving Saturday | Plan absent |
| BE-04 | Promo code expired | Search with the code | Results without promo plus reason "code expired" |
| BE-05 | Promo 10 percent stackable with early bird 5 percent fixed order percent then fixed | Priced | Applied in that order and recorded on the quote |
| BE-06 | Two parties, 2 adults each, one room type with 1 room left | Search | Combination offers two different room types or a split; never two of the same |
| BE-07 | Guest holds a room then waits past the hold time | Hold expires | Availability restored; checkout re-price shows new total if changed |
| BE-08 | Price rises between search and payment | Checkout | Guest sees old and new total and must confirm; no charge at stale price |
| BE-09 | Flexible dates plus or minus 3 | Search | Seven totals returned; each equals a direct search for that date |
| BE-10 | Group of 9 rooms requested | Search | Redirected to enquiry; enquiry appears in events pipeline |
| BE-11 | Property displays prices inclusive of VAT | Search | Totals inclusive; breakdown still shows VAT amount |
| BE-12 | Foreign currency display EUR | Search | EUR shown with rate and time; reservation stored in GBP |
| BE-13 | Search completes | Timing | Under 300 ms server time for the reference dataset |
| BE-14 | Abandoned at step 2 with email captured | 60 minutes pass | Recovery email sent once with the quoted price and link |

### 14.4 Reservations and front desk

| ID | Given | When | Then |
|---|---|---|---|
| RES-01 | A confirmed reservation | Check-in | Status checked in, room status occupied, key issued event emitted, pre-auth taken if policy requires |
| RES-02 | Room is dirty | Attempt check-in without override | Blocked with message; with override permission, allowed and logged |
| RES-03 | Reservation modified from 2 to 3 nights | Saved | Night 3 priced on the pinned rate plan rules, availability decremented, ARI delta queued, housekeeping tasks regenerated |
| RES-04 | Room move from 101 to 205 mid-stay | Confirmed | Folio keeps postings, new key issued, 101 set dirty, 205 occupied, both housekeeping tasks updated |
| RES-05 | Guest departs one night early with early departure fee policy | Checked out | Unused night removed, fee posted per policy, availability restored |
| RES-06 | Walk-in at 23:30 | Quick book for tonight | Allowed; business date logic assigns tonight correctly |
| RES-07 | Cancel inside penalty window | Cancelled | Penalty computed from pinned policy, posted, refund of remainder issued, channel notified if OTA |
| RES-08 | No-show at audit with first-night policy | Audit runs | Status no-show, first night charged on card on file, room released |
| RES-09 | Two guest profiles with same email | Merge | One profile, stay history combined, consents preserved as most restrictive, audit logged |
| RES-10 | Blacklisted guest books online | Reservation created | Flag raised to front desk; booking not auto-blocked unless property setting says so |
| RES-11 | Waitlist entry for a sold-out night | A cancellation frees a room | Waitlist guest notified within one minute |
| RES-12 | Group rooming list of 20 uploaded | Imported | 20 reservations in the block, errors listed per row, master account linked |
| RES-13 | Trace due today unactioned at 18:00 | Escalation time | Escalated to manager |
| RES-14 UI | Arrivals board | Tap check-in on a pre-checked-in guest | Completes in one tap with no further data entry |

### 14.5 Folio, payments and ledger

| ID | Given | When | Then |
|---|---|---|---|
| FIN-01 | Any posting | Created | A balanced journal entry exists; sum of debits equals credits |
| FIN-02 | A posting | Edit attempted | Rejected; only a reversing posting with reason is allowed |
| FIN-03 | Room 100 plus VAT, night audit | Audit runs | Room and tax posted to every in-house folio once; re-running audit does not double post |
| FIN-04 | Folio with guest and company windows | Room to company, extras to guest | Two invoices with correct payers and VAT, sequential numbers |
| FIN-05 | Pre-auth 200 at check-in, folio reaches 250 | Incremental auth rule | Auth increased or staff alerted per configuration |
| FIN-06 | Check-out with card on file | Capture | Captured amount equals balance; folio closes at zero; invoice emailed |
| FIN-07 | Refund 30 on a 100 payment | Refund | Gateway refund 30, reversing posting, credit note referencing the invoice |
| FIN-08 | Original card expired | Refund | Alternative method with manager approval; audit shows approver |
| FIN-09 | OTA virtual card valid from check-in date | Charge attempted before | Blocked; on the date, charged automatically |
| FIN-10 | Deposit policy 50 percent at booking | Booking engine checkout | Deposit collected; deposit ledger shows liability; applied to folio at check-in |
| FIN-11 | Dojo settlement file imported | Reconciliation | Every payment matched or listed as exception with reason |
| FIN-12 | Company invoice unpaid 45 days | Debtors report | Appears in 31 to 60 ageing bucket; statement generated |
| FIN-13 | Daily GL export | Run | Totals by account equal the trial balance for the day |
| FIN-14 | Night audit pre-check finds an unposted EPOS check | Audit | Audit held with the exception listed; auditor resolves; audit completes |
| FIN-15 | Property with continuous audit enabled | Midnight passes | Business date rolls, postings run, audit pack produced, exceptions queued, no manual step |
| FIN-16 | Invoice sequence | Two invoices created concurrently | Distinct consecutive numbers, no gaps |
| FIN-17 | Cash drawer shift close | Count differs from system | Variance recorded with mandatory reason |

### 14.6 EPOS bridge

| ID | Given | When | Then |
|---|---|---|---|
| POS-01 | In-house guest with room charge allowed | Till posts 24.50 to room 101 | Posting on folio with outlet, server, items and tax; journal balanced; receipt shows room |
| POS-02 | Guest checked out 10 minutes ago | Till searches room 101 | Not offered; previous guest cannot be charged |
| POS-03 | Credit limit 100, balance 90 | Till posts 24.50 | Blocked with message; manager override allowed and logged |
| POS-04 | Breakfast included package | Till redeems breakfast for 2 | Zero charge, redemption recorded, third breakfast charged |
| POS-05 | Room charge voided on till | Void | Reversing posting on folio, approval recorded |
| POS-06 | End of day | Reconciliation | EPOS room charge total equals PMS postings from EPOS; differences listed |
| POS-07 | Event tab by event code | Till closes to event | Posts to the event master account window configured |
| POS-08 | Till offline for 10 minutes | Room charge taken offline | Queued and posted when back online; duplicate protection by idempotency key |

### 14.7 Channel manager

| ID | Given | When | Then |
|---|---|---|---|
| CH-01 | Rate change on D | Saved | ARI delta in outbox within 1 second; pushed and acknowledged within 60 seconds; health dashboard updated |
| CH-02 | Channel returns 500 | Push | Retried with backoff up to the limit, then dead-lettered and alerted; no duplicate pushes |
| CH-03 | Inbound OTA booking webhook | Received | Signature verified, raw stored, reservation created with mapped room type and plan, policies pinned, virtual card tokenised, acknowledged within SLA |
| CH-04 | Same webhook delivered twice | Received | One reservation; second acknowledged as duplicate |
| CH-05 | Inbound booking for an unmapped rate | Received | Exceptions queue entry; fix maps the rate and completes the booking |
| CH-06 | Last room sold on D | Sold | Stop sell pushed to all channels for D within 60 seconds |
| CH-07 | OTA modification changes dates | Received | Reservation updated, availability adjusted on both old and new dates, ARI deltas queued |
| CH-08 | OTA cancellation | Received | Reservation cancelled, penalty per OTA policy mapping, availability restored |
| CH-09 | Nightly full refresh | Runs | Complete 365-day ARI pushed; drift report lists any corrected differences |
| CH-10 | Channel markup plus 15 percent | Push | Pushed prices equal Zynthio price times 1.15 rounded per rule |
| CH-11 | Allotment of 3 rooms for channel X on D | 3 sold via X | X shows 0 while other channels still show remaining pool |
| CH-12 | Synthetic monitor | Hourly | Test booking and cancellation round-trips through a test channel; failure alerts |
| CH-13 | iCal channel configured | Room blocked in PMS on D | Exported iCal shows D blocked within 5 minutes; imported iCal block on E reduces availability on E |
| CH-14 | Booking.com sandbox certification suite | Run | All certification test cases pass and the report is stored |

### 14.8 Revenue engine

| ID | Given | When | Then |
|---|---|---|---|
| REV-01 | Rule: occupancy over 80 percent raises BAR 10 percent | Occupancy crosses 80 on D | Recommendation created with reason text; in autopilot, rate applied within guardrails and ARI queued |
| REV-02 | Ceiling 250 | Rule would set 270 | Capped at 250, reason notes the cap |
| REV-03 | Max daily change 15 percent | Two rules together imply 25 percent | Capped at 15 |
| REV-04 | Approve mode | Recommendation created | Nothing changes until approved; approval applies it and logs the approver |
| REV-05 | Comp set feed down | Nightly run | Rules depending on comp set skipped with a visible notice; others run |
| REV-06 | Promo capped at 50 uses | 51st booking | Code rejected with reason |
| REV-07 | Event added with high demand tag | Rules run | Event rules fire for the dates; explanation names the event |
| REV-08 | Displacement analysis for a 20-room block | Run | Shows forgone transient revenue using forecast and pace |

### 14.9 Housekeeping and linen

| ID | Given | When | Then |
|---|---|---|---|
| HK-01 | Check-out at 10:00 | Processed | Room dirty; departure clean task created with credits and priority by next arrival ETA |
| HK-02 | Stay-over on day 3 with linen change every 3 nights | Night audit | Stay-over task with linen change flag |
| HK-03 | Housekeeper offline in corridor | Completes two rooms | Status updates queue and sync on reconnect with correct timestamps |
| HK-04 | Supervisor fails inspection with photo | Saved | Room back to dirty, task reassigned, fail reason reported |
| HK-05 | Rota shows 3 housekeepers on shift | Assignment runs | Tasks distributed only to those 3, balanced by credits within 10 percent |
| HK-06 | Front desk occupied, housekeeper reports vacant | Saved | Discrepancy on room status report; alert to front desk |
| HK-07 | Linen count: issued 40, collected 38, returned 37 | Saved | Variances computed; shrinkage report updated; par shortfall alert if below par |
| HK-08 | Laundry delivery note 37 returned, invoice 40 | Reconciliation | Mismatch flagged for finance |
| HK-09 | Missing robe flagged at departure inspection | Saved | Unreturned item fee priced from catalogue, approval requested, guest notified per settings |
| HK-10 | Maintenance ticket sets room OOO for 2 days | Saved | Availability reduced for those dates, ARI pushed, tasks for the room suspended |

### 14.10 Guest journey

| ID | Given | When | Then |
|---|---|---|---|
| GJ-01 | Reservation 48 hours before arrival with schedule set to 48 hours | Scheduler runs | Online check-in message sent once on the right channel; not sent for cancelled reservations |
| GJ-02 | Guest completes online check-in with 2 names, ID, card, signature | Submitted | Reservation ready, registration cards stored for both guests, pre-auth taken, room pre-assigned if clean, fire sheet shows both names |
| GJ-03 | Non-UK guest omits passport details | Submitted | Blocked with field-level message per property rules |
| GJ-04 | Express check-out at 07:00 | Guest confirms | Balance captured, invoice emailed, room dirty, departure task created |
| GJ-05 | Guest disputes a 12 minibar line | Dispute | Ticket to front desk; line flagged; invoice held or issued per setting |
| GJ-06 | Guest requests late check-out 13:00 with fee band 10 | Request | Availability checked; fee shown; on accept, posted and housekeeping deadline moved |
| GJ-07 | Guest with marketing consent withdrawn | Campaign send | Excluded |
| GJ-08 | Unified inbox receives Booking.com message | Received | Thread attached to reservation; reply from inbox delivered via the channel |
| GJ-09 | AI assistant asked about refund | Query | Escalated to staff; no answer given on payment topics |
| GJ-10 | AI assistant asked check-in time | Query | Answers from property knowledge base in the guest's language; logged |

### 14.11 Extra charges and damages

| ID | Given | When | Then |
|---|---|---|---|
| FEE-01 | Late check-out bands 11 to 13 fee 10, 13 to 15 fee 20, grace 15 minutes | Guest departs 11:10 | No fee; departs 11:20 | 10 posted automatically |
| FEE-02 | Fee requires approval | Housekeeper flags smoking | Fee pending; manager approves; posted; guest notified with photos |
| FEE-03 | Guest consent required for post-departure charges | Charge attempted without consent record | Blocked |
| FEE-04 | Damage 180, approval threshold 100 | Created | Requires manager approval; evidence attached; posts after approval; dispute window opens |
| FEE-05 | Card surcharge item | Property in UK, consumer card | Item cannot be enabled; message explains |
| FEE-06 | Waiver of a fee | Done | Reversing posting with reason; waiver report shows user |
| FEE-07 | Parking cap 10 bays | 11th booking for D | Offered as unavailable |

### 14.12 Events

| ID | Given | When | Then |
|---|---|---|---|
| EV-01 | Ballroom combinable into A and B | A provisionally held 10:00 to 14:00 | Ballroom whole unavailable in that window plus setup buffers; B still available |
| EV-02 | Hold with release date | Date passes without decision | Reminder sent 48 hours before; auto-release or escalation per setting |
| EV-03 | Contract signed with deposit 25 percent | Signed | Payment schedule created; first pay-by-link sent; space status confirmed |
| EV-04 | Function sheet changed after lock date | Saved | New version, delta highlighted, change notice with cost impact, departments notified |
| EV-05 | 24-hour delegate package for 30 | Confirmed | 30 bedroom reservations created in the block, components posted to departments correctly |
| EV-06 | Pre-orders with 3 allergens | Submitted | Allergen matrix per table generated; kitchen production view updated |
| EV-07 | Bar tab by event code on till | Closed | Posts to master account window for beverage |
| EV-08 | Final numbers 28 against contracted 30 | Actuals applied | Attrition charge per contract; invoice reflects 30 or 28 as the rule states |
| EV-09 | Event completes | Profitability report | Revenue by component less supplier, labour from rota and food cost from EPOS; margin shown |
| EV-10 | Staffing from sheet | Published | Shifts appear in the rota module for the right roles and times |

### 14.13 Sheets and reporting

| ID | Given | When | Then |
|---|---|---|---|
| RPT-01 | Fire sheet at 23:00 | Printed and cached | Every in-house registered name present; staff on shift listed; matches in-house count |
| RPT-02 | Breakfast sheet | Generated | Pax by meal plan and time slot equal in-house guests with breakfast plus event delegates |
| RPT-03 | Manager's report | Audit | Occupancy, ADR, RevPAR and revenue reconcile to journal totals to the penny |
| RPT-04 | Pickup report | Day after 5 bookings and 1 cancellation | Net 4 rooms shown by arrival month and channel |
| RPT-05 | Scheduled report to 2 recipients at 07:00 | Scheduler | Delivered once, PDF and CSV, correct period |
| RPT-06 | Custom report saved with filter | Shared with role | Visible to that role only; pinned tile on KPI dashboard refreshes |
| RPT-07 | Deposit ledger at month end | Report | Liability equals sum of deposits held minus applied minus refunded |

### 14.14 Admin, security and compliance

| ID | Given | When | Then |
|---|---|---|---|
| ADM-01 | User with front desk role | Attempts to edit rate plan | Denied and logged |
| ADM-02 | Refund above role threshold | Attempted | Requires approval from a role with sufficient limit |
| ADM-03 | Any config change | Saved | Audit log entry with before and after, actor, time, IP |
| ADM-04 | Erasure request for a guest | Processed | Profile anonymised; ledger and invoices retained with anonymised reference; confirmation stored |
| ADM-05 | Retention 3 years for ID documents | Job runs | Documents older than 3 years purged; log kept |
| ADM-06 | Tenant A user | Requests tenant B data by ID | Not found; attempt logged |
| ADM-07 | Training mode property | Nightly reset | Sample data restored; no messages sent to real addresses |
| ADM-08 | CSV import of 500 future reservations | Imported | Row-level validation report; successful rows create reservations and adjust availability; ARI pushed once at the end |
| ADM-09 | Feature flag Yield off for property | User opens Yield | Not visible; API returns entitlement error |
| ADM-10 | Backup restore drill | Quarterly | Restores to a point in time within the RPO; checklist signed |

### 14.15 Non-functional

| ID | Requirement | Acceptance |
|---|---|---|
| NFR-01 | Availability 99.9 percent monthly for booking engine, PMS and channel sync | Measured by external monitor |
| NFR-02 | ARI push to acknowledgement under 60 seconds at p95 | Health dashboard |
| NFR-03 | Inbound booking to reservation created under 5 seconds at p95 | Logs |
| NFR-04 | Search under 300 ms at p95 for the reference dataset | Load test |
| NFR-05 | Zero stored PANs | Code scan and PCI SAQ |
| NFR-06 | All user-facing pages WCAG 2.2 AA | Automated scan plus manual check on booking engine and guest app |
| NFR-07 | Housekeeping app usable offline for 8 hours | Field test |
| NFR-08 | Data residency UK | Infrastructure review |
| NFR-09 | Recovery point 15 minutes, recovery time 4 hours | Restore drill |
| NFR-10 | Every automated price, task assignment and stop-sell has a stored human-readable reason | Code review and sample audit |
