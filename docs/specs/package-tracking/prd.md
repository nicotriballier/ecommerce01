# PRD: Package tracking across major US carriers

| Owner | Status | Last updated |
|---|---|---|
| Nicolas Triballier | Draft | 2026-11-15 |

## 1. Problem
Today an admin types a free-text carrier and tracking code when labeling or shipping an order (`/admin/orders/{id}`), then manually clicks **mark shipped** and later **mark delivered**. Customers see the raw carrier string and tracking code on `/order/{id}`. They have no link and no progress, and they get no update between "shipped" and "delivered".

Consequences:
- Shoppers have to copy the code into the carrier's site. "Where is my order?" stays opaque.
- `delivered` is only as accurate as an admin remembering to click it. That drives the `order_delivered` email and the review gating that depends on delivery.
- Typos in the carrier or tracking code are never caught.
- Exceptions (failed delivery, returned to sender) go unnoticed until the customer complains.

## 2. Goals and success metrics
| Goal | Metric | Target |
|---|---|---|
| Shoppers can follow their parcel without leaving the store | % of shipped orders whose order page shows ≥1 carrier scan event | ≥ 95% within 24 h of label |
| Fulfillment state reflects reality without manual clicks | % of fulfillments moved to `shipped` / `delivered` automatically from carrier data | ≥ 90% |
| Delivery problems are caught early | Median time from carrier exception to admin visibility | < 1 h |
| Tracking data is valid | % of tracking numbers rejected by the carrier after entry | < 2% |

**Non-goals:** buying shipping labels, rate shopping at checkout, returns logistics, international shipping.

## 3. Users and use cases
- **Shopper** (signed-in customer or guest with order link)
  - As a shopper, I want to see where my parcel is, and when it will arrive, on my order page so I don't have to visit the carrier's site.
  - As a shopper, I want an email when my parcel is out for delivery, delivered, or has a problem, so I can act.
- **Admin / warehouse operator**
  - As an operator, I want to pick the carrier from a list and have the tracking number validated, so typos are caught at entry.
  - As an operator, I want orders to move to shipped/delivered on their own from carrier scans, so I stop clicking buttons.
  - As an operator, I want to see every shipment with a delivery exception in one place, so I can contact the customer.

## 4. Requirements
### Functional
| ID | Requirement | Priority |
|---|---|---|
| FR-1 | Supported carriers: **UPS, USPS, FedEx, DHL Express** (US domestic). An "Other" option keeps today's free-text behavior with no live tracking. | Must |
| FR-2 | Admin selects the carrier from a dropdown. The carrier is auto-suggested from the tracking number format. The tracking number format is validated per carrier. | Must |
| FR-3 | When a valid tracking number is saved, the store registers it with the tracking provider and starts receiving updates. | Must |
| FR-4 | Carrier status is normalized to: `pre_transit`, `in_transit`, `out_for_delivery`, `delivered`, `exception`, `returned_to_sender`, `unknown`. | Must |
| FR-5 | First carrier acceptance scan moves a `labeled` fulfillment to `shipped`. A `delivered` scan moves `shipped` to `delivered`. Both use the existing transitions and events. | Must |
| FR-6 | Customer order page shows the current status, estimated delivery date, a scan-event timeline (newest first), and a link to the carrier's tracking page. | Must |
| FR-7 | Admin order detail shows the same timeline plus a **Refresh tracking** action. | Must |
| FR-8 | Admin orders list can be filtered to shipments in `exception` / `returned_to_sender`. | Should |
| FR-9 | Emails: the existing `order_shipped` email includes the carrier tracking link. New `out_for_delivery` and `delivery_exception` emails. The existing `order_delivered` email fires on carrier-confirmed delivery. | Should |
| FR-10 | Demo mode and local dev work with no carrier credentials, using a fake provider that simulates progress. | Must |
| FR-11 | Admin can still manually override status (existing `AdminOverrideStatus`). A manual override stops automatic transitions for that fulfillment. | Must |

### Acceptance criteria
- **AC-1** Given an admin on a scheduled order's detail page, when they choose carrier "UPS" and enter `1Z999AA10123456784`, then the fulfillment is labeled with carrier `ups` and the tracking number is registered with the provider.
- **AC-2** Given an admin enters a tracking number that doesn't match the selected carrier's format, when they submit, then the form re-renders with a field error and nothing is saved.
- **AC-3** Given a labeled fulfillment, when the first `in_transit` update arrives (webhook or poll), then the fulfillment becomes `shipped`, `fulfillment.OrderShipped` and `fulfillment.OrderShippedECST` are published once, and the shipped email includes a working carrier link.
- **AC-4** Given a shipped fulfillment, when a `delivered` update arrives, then the fulfillment becomes `delivered` with `delivered_at` set to the carrier's event time, and `fulfillment.OrderDelivered` is published once.
- **AC-5** Given the same carrier update is delivered twice (webhook retry, or webhook plus poll), when both are processed, then exactly one tracking event is stored and no event is published twice.
- **AC-6** Given updates arrive out of order, when an older `in_transit` arrives after `delivered`, then the timeline includes it in chronological position and the fulfillment status does not regress.
- **AC-7** Given a shopper views `/order/{id}` for a shipped order, then they see the normalized status, ETA (if known), the event timeline, and a carrier link. They see only their own orders (existing authz).
- **AC-8** Given an `exception` update arrives, then the admin orders list flags the order, and the customer receives one `delivery_exception` email per exception episode.
- **AC-9** Given `APP_DEMO=true` or no provider credentials, when an admin labels an order with any supported carrier, then the fake provider advances it through `in_transit → out_for_delivery → delivered` on a short schedule.
- **AC-10** Given an admin manually overrides the status of a fulfillment, when later carrier updates arrive, then events are still appended to the timeline but the status is not changed automatically.
- **AC-11** Given a webhook request with an invalid signature, then it is rejected with 401 and nothing is stored.

