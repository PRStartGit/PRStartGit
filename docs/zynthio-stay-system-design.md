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
| Channel account | channel, property id on channel, credentials via aggregator, status | One per channel per property |
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
| Comp set | list of competitor property ids on the rate feed, weight | Market position and recommendations |
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
