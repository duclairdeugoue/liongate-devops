# Paligo Platform - Domain Architecture Reviews

**Purpose:** Structured architecture review, one section per domain, produced using the standard review prompt below. Intended for engineering review and sign-off.
**Inputs:** current architecture diagram, Platform Roadmap v1.0 (June 2026), Product Owner discovery interview transcript.
**Scope:** service boundaries, data ownership, integration patterns. No technology stack recommendations.

**How this document is organized:**

- **Part 1 - Current Architecture** reviews the six domains that already exist on the architecture diagram today (in some form), with their current state, issues, and how they should evolve.
- **Part 2 - Future Architecture** covers the six domains that the platform needs but don't have a service home yet - described in detail in the roadmap and the discovery interview, but not yet reflected in any diagram or codebase boundary. These are what needs to be _added_, not fixed.

---

## Review Prompt Used

```
You are acting as a senior software architect reviewing one domain of the Paligo
marketplace platform. For the domain named: "{{DOMAIN_NAME}}", produce a
structured write-up with exactly these four sections:

1. CURRENT STATE
   - What exists today for this domain (services, data stores, integrations).
   - Which other domains it currently talks to, and how (sync API, event,
     direct DB access, none).
   - Cite the specific roadmap item(s) or interview statement(s) this is
     based on, if any.

2. ISSUES & RISKS
   - Coupling risks: does this domain depend on something outside its
     natural boundary (e.g. a customer-facing path depending on a
     back-office system)?
   - Scalability risks: what happens to this domain under 10x current load?
   - Clarity risks: could a new engineer understand this domain's
     responsibility from its name and interfaces alone?
   - Extensibility risks: what upcoming roadmap item would be hard to add
     given the current design?

3. RECOMMENDATION
   - Target service boundary: what should this domain own, and NOT own.
   - Integration pattern: synchronous API (when a real-time answer is
     required) vs. event (when eventual consistency is acceptable).
   - Data ownership: which database/store is the source of truth, and what
     is a read-only projection elsewhere.

4. CONSTRAINTS
   - Business rules that are non-negotiable (pull these directly from the
     Product Owner discovery interview - do not infer or soften them).
   - Compliance/legal constraints (GDPR, PSD2, VAT, multi-country).
   - Organizational constraints (team ownership, on-call boundary - a
     service should be ownable by one team).

Do not propose a specific technology stack unless asked. Focus on service
boundaries, data ownership, and integration patterns only. Flag explicitly
if the current documentation (Wiki/interview) does not contain enough
information to answer a section confidently, rather than guessing.
```

---

## Domain Index