### Non-functional
- **Reliability:** Webhooks are the primary update path and polling is the fallback. A provider outage must never block admin labeling or shipping.
- **Performance:** The order page renders from stored events only, with no live carrier calls on page load. p95 is unchanged.
- **Security:** Webhooks are authenticated by HMAC signature. Provider API keys come from env config and are masked in logs. Tracking numbers are not PII, but recipient names and addresses in raw payloads are, so we don't store raw payloads beyond what's needed (see tech spec).
- **Cost:** Tracking registrations are bounded to one per fulfillment (plus re-registration on tracking-number change).

## 5. UX
**Storefront `/order/{id}`:** the current fulfillment `<p>` line becomes a "Shipment" card:
- Status pill (e.g. *In transit*) and "Estimated delivery: Tue, Nov 18" if known.
- Carrier name and tracking number, linked to the carrier's public tracking page.
- Timeline: date/time, location, description. Collapsed after 5 events.
- States: *Label created, awaiting carrier pickup* (labeled, no scans yet). *Tracking unavailable* ("Other" carrier: raw code shown, as today).

**Admin `/admin/orders/{id}`:**
- Carrier `<select>` (UPS, USPS, FedEx, DHL Express, Other) replaces both free-text carrier inputs. Inline validation error on the tracking field.
- Same timeline. **Refresh tracking** button (HTMX, swaps the card). "Last checked 12 min ago."
- Exception banner when the latest status is `exception`.

**Admin `/admin/orders`:** a "Delivery issues" filter chip.

**Emails:** `order_shipped` gets a "Track your package" button. New `out_for_delivery` and `delivery_exception` templates follow the existing email layout.

## 6. Scope
**In scope:** tracking for UPS, USPS, FedEx and DHL Express on US domestic shipments. Carrier selection and validation in admin. Automatic shipped/delivered transitions. Customer and admin timelines. Three email touchpoints. Fake provider for dev and demo.

**Out of scope:**
- Purchasing labels or rates.
- Multiple packages per order (the schema keeps one fulfillment per order).
- Returns flow (`returned` status stays reserved).
- SMS or push notifications.
- Regional carriers (OnTrac, LaserShip, etc.). The design allows adding them later.
- Guest tracking without an order link.

## 7. Assumptions, dependencies, risks
| Type | Item | Mitigation |
|---|---|---|
| Assumption | We use a tracking aggregator (proposed: EasyPost Trackers API) rather than four direct carrier integrations. | Provider sits behind a port. Direct adapters can be added per carrier. See tech spec §4. |
| Dependency | Aggregator account and API key, plus a webhook endpoint reachable from the internet in staging/prod. | Fake provider covers dev, CI and demo. Polling covers missing webhooks. |
| Risk | Carrier status vocabularies differ, so normalization can mis-map edge statuses. | Map unknowns to `unknown`, store the raw carrier status for display, and unit-test the mapping table per carrier. |
| Risk | Auto-transitions fire the existing shipped/delivered emails at the wrong time. | Transitions only on normalized statuses. Idempotent per fulfillment. Admin override disables automation (AC-10). |
| Risk | Per-tracker cost at volume. | One registration per fulfillment. Stop polling at terminal statuses. |

## 8. Open questions
| # | Question | Proposed default | Decision |
|---|---|---|---|
| 1 | Aggregator (EasyPost / Shippo / AfterShip) or direct carrier APIs? | EasyPost Trackers: pay-per-tracker, webhooks, covers all four carriers. | **EasyPost** (2026-11-15) |
| 2 | Should the `out_for_delivery` email be sent at all, or is it too noisy? | Send it. It's a single email per shipment. | **Send it** (2026-11-15) |
| 3 | Should a carrier `delivered` scan auto-trigger delivery even if the admin never clicked **mark shipped**? | Yes. Apply `Ship` then `Deliver` in sequence from `labeled`. | **Yes** (2026-11-15) |
| 4 | Do we need a public, unauthenticated tracking page (e.g. `/track/{token}`) for guests? | No. Out of scope for v1. | |

## Changelog
- 2026-11-15: initial draft
- 2026-11-15: decisions confirmed by Nicolas: EasyPost as the provider; UPS/USPS/FedEx/DHL Express as the carriers; one package per order in v1; out-for-delivery email is sent; a carrier `delivered` scan auto-delivers an order that is still `labeled`. Q4 is still open.