**Part 1 - Current Architecture (exists today, in some form)**
1.1 [Identity & Access](#11-identity--access)
1.2 [Storefront / API Gateway](#12-storefront--api-gateway)
1.3 [Product Catalog & Master Data](#13-product-catalog--master-data)
1.4 [Merchant & Listing Service](#14-merchant--listing-service)
1.5 [Pricing Engine](#15-pricing-engine)
1.6 [Order & Checkout](#16-order--checkout)

**Part 2 - Future Architecture (to be added, no service home yet)**
2.1 [Payment & Escrow](#21-payment--escrow)
2.2 [Logistics Connector (Logistix)](#22-logistics-connector-logistix)
2.3 [Ratings & Claims](#23-ratings--claims)
2.4 [Paligo AI OS](#24-paligo-ai-os)
2.5 [Market Intelligence / Data & Analytics Platform](#25-market-intelligence--data--analytics-platform)
2.6 [Multi-Country Expansion](#26-multi-country-expansion-cross-cutting)

---

# Part 1 - Current Architecture

These six domains already exist on the current architecture diagram, in some form - either as a clean, dedicated service or as logic folded into another system (most often Odoo ERP). The reviews below focus on how each should be **fixed or clarified**, not built from scratch.

### 1.1 Identity & Access

#### CURRENT STATE

Keycloak is the identity provider, with its own database. Two components call it directly today: the Storefront (for customer login/token) and Directus CMS (for auth/token on content operations). There is no evidence in the diagram of the Middleware validating or brokering tokens - both the Storefront and Directus appear to hold their own trust relationship with Keycloak.
No roadmap item explicitly addresses identity architecture; it's implicit in Phase 0 ("VPN access," "shared password manager") for internal team access, not customer/merchant identity. The interview confirms one identity-related business rule: a merchant workspace is single-login per entity - a holding company or CEO wanting oversight across multiple merchant accounts is explicitly called "a theoretical exception case" that the business does not want to support ("it's better to have to separate... this is how we set it up now, and it makes no sense [to combine them]").

#### ISSUES & RISKS

- **Coupling:** Two independent components (Storefront, Directus) each hold a direct trust relationship with Keycloak. Any change to token format, session policy, or MFA has to be implemented and tested in two places.
- **Scalability:** Not a major concern for Keycloak itself at 10x load (identity providers scale well), but the _lack of a single validation point_ means 10x traffic means 10x the places a token-handling bug can surface.
- **Clarity:** A new engineer looking at the diagram cannot tell why some services call Keycloak directly and others go through the Middleware - there's no stated rule.
- **Extensibility:** Roadmap items requiring distinct permission sets (merchant dashboard access, Type A/B/C fulfillment permissions, future Paligo merchant app in 2027, Logistix as a separate product with its own login) will each need a clear realm/client boundary. Without one now, each new consumer will likely repeat the "call Keycloak directly" pattern.

#### RECOMMENDATION

- **Owns:** authentication, session/token issuance, merchant/customer/admin realm separation, password and MFA policy.
- **Does not own:** authorization logic specific to a business domain (e.g. "can this merchant edit this product" belongs to the Merchant & Listing Service, using the identity/role claims Keycloak provides - not Keycloak enforcing business rules itself).
- **Integration pattern:** synchronous, but only at one point - the API Gateway validates and terminates the token exchange with Keycloak; internal services validate signed tokens locally rather than each calling Keycloak per request.
- **Data ownership:** Keycloak's own DB is the source of truth for identity; no other service should store credentials or duplicate session state.

#### CONSTRAINTS

- **Business rule (non-negotiable, from interview):** one login = one merchant entity/workspace. Multi-entity oversight (e.g. a franchise owner viewing several merchant accounts) is explicitly _not_ a requirement for this system and should not be designed in as a shortcut.
- **Compliance:** GDPR applies to all stored personal data (customer and merchant contact details) - data residency and right-to-erasure need to be supported per the eventual multi-country rollout.
- **Organizational:** identity should be owned by one team (likely platform/infra), since every other domain depends on it - it is a shared-infrastructure service, not a feature team's responsibility.

---

### 1.2 Storefront / API Gateway

#### CURRENT STATE

The Storefront (frontend) is a single component today. It calls the Middleware for most functionality (API Call), but calls Keycloak and Pricing directly for login/token and price display respectively. The Middleware in turn talks to Merchant Module, Directus CMS, and Odoo ERP.
Roadmap: Phase 1's Customer Module items (category-first navigation, filter/sort, offer comparison, all-inclusive price display) all live behind this Storefront. No roadmap item currently addresses gateway consolidation - this is a gap identified in this review, not something already planned.

#### ISSUES & RISKS

- **Coupling:** the Storefront currently has three distinct backend relationships (Middleware, Keycloak, Pricing) instead of one. This is the most direct violation of "clear and easy to understand" in the current architecture.
- **Scalability:** direct Storefront→Pricing calls mean pricing traffic scales 1:1 with storefront traffic, with no gateway-level caching, rate limiting, or circuit breaking available to protect Pricing under load spikes (e.g. seasonal demand, explicitly flagged as a Phase 2 concern: "load and stress testing ahead of seasonal demand spikes").
- **Clarity:** a new frontend engineer has to learn two different integration patterns for what should be one consistent client-to-backend contract.
- **Extensibility:** every new backend domain (Logistics tracking, Ratings, AI assistant) will need a decision - "does this go through the Middleware or get called directly?" Without a stated rule, direct-call sprawl is the likely default, repeating this problem at a larger scale.

#### RECOMMENDATION

- **Owns:** a single, consistent entry point (API Gateway / Backend-for-Frontend) for all Storefront traffic - routing, auth enforcement, rate limiting, response aggregation where needed (e.g. combining product + price + rating data for a listing page).
- **Does not own:** business logic belonging to any specific domain (pricing calculation, order splitting, etc.) - the gateway routes and aggregates, it does not compute.
- **Integration pattern:** synchronous, gateway-brokered for all Storefront calls. No domain service should be called directly by the frontend.
- **Data ownership:** none - this is a stateless routing/aggregation layer, not a system of record.

#### CONSTRAINTS

- **Business rule:** the customer journey is explicitly category-first (product group before anything else - "the customer journey starts on the level of the product group... they know which type of product and means of product group they need"), and delivery address must be captured _before_ pricing is shown ("delivery address input as first step (before pricing is shown)" - Phase 1 roadmap, P0). The gateway/BFF should enforce this sequencing at the API contract level so it can't be bypassed by a future frontend redesign.
- **Compliance:** none specific to this layer beyond standard transport security (delegated to Identity domain).
- **Organizational:** should be owned by whichever team owns the Storefront/frontend, since its purpose is entirely in service of frontend needs - it is a "backend for frontend," not shared platform infrastructure.

---

### 1.3 Product Catalog & Master Data

#### CURRENT STATE

Directus CMS holds "Produktdaten / Inhalte" (product data / content) with its own DB, reached via the Middleware and authenticated through Keycloak. There is no separately visible "product master data" or barcode/taxonomy service - this appears to be folded into Directus and/or the Merchant Module today.
Roadmap, Phase 1 (3.1 Merchant Module): "Product group taxonomy (heating, garden, construction, firewood)" [P0, S1], "Product filter criteria per category (weight, volume, diameter, etc.)" [P0, S2], "Paligo barcode / SKU system (generic + merchant-internal SKUs)" [P1, S3]. Interview: the barcode system is described in detail as a _generic, standardized_ product identifier ("we have a Paligo barcode... for us it's a standard Paligo product... one product can be characterized by 3 to maximum 4 different criteria") separate from each merchant's internal SKU, and new/unrecognized products go through a moderation step before publishing ("it runs through a permission process, and there is someone who verifies this product on behalf of Paligo").

#### ISSUES & RISKS

- **Coupling:** if generic product taxonomy/barcode data and merchant-specific listing data (price, quantity) live in the same store or service, every merchant listing update risks touching shared master data, and every taxonomy change risks breaking merchant listings.
- **Scalability:** taxonomy/master data is read-heavy and changes rarely; merchant listings (price, quantity) change frequently ("how frequently do product prices change" is an explicit open question in your own discovery script). Mixing a low-write, high-read dataset with a high-write dataset in one service makes both harder to scale and cache correctly.
- **Clarity:** "Directus CMS" as a name suggests a content/marketing tool, not a system of record for sellable product attributes and barcodes - the current naming and boundary invite confusion about what belongs where.
- **Extensibility:** the roadmap's Phase 2 item "Product multi-use group assignment (e.g., sand for construction vs. winter roads)" requires one physical product to belong to multiple taxonomy groups simultaneously, each with different filter criteria - this is explicitly confirmed in the interview as a real, recurring case ("it oftentimes happens that the usage of a commodity is diverse... you have one product, but you put it into 3 different groups"). A single-parent taxonomy model would need rework to support this; a many-to-many model designed now avoids that.

#### RECOMMENDATION

- **Owns:** the generic product taxonomy, filter-criteria schema per product group, the Paligo barcode system, and the product moderation/approval workflow for new generic products.
- **Does not own:** merchant-specific price, quantity, or shipping-price tables (see Merchant & Listing Service) or marketing content/imagery (Directus can remain purely for content in this narrower role, or be absorbed if content needs are simple enough).
- **Integration pattern:** synchronous read API for the Storefront/Gateway (taxonomy and filters must be available at request time for search/filter UI); publishes `product.taxonomy.updated` and `product.approved` events for consumers like the Merchant & Listing Service and AI OS (for assisted-listing matching).
- **Data ownership:** this service is the source of truth for taxonomy and barcodes; the Storefront's search/filter index is a read-only projection, not a second source of truth.

#### CONSTRAINTS

- **Business rule (non-negotiable):** the barcode is universal across merchants and is _the_ mechanism for cross-merchant price comparison ("if you scan the barcode that belongs to our SKU... you can compare alike with like"); the SKU is merchant-private and must never be conflated with the barcode. New products without a recognized barcode must go through human/AI-assisted moderation before appearing live - this cannot be a self-service publish for unrecognized products.
- **Compliance:** EU e-commerce law requires a technical specification sheet with safety information for certain product categories (explicitly noted in the interview: "it's mandatory in e-commerce in Europe to have a technical specification sheet with security warnings") - this is master-data content, not merchant-listing content, and should be enforced at the taxonomy/master-data level so it can't be omitted per merchant.
- **Organizational:** this is a natural single-team ownership boundary (a "catalog" or "product platform" team), since taxonomy changes have wide blast radius and need a single point of accountability.

---

### 1.4 Merchant & Listing Service

#### CURRENT STATE

"Merchant Module" on the diagram, with its own DB, holding "Bestellungen" (orders/inventory) and reached by Merchants directly as well as through the Middleware. The current implementation reportedly still supports multiple warehouses per merchant, each requiring separate product registration - explicitly flagged in the interview as the current system's known flaw ("in our backend, we can register multiple warehouses. And for every warehouse, they can register individually... this leads to a nightmare of complexity").
Roadmap, Phase 1 (3.1): "Single product catalogue per merchant (replacing multi-warehouse)" [P0, S1–S2] - this fix is already planned and is the top priority item in the merchant domain.

#### ISSUES & RISKS

- **Coupling:** as currently built (per the interview), one merchant's product listing is duplicated per warehouse - this is a data-modeling issue internal to the domain, not a cross-service coupling risk, but it inflates the domain's complexity and will make any future integration (ERP connector, bulk import) harder than necessary.
- **Scalability:** a large merchant with many warehouses under the current model creates N× the listing records for the same product - this scales _badly_ with merchant size, which is the opposite of what's needed as larger merchants (500–1000+ SKUs, per the interview) onboard.
- **Clarity:** the current data model requires a new engineer to understand _why_ warehouse-level duplication exists before they can safely change anything - a well-known sign of accumulated complexity from a workaround, not a designed feature.
- **Extensibility:** the roadmap's Phase 1 item "API connector for ERP integration (SAP, Odoo, custom ERP)" [P2, S5–S6] and "Bulk product import via standardised CSV/Excel template" [P1, S4] will both be significantly harder to build against a multi-warehouse-duplicated model than against the single-catalogue model already planned - this reinforces that the catalogue fix should land _before_ the import/API work, not in parallel.

#### RECOMMENDATION

- **Owns:** one product catalogue per merchant (quantity + ex-works price, referencing the generic Product Master), the postcode-based shipping price table attached to that catalogue, and the merchant's dashboard data (orders, SLA, ratings overview).
- **Does not own:** which physical warehouse fulfills an order - explicitly out of scope per the interview ("it doesn't even matter how many warehouses they have... it's not our problem to decide from where they must ship it... this is their business").
- **Integration pattern:** synchronous for merchant dashboard reads/writes; publishes `listing.updated`, `listing.price.changed`, `listing.quantity.changed` events for Pricing, Catalog/Search, and Analytics to consume.
- **Data ownership:** source of truth for price, quantity, and shipping-price-per-postcode-zone per merchant per generic product. Pricing and Storefront search hold read-only projections.

#### CONSTRAINTS

- **Business rule (non-negotiable):** "one product catalogue, together with this postcode-based transport price table" per merchant - multi-warehouse detail is explicitly irrelevant to Paligo and must not resurface as a modeling concept. A merchant may set a flat national shipping price or granular postcode-level pricing (down to the 5-digit postcode) at their discretion - the system must support both without requiring the merchant to declare warehouses.
- **Compliance:** merchant KYC/onboarding (already a Phase 1 P0 item) needs to satisfy relevant merchant-of-record legal requirements, since Paligo is the legal buyer/seller, not a pure commission marketplace.
- **Organizational:** this is a high-traffic, merchant-facing domain suited to a dedicated "merchant platform" team, separate from the customer-facing storefront team, since its release cadence and stakeholders (merchant support, KYC/compliance) differ.

---

### 1.5 Pricing Engine

#### CURRENT STATE

"Pricing" on the diagram, with its own DB, fed by Odoo ERP and called directly by the Storefront (bypassing the Middleware).
Roadmap, Phase 1 (3.3 Pricing Engine): "Static margin configuration per product group" [P0, S2], "Sales price calculation: ex-works + calculative shipping + margin + VAT" [P0, S2], "Foundation for dynamic margin engine" [P1, S5–S6]. Phase 3: "Dynamic pricing engine (real-time margin adjustment per market)" [P0, Q1 2027]. Interview: pricing is explicitly detached from actual logistics cost - "the calculated cost, bottom up, from the product to the sales price... margin and the VAT, very simple. The other is the actual cost, we do not know his actual cost."

#### ISSUES & RISKS

- **Coupling:** direct dependency on Odoo ERP puts a customer-facing, latency-sensitive read path behind a back-office transactional system not designed for storefront-scale traffic. This is the single highest architectural risk currently visible on the diagram.
- **Scalability:** under 10x storefront load, Pricing calls to Odoo would multiply 10x - Odoo is not built to serve as a real-time read API for product-listing-scale traffic (every filtered search result needs a price). This risks both storefront slowdowns and ERP contention with actual accounting/order-processing work.
- **Clarity:** the direct Storefront→Pricing call (bypassing the Middleware) means pricing has a different integration contract than every other Storefront-facing feature, which a new engineer would not expect without being told.
- **Extensibility:** the Phase 3 goal of real-time, per-minute dynamic margin adjustment is very difficult to achieve if Pricing's data source is ERP-driven - ERP systems are not built for sub-second, high-frequency rule evaluation. Building dynamic pricing on the current foundation would likely require a rebuild rather than an extension.

#### RECOMMENDATION

- **Owns:** the price calculation itself (ex-works + calculative shipping + margin + VAT), the margin rules engine (static today, dynamic later), and price snapshots per order line at time of purchase.
- **Does not own:** actual carrier/logistics cost (explicitly irrelevant to this domain per the interview), and the underlying ex-works price/listing data (owned by Merchant & Listing Service - Pricing subscribes to it, doesn't own it).
- **Integration pattern:** consumes `listing.price.changed` and `listing.quantity.changed` events asynchronously from Merchant & Listing to maintain its own fast read store; exposes a synchronous API to the Gateway for real-time price display, but this API should never depend on a live Odoo call.
- **Data ownership:** Pricing owns its own price computation store; Odoo remains the system of record for the _legal_ transaction (purchase order to merchant, sales order to customer) but is downstream of Pricing, not upstream.

#### CONSTRAINTS

- **Business rule (non-negotiable):** price shown to the customer is always all-inclusive (product + delivery, no breakdown shown) - "he sees 200 + 50 + margin + VAT... he buys for 320... we do not show him the single partial payments." Once an order is placed, the price is locked regardless of any later change in actual logistics cost - "it will never, never, never [change]... we don't care even if... the logistic[s] price changes." Margin percentages today are static per product group (e.g., "5% for pellets, 8–10% for [others], 15% for garden") but must become dynamically adjustable per country/product group/time in future - the rules engine should be designed for this from the start even while only static rules are active.
- **Compliance:** VAT must be correctly calculated and displayed per the customer's country as multi-country rollout proceeds - VAT rates and rules are country-specific and need to be modeled as configuration, not hardcoded.
- **Organizational:** given its central role and sensitivity (this is the domain most tied to competitive strategy - Phase 3's per-minute dynamic pricing is explicitly a strategic weapon: "maybe sometimes we send even product on a minus margin because I want to kill a competitor"), this domain likely warrants dedicated ownership rather than being a shared responsibility, with tightly controlled access to margin-rule changes.

---

### 1.6 Order & Checkout

#### CURRENT STATE

Not a separately visible box on the diagram - order handling ("Aufträge") lives inside Odoo ERP, reached via the Middleware.
Roadmap, Phase 1 (3.2 Customer Module): "Multi-supplier basket (multiple order lines, single checkout)" [P1, S3], "Order confirmation and per-order-line status tracking" [P1, S3–S4]. Interview, extensively: a single checkout can span multiple merchants ("at the end of the day there's 3 different products in the basket... but it comes from different suppliers"), and each resulting order line is tracked and fulfilled completely independently, including independent shipping schedules ("position number one will be shipped by Tuesday... position number 2 will be shipped on Friday").

#### ISSUES & RISKS

- **Coupling:** if order-line-splitting logic lives inside Odoo (a back-office/accounting system), exposing real-time per-line status to the Storefront requires either polling Odoo directly (a scalability risk, same class as Pricing) or building a workaround layer - neither is a clean long-term position.
- **Scalability:** one customer checkout can legitimately fan out into many independent merchant order lines; if this fan-out and subsequent status tracking is not modeled as an explicit workflow/state machine, it will be difficult to query efficiently ("show me all my order lines and their statuses") at any real customer volume.
- **Clarity:** "Order" as a customer-facing concept (one checkout, one payment) and "order line" as the actual unit of fulfillment (one per merchant) are two different things that need to be clearly named and separated in the domain model - conflating them is a likely source of confusion for both engineers and, eventually, customer support.
- **Extensibility:** future fulfillment-type-specific behavior (Type A/B/C merchants, per Logistics domain) needs to hang off the order line, not the order - a model that doesn't cleanly separate order from order-line will make this harder to extend correctly.

#### RECOMMENDATION

- **Owns:** the cart-to-checkout flow, splitting a multi-merchant basket into independent order lines, and the per-line status state machine (placed → confirmed → ready-to-ship → shipped → delivered → payout-eligible).
- **Does not own:** payment capture (see Payment & Escrow), price computation (already finalized/snapshotted by Pricing at time of order), or logistics/tracking detail (see Logistics Connector) - Order tracks _state_, not carrier detail.
- **Integration pattern:** synchronous for checkout itself (customer needs immediate confirmation); publishes `order.placed` and `order.line.split` events for Payment, Logistics, Merchant notification, and Analytics to consume asynchronously.
- **Data ownership:** Order service is the source of truth for order/order-line state; Odoo consumes `order.line.*` events to create the corresponding accounting purchase/sales orders, but does not originate order state.

#### CONSTRAINTS

- **Business rule (non-negotiable):** one payment, one checkout - but N independent order lines, each with its own merchant, fulfillment path, and tracking, and the customer sees all of this within a single order view, not N separate purchase flows. A product may be listed by the same merchant more than once in a basket and combine into one line if fulfilled together - this and other basket-consolidation rules should be confirmed explicitly with the Product Owner before implementation (not fully specified in source material).
- **Compliance:** order records need to satisfy invoicing/accounting retention requirements per country as multi-country rollout proceeds.
- **Organizational:** natural fit for the same team owning checkout/frontend commerce flow, working closely with the Payment and Merchant domains given the tight sequencing between checkout, payment capture, and order-line creation.

---

---

# Part 2 - Future Architecture

These six domains do **not** yet exist as services anywhere in the current architecture. They are described in real detail in the Product Owner discovery interview and/or the roadmap, but today they either live implicitly inside another system (e.g. payment inside Odoo) or don't exist at all (e.g. AI OS, Market Intelligence). The reviews below focus on **what needs to be built**, using the business rules already captured in the interview as the specification.

### 2.1 Payment & Escrow

#### CURRENT STATE

Not visible as a distinct component on the diagram - likely implicit within Odoo ERP today.
Roadmap, Phase 1 (3.2): "Payment integration (Paypal / Stripe / Mollie, card + bank transfer)" [P0, S3]. Phase 2: "Merchant payout management (net D+14 after customer confirmation)" [P0, Q4 2026]. Interview: Paligo is legally the buyer and seller of record - it purchases from the merchant and pays the merchant only after customer delivery confirmation ("we pay the invoice only 2 weeks after the customer has been satisfied... then this payment is released").

#### ISSUES & RISKS

- **Coupling:** if payment capture and merchant payout logic live inside Odoo (an accounting system), the specific business rule of _conditional, delayed, per-order-line payout_ is harder to express and audit than in a purpose-built service, and harder to reconcile with the Order domain's per-line status.
- **Scalability:** not a major risk at current volume, but payout timing logic (D+14, or automatic after 2 weeks of customer silence - "if after 2 weeks the customer did not raise a claim, we assume the product has been delivered") is exactly the kind of time-triggered business rule that becomes error-prone if implemented as manual/batch ERP processes rather than an explicit scheduled workflow.
- **Clarity:** "payment" as a concept doesn't currently have a clearly named home in the architecture at all - this is a gap, not just a coupling issue.
- **Extensibility:** multi-currency support (implied by multi-country expansion - different currencies per country, confirmed in interview: "we counted the supplier and the customer, they are always within the same country... different currencies [otherwise]") needs to be designed into this domain from the start.

#### RECOMMENDATION

- **Owns:** payment capture from the customer at checkout, holding funds, and releasing payout to each merchant per order line based on delivery-confirmation events (or the 2-week auto-confirmation rule).
- **Does not own:** the legal purchase/sales order bookkeeping (that remains Odoo's role as the accounting system of record) - Payment triggers and informs Odoo, rather than being subsumed by it.
- **Integration pattern:** synchronous for the capture step at checkout (needs immediate success/failure response); event-driven for payout release, triggered by `order.line.delivered` or a time-based `order.line.auto_confirmed` event.
- **Data ownership:** source of truth for payment/payout ledger entries; Odoo receives payout events for accounting/invoicing purposes but does not originate payout decisions.

#### CONSTRAINTS

- **Business rule (non-negotiable):** merchant payout is always conditional on delivery confirmation (explicit or 2-week implicit), and is per order line, not per order - a merchant cannot be paid out for a line that hasn't been confirmed delivered, even if other lines in the same customer order have been.
- **Compliance:** this is very likely a regulated activity (holding customer funds before releasing to a third party is a form of payment intermediation/safeguarding under frameworks like PSD2 in the EU) - **flagging explicitly: the source material does not specify Paligo's payment license/PSP arrangement, and this should be confirmed with legal/compliance before finalizing the service design**, since it affects whether Paligo can hold funds directly or must route through a licensed payment processor's escrow feature.
- **Organizational:** given the compliance sensitivity, this domain should have clear, restricted ownership (small team, audited changes) rather than broad access.

---

### 2.2 Logistics Connector (Logistix)

#### CURRENT STATE

Not present on the current architecture diagram. Today, per the interview, this logic exists as Python-based connector code inside 3NRG's Odoo instance, pushing order/delivery data to various European carrier networks and retrieving tracking numbers, using formats including the "Fortras 100" standard.
Roadmap, Phase 1 (3.4 Logistics Connector): "Extract and refactor existing Python expedition connectors" [P0, S2–S3], "Standalone logistics connector service" [P0, S3–S4], support for Merchant Type A (self-delivery) and Type B (external logistics) [P1/P0, S4], "Centralised tracking status normalisation" [P0, S4]. Phase 3: "Logistix standalone product (logistics connector as SaaS)" [P1, Q1 2027]. This is the most extensively described domain in the interview - carrier selection is the merchant's decision, not the platform's; the connector only pushes order data and retrieves tracking, it does not compare prices or select carriers ("the selection, which [carrier] partner is the cheapest and best, is not the job of this tool").

#### ISSUES & RISKS

- **Coupling:** currently entangled with 3NRG's Odoo instance - the roadmap already identifies this as a problem to fix, but until extracted, every other merchant's logistics integration is being designed against code that was written for one specific company's operational needs, not a general-purpose connector.
- **Scalability:** the described carrier landscape is "dozens of different networks, partners... APIs and connectors" across Europe - a connector not designed for pluggable, per-carrier adapters from day one will need significant rework as more carriers and countries are added.
- **Clarity:** because this logic currently lives inside another company's ERP system, a new engineer has no clear, dedicated place to look for "how does Paligo talk to carriers" - this is presently the least discoverable domain in the architecture.
- **Extensibility:** the Phase 3 goal of spinning this into an independent SaaS product ("Logistix," with its own domain, plug-ins for Shopware/JTL) will be far more expensive if the extraction happens late, after Paligo-specific assumptions have accumulated in the code. Designing it as a standalone bounded service now - even while only serving Paligo - directly avoids a second migration later.

#### RECOMMENDATION

- **Owns:** pushing order/delivery data to the carrier the merchant has chosen, retrieving and normalizing tracking status across all integrated carriers into one standard status model, and merchant-level configuration of carrier credentials/relationships.
- **Does not own:** carrier selection or price comparison (explicitly the merchant's job today; a possible _separate_ future product per the interview - "this I see rather in the separate module of logistics... a different service with a different membership") and does not own the calculative shipping price shown to customers (that's Merchant & Listing Service / Pricing).
- **Integration pattern:** synchronous push on order-line creation (send delivery data to the chosen carrier); asynchronous/webhook or polling for tracking updates, normalized and published as `shipment.status.updated` events for the Order domain and customer notifications to consume.
- **Data ownership:** source of truth for raw carrier tracking data and its normalized status; Order domain holds a read-only projection of "current shipment status" per order line.

#### CONSTRAINTS

- **Business rule (non-negotiable):** the connector has exactly two jobs - push order data, retrieve tracking - and must not attempt carrier selection or price comparison; that decision belongs entirely to the merchant. Type A merchants (self-delivery) have no tracking today by design, not by omission - the system must support "no tracking available" as a valid, expected state, not an error. Payout eligibility depends on a delivery confirmation signal from this domain (or the 2-week silent-customer rule) - this is a hard dependency the Payment domain has on this one.
- **Compliance:** none specific beyond standard data handling for delivery addresses (personal data, GDPR-relevant) shared with third-party carriers - data processing agreements with each carrier integration should be tracked.
- **Organizational:** given the explicit roadmap ambition to spin this into an independent commercial product (Logistix) sold to other businesses beyond Paligo, this strongly warrants a dedicated team and a clean, product-quality API contract from the outset - building it as "just another internal service" risks the kind of rework the extraction from 3NRG's Odoo is already trying to escape.

---

### 2.3 Ratings & Claims

#### CURRENT STATE

Not present on the current architecture diagram or explicitly itemized as a distinct roadmap component, though related items exist: Phase 1 (3.2) "Dual rating system: product rating + service/fulfillment rating" [P1, S4] and "Customer claim/dispute flow with file upload support" [P1, S4–S5]; Phase 2: "Claim resolution workflow (AI-assisted triage, human judge)" [P1, Q4 2026]. The interview describes this domain in significant detail: ratings are split into two independent scores (product quality vs. fulfillment/service quality) because the same product sold by different merchants can have very different fulfillment outcomes; claims go through AI-assisted triage before reaching a human moderator ("the judge") who has visibility into both customer and merchant-provided evidence; merchants can flag a rating as unfair but cannot delete it.

#### ISSUES & RISKS

- **Coupling:** none observable yet, since the domain doesn't exist in the current architecture - the risk is entirely in _how_ it gets built once started. If claims/ratings logic is added directly into the Order or Merchant domain (the likely default, since that's the existing pattern on this diagram), it will inherit the same "everything talks to everything" coupling already present elsewhere.
- **Scalability:** ratings triggered per order-line completion, at 10x order volume, need to be an asynchronous, event-triggered process (not a synchronous step blocking order completion) to avoid becoming a bottleneck.
- **Clarity:** the interview's explicit design goal - customer and merchant never communicate directly, all feedback and disputes are platform-mediated - is a trust-and-safety rule that needs to be visible as an architectural boundary (no direct customer↔merchant messaging channel exists anywhere in the system), not just a UI convention that could be quietly bypassed by a future feature.
- **Extensibility:** the AI-assisted triage step (Phase 2) depends on this domain publishing structured claim data that an AI service can consume - if claims data isn't modeled with that in mind from the start, retrofitting AI triage later is harder.

#### RECOMMENDATION

- **Owns:** the dual rating model (product + service, independently), the claim state machine (opened → AI-triaged → evidence collected → resolved/appealed), and the platform-mediation rule (all customer↔merchant communication about an order routes through this domain, never peer-to-peer).
- **Does not own:** payout decisions (Payment domain owns actual payout release, though it should consume `claim.resolved` events from this domain as an input) or delivery-status data (consumes `shipment.status.updated` from Logistics as context for claims, doesn't own it).
- **Integration pattern:** event-triggered - subscribes to `order.line.delivered` to prompt ratings, and to `shipment.status.updated` for claims context; exposes a synchronous API to the Gateway for customers/merchants to submit ratings, claims, and evidence.
- **Data ownership:** source of truth for ratings, reviews, and claim records; Merchant & Listing Service's displayed "average rating" is a read-only projection, refreshed via events, not computed live from this domain on every storefront request.

#### CONSTRAINTS

- **Business rule (non-negotiable):** ratings are always dual (product and service, scored separately) - never a single combined score. Customers and merchants never communicate directly; all feedback flows through the platform. Merchants may flag/dispute a rating as false but cannot delete it; Paligo staff have final adjudication authority.
- **Compliance:** claim evidence (photos, descriptions) constitutes personal/commercial data subject to GDPR retention rules; dispute resolution records may need to be retained for consumer-protection compliance purposes per country.
- **Organizational:** this domain sits at the intersection of engineering and customer operations (the human "judge" role is an operational, not purely technical, function) - ownership should include a clear interface for the customer service/operations team, not just engineering.

---

### 2.4 Paligo AI OS

#### CURRENT STATE

Not present on the current architecture diagram. Roadmap, Phase 0: "Initialise Paligo AI OS infrastructure (private LLM setup)" [AI Lead, Week 2–3], "Feed Paligo Wiki into AI knowledge base" [AI Lead, Week 3–4]. Phase 2: "AI-assisted product listing (auto-fill from PDF/image upload)" [P0, Q4 2026], "AI conversational search assistant (customer-facing)" [P0, Q4 2026]. Phase 4: "Full Paligo AI OS v2 (self-updating knowledge base)... agents for onboarding, customer service, pricing." The interview describes an extensive, multi-role AI vision: assisted merchant listing (extracting structured product data from PDFs/images/voice), a customer-facing conversational assistant that narrows down needs and personalizes the storefront per logged-in customer, AI-assisted claims triage, and an internal company knowledge base feeding both developer onboarding and customer-service agents. A specific concern is raised about data confidentiality - wanting a private/self-hosted setup for sensitive business data rather than sending everything to a third-party commercial LLM provider.

#### ISSUES & RISKS

- **Coupling:** if built as a single undifferentiated "AI service" that all use cases route through, low-sensitivity tasks (e.g. extracting text from a product PDF) and high-sensitivity tasks (e.g. anything touching margin strategy or the internal wiki) would share the same infrastructure and access controls - either over-restricting the former or under-protecting the latter.
- **Scalability:** the customer-facing conversational assistant is potentially the highest-traffic AI use case (every storefront visitor could interact with it) and has very different latency/cost requirements than the merchant-onboarding assistant (used occasionally, can tolerate longer processing time) - treating them as one workload risks both over-provisioning and poor customer experience.
- **Clarity:** "Paligo AI OS" as currently described spans at least four distinct use cases (listing assistance, customer search/personalization, claims triage, internal knowledge base) - without an explicit gateway/tiering structure, a new engineer has no way to know which use case's requirements govern the whole system's design.
- **Extensibility:** Phase 4's "agents for onboarding, customer service, pricing" implies several more agents beyond what's planned for Phase 0–2. Building the first agents without a shared, reusable retrieval/knowledge layer means each new agent likely re-ingests the wiki independently, multiplying maintenance cost.

#### RECOMMENDATION

- **Owns:** an AI gateway with tiered access (commercial LLM API access for low-sensitivity tasks; private/self-hosted model access for high-sensitivity tasks), and a shared retrieval layer over the company knowledge base that all agents query rather than each ingesting independently.
- **Does not own:** the business logic each agent assists with (e.g. the listing-assistance agent doesn't own product taxonomy - it calls the Product Catalog domain; the claims-triage agent doesn't own claim resolution - it assists the Ratings & Claims domain).
- **Integration pattern:** synchronous for real-time use cases (customer-facing conversational assistant, claims triage during an active session); can be asynchronous/batch for merchant-listing assistance (extract-then-review is not latency sensitive). Knowledge-base updates should be event-driven (Wiki changes trigger re-indexing) rather than static/manual imports, per the explicit "living document... dynamically documented... constantly learning" goal stated in the interview.
- **Data ownership:** the AI Gateway does not own business data - it's a consumer of Product Catalog, Merchant, Order, and Ratings data via their published events/APIs, plus the company knowledge base as its own retrieval index.

#### CONSTRAINTS

- **Business rule (non-negotiable):** confidential business data (pricing strategy, merchant financials, internal wiki content) must not be sent to third-party commercial LLM providers - this requires the tiered-access design above, not a single shared integration. **Flagging explicitly: the source material does not specify which use cases are considered "confidential" versus safe for commercial APIs with precision - this classification should be defined and signed off by the Product Owner/Tech Lead before implementation, rather than assumed by engineering.**
- **Compliance:** GDPR applies to any customer or merchant personal data processed by AI agents (e.g. the conversational assistant, claims triage) - data processing agreements and retention rules apply as they would to any other personal-data-handling service.
- **Organizational:** this is explicitly called out as a Phase 0 priority requiring a dedicated "AI Lead" - the interview strongly implies this should be a distinct role/team from general backend engineering, given its infrastructure (private LLM hosting) and governance requirements.

---

### 2.5 Market Intelligence / Data & Analytics Platform

#### CURRENT STATE

Not present on the current architecture diagram or as an explicit roadmap line item in Phase 0–2, though it's described at length in the interview as a major strategic and commercial goal: a price transparency index per product group (modeled explicitly on the "heizöl24" pricing-index example), a merchant-facing competitive pricing map ("green/yellow/red market share" - Phase 2, P1, Q4 2026 roadmap item), and monetized market intelligence reports ("€199/month" - Phase 3, P2, Q2 2027 roadmap item).

#### ISSUES & RISKS

- **Coupling:** if built by querying the operational databases of Order, Pricing, and Merchant & Listing directly (the path of least resistance), analytics workloads will compete with production traffic for the same database resources - a classic OLTP/OLAP contention problem, most damaging exactly when the data is most valuable (peak season, high order volume).
- **Scalability:** a price transparency index across "every filter you set" (per the interview - arbitrary combinations of product group, wood type, dimensions, dryness, etc.) implies a genuinely large combinatorial aggregation workload; this needs purpose-built analytical infrastructure, not ad-hoc reporting queries against transactional stores.
- **Clarity:** there is currently no named owner or component for this domain at all - for a capability described as central to the business's competitive strategy, this is a significant documentation and planning gap, not just a technical one.
- **Extensibility:** the roadmap explicitly plans to commercialize this data externally (paid reports, potentially the "logistics auction/bidding platform" in Phase 4, where carrier networks bid based on market-share transparency) - building this as an internal-only reporting afterthought would require significant rework to expose it as an external product later.

#### RECOMMENDATION

- **Owns:** an event-driven ingestion pipeline (consuming `price.changed`, `order.placed`, `listing.updated`, and similar events from operational domains), a data warehouse/analytical store, and the computation of derived products (price indices, competitiveness maps, market share by postcode/product group).
- **Does not own:** any operational/transactional data - it is strictly a downstream consumer via events, never a direct database reader of Order, Pricing, or Merchant services.
- **Integration pattern:** fully event/async - this domain should never be in the synchronous path of any customer- or merchant-facing transaction; if the analytics platform is down, checkout and pricing must continue to function normally.
- **Data ownership:** source of truth for aggregated/derived analytical products (indices, maps, reports); never a source of truth for any operational entity (order, price, listing) - those remain owned by their respective domains.

#### CONSTRAINTS

- **Business rule (non-negotiable):** the price index model is explicitly designed to be independent, market-driven, and non-negotiable evidence in supplier price negotiations ("I tell them, you know what, we go with the index... this job shows what the customers are currently paying") - the underlying data pipeline must be trustworthy and auditable, since it will be cited as a factual reference in commercial negotiations and potentially sold externally.
- **Compliance:** GDPR requires that market intelligence be aggregated/anonymized for any public-facing index; per-merchant competitive detail (e.g. "your market share in postcode X") is explicitly a permissioned, paid feature shown only to that merchant, not public data - this access control needs to be enforced at the data-platform level, not just the UI level.
- **Organizational:** given the stated ambition to commercialize this as a standalone product line, this domain likely warrants dedicated ownership (a data/analytics team) separate from the core transactional engineering teams, similar to the reasoning for Logistix.

---

### 2.6 Multi-Country Expansion (cross-cutting)

#### CURRENT STATE

The current architecture is implicitly single-country (Germany) - no domain currently carries an explicit country dimension.
Roadmap, Phase 2: "Austria (PaligoAT) market expansion - domain + localisation" [P2, Q4 2026]. Phase 3: Poland, Netherlands, Spain [Q1–Q2 2027]. Phase 3 (H2 2027): France, Italy. Interview: the domestic-to-domestic rule is explicit and strict - "we counted the supplier and the customer, they are always within the same country... we use the same approaching methodology, but we match domestic to domestic" - with country-specific domains already named (PaligoAT, PaligoPoland, PaligoNL, PaligoES, PaligoFR, PaligoIT).

#### ISSUES & RISKS

- **Coupling:** this is not a service in itself but a cross-cutting dimension - the risk is that any domain built without a `country` field today (Catalog, Merchant Listing, Pricing, Payment) will require a data-model migration, not just a configuration change, when Austria launches.
- **Scalability:** not a traffic-scaling concern; the risk is operational/engineering-scaling - each new country should be a configuration exercise (VAT rules, currency, localized content, legal domain), not a new codebase or forked deployment, given the "domestic-to-domestic" rule keeps the business logic identical across countries.
- **Clarity:** without country modeled explicitly and consistently across domains now, a new engineer joining during the Austria expansion would have no single, predictable place to look for "how does country-specific behavior work."
- **Extensibility:** this is precisely an extensibility concern by nature - the entire point of reviewing it now (per Phase 1, before Phase 2's Austria launch) is to avoid the roadmap's Phase 2 milestone becoming a re-architecture instead of a configuration rollout.

#### RECOMMENDATION

- **Owns:** nothing operationally - this is a shared data dimension (`country`) and a shared configuration concern (VAT rules, currency, legal/compliance rules, localized content per country), not a bounded service.
- **Does not own:** business logic, which should remain identical across countries per the domestic-to-domestic rule - country expansion should never require country-specific service logic, only country-specific configuration and content.
- **Integration pattern:** every domain (Catalog, Merchant Listing, Pricing, Order, Payment) should carry `country` as a required field from day one, even while only Germany is populated - this is a schema decision, not an integration pattern per se.
- **Data ownership:** no independent data ownership; country-specific configuration (VAT rules, currency) could reasonably live in a small shared configuration service that Pricing and Payment both reference.

#### CONSTRAINTS

- **Business rule (non-negotiable):** supplier and customer are always matched within the same country - cross-border transactions are explicitly out of scope ("if they could be potentially in Nigeria, be it at the border to Cameroon... different currencies... [we] match domestic to domestic"). Traders/importers are the mechanism for cross-border product sourcing (e.g. importing granite from Poland into a German warehouse) - the imported product is still sold as a domestic German listing, not a cross-border transaction.
- **Compliance:** VAT, consumer protection law, and data residency requirements vary by country and must be modeled as country-specific configuration across all relevant domains (Pricing, Payment, Order, Catalog content).
- **Organizational:** country rollout is a cross-team coordination exercise (every domain owner needs to confirm their service supports the `country` field before a new market launches) - worth a lightweight, explicit "country readiness checklist" owned by whoever coordinates the Phase 2/3 rollout, rather than assuming each team will remember independently.

---

---

# Summary Tables

## Part 1 - Current Architecture

| Domain                        | Exists today?              | Primary risk today                          | Roadmap phase to address       |
| ----------------------------- | -------------------------- | ------------------------------------------- | ------------------------------ |
| Identity & Access             | Yes (partial)              | Two integration patterns (direct + gateway) | Phase 0/1                      |
| Storefront / API Gateway      | Yes (partial)              | Inconsistent entry points                   | Phase 0/1                      |
| Product Catalog & Master Data | Yes (folded into Directus) | Mixed with merchant listing data            | Phase 1 (3.1)                  |
| Merchant & Listing Service    | Yes (needs fix)            | Multi-warehouse duplication                 | Phase 1 (3.1), already planned |
| Pricing Engine                | Yes (miscoupled)           | Fed directly by Odoo ERP                    | Phase 1 (3.3)                  |
| Order & Checkout              | Yes (inside Odoo)          | No dedicated order-line state machine       | Phase 1 (3.2)                  |

## Part 2 - Future Architecture

| Domain                       | Exists today?                 | Primary risk today                          | Roadmap phase to address            |
| ---------------------------- | ----------------------------- | ------------------------------------------- | ----------------------------------- |
| Payment & Escrow             | Partial/unclear (inside Odoo) | No dedicated domain; compliance unconfirmed | Phase 1 (3.2), Phase 2 payout       |
| Logistics Connector          | No (in 3NRG Odoo)             | Entangled with 3NRG, not yet standalone     | Phase 1 (3.4), already planned      |
| Ratings & Claims             | No                            | Not yet modeled as platform-mediated        | Phase 1 (3.2), Phase 2 AI triage    |
| Paligo AI OS                 | No                            | No tiering by data sensitivity yet          | Phase 0 infra, Phase 2 features     |
| Market Intelligence Platform | No                            | No event pipeline; risk of OLTP contention  | Phase 2/3                           |
| Multi-Country Expansion      | No (implicit DE-only)         | No `country` field across domains           | Phase 1 (design), Phase 2 (Austria) |

---

_Prepared for engineering review. Flagged open items requiring Product Owner/legal confirmation before implementation: Payment & Escrow licensing/compliance model (Part 2, §2.1), AI data-sensitivity classification (Part 2, §2.4), and basket-consolidation rules for Order & Checkout (Part 1, §1.6)._
