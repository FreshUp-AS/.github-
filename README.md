# FreshUp

> The world's first real-time AI distribution engine for services — replacing booking completely.

---

## What this is

Today you wait days or weeks for one provider. Not because capacity is missing — but because everyone goes to the same few while the rest stay empty.

People wait because they trust the outcome they know, not the provider itself. Once the outcome is the same regardless of who delivers it, waiting stops making sense. You just want it done — fast, close, now.

FreshUp does the opposite of booking platforms. We never show providers. Only results.

You pick the service. The engine finds who's best at that exact service, available right now, closest to you — and fires the request instantly. The first to accept gets the job. They fill capacity that would otherwise sit empty. You get the service in minutes instead of weeks.

This is how we built it.

---

## Built for AI agents

FreshUp is built on MCP (Model Context Protocol). AI agents — Claude, ChatGPT, anything — can book services on your behalf without opening an app.

```
> book("skin fade")
> matched in 42ms
> provider · 1.2 km away
```

Booking becomes an API call. The engine handles the rest.

---

## Core systems

### 1. Service hierarchy

The service card is the atomic unit of matching. Everything above it is UI navigation.

```
Mode
└── Target
      └── Category
            └── Service  ← the engine matches on this
```

Example:
```
Beauty
└── Male
      └── Haircut
            └── Skin Fade
```

A provider only matches a request if they are registered at every level of the exact path. A provider registered for Beauty → Male → Haircut will never match a Beauty → Female → Nails request.

---

### 2. Per-service rating

A provider is not rated as a person. Each service gets its own rating.

You can be best at Skin Fade without being best at Mid Fade. A poor rating on Beard Trim does not affect a provider's Skin Fade card. Jobs go to whoever scores highest on that exact service.

```
Provider: Alex

Service          Jobs    Rating    Status
────────────────────────────────────────────
Skin Fade        247     ★ 4.9     Active
Low Fade         183     ★ 4.8     Active
Mid Fade         118     ★ 4.6     Active
Buzz Cut          94     ★ 4.7     Active
Beard Trim        42     ★ 3.8     Locked
```

A low rating on one service locks that card — not the provider. They still receive jobs on the services they're good at.

---

### 3. Dispatch engine

Requests are distributed in **batches** (who is eligible) and **waves** (in what order within each batch).

#### Batches — distance × rating

We try each distance ring twice — first the 5-star providers, then the 4-star providers — before expanding outward. The reasoning: a skilled 4-star provider 2 km away is a better match than a 5-star provider 7 km away. Proximity means faster delivery.

```
Batch   Filter        Window        Logic
────────────────────────────────────────────────────────────────
1       0–3 km, 5★    sec 0–10      Closest, top rating
2       0–3 km, 4★    sec 10–20     Still close, slightly lower rating
3       3–6 km, 5★    sec 20–30     Expand distance, top rating
4       3–6 km, 4★    sec 30–40     Expand distance, lower rating
5       6–10 km, 5★   sec 40–50     Wider radius, top rating
6       6–10 km, 4★   sec 50–60     Final expansion
```

If all 6 batches exhaust without an accept → "No providers available right now."

1★, 2★ and 3★ providers never receive dispatch requests.

#### Waves — tier within each batch

Within each batch, providers are notified in waves based on tier:

```
Wave    Tier      Timing
──────────────────────────────────────────
1       Gold      t = 0s   (notified first)
2       Silver    t = 3s   (if no accept)
3       Bronze    t = 6s   (if no accept)
```

Race condition protection: only one provider can accept a job. The first atomic database update wins — all others see "Job already taken."

---

### 4. Provider tier system (Gold / Silver / Bronze)

Tiers are recalculated daily on a rolling 30-day window. They are earned through behavior — not assigned permanently.

```
score = (accept_rate + completion_rate + response_speed) / 3
```

All three metrics use the same denominator — requests received — so they are directly comparable and cannot be gamed by selectively accepting few requests.

| Metric | Formula |
|--------|---------|
| Accept rate | accepted / received |
| Completion rate | completed / received |
| Response speed | total_speed_points / received |

#### Response speed — points per request

The speed points system maps directly to the dispatch wave windows:

| Response time | Points | Maps to |
|---------------|--------|---------|
| Within 3s | 1.00 | Wave 1 (Gold window) |
| Within 6s | 0.50 | Wave 2 (Silver window) |
| Within 9s | 0.25 | Wave 3 (Bronze window) |
| After 9s / no response | 0 | Missed |

A provider who consistently responds in Wave 1 earns full points. One who misses most requests earns close to zero — regardless of how many they eventually accept.

**Example** — Provider with 100 requests received:

```
60 responses within 3s  →  60 × 1.00  =  60.00
15 responses within 6s  →  15 × 0.50  =   7.50
 5 responses within 9s  →   5 × 0.25  =   1.25
20 no response          →  20 × 0.00  =   0.00
                                       ────────
Total speed points                     =  68.75

Response speed = 68.75 / 100 = 68.75%
Accept rate    = 80 / 100    = 80%
Completion rate = 76 / 100   = 76%

Score = (80 + 76 + 68.75) / 3 = 74.9% → Gold
```

#### Tier thresholds

| Tier | Score | Dispatch wave |
|------|-------|---------------|
| Gold | ≥ 70% | Wave 1 (first) |
| Silver | 50–69% | Wave 2 |
| Bronze | < 50% | Wave 3 |

New providers start as Silver with a 30-day grace period — they cannot drop below Silver until they have 30 days of data. After that, full evaluation applies.

**Tiers are visible to providers as a gamification mechanic. Customers never see tiers.**

---

### 5. Dynamic pricing engine

Price is not fixed. It breathes with the market — the same way supply and demand work in reality.

```
used_capacity = (active_bookings_last_30min / online_providers) × 100%
multiplier    = −0.30 + (used_capacity / 100 × 0.60)
```

Linear scale — no sudden jumps:

| Market state | Capacity | Multiplier |
|---|---|---|
| Many available | 0% | −30% |
| Balanced | 50% | ±0% |
| Almost full | 100% | +30% |

When providers are idle, price drops to fill empty capacity. When demand spikes, price rises to reward providers who are available — and to balance the market.

**Example** — service with 500 kr base price:

| Capacity | Adjustment | Customer price |
|----------|------------|----------------|
| 10/10 busy | +30% | 650 kr |
| 7/10 busy | +12% | 560 kr |
| 5/10 busy | ±0% | 500 kr |
| 3/10 busy | −12% | 440 kr |
| 0/10 busy | −30% | 350 kr |

Price is updated every 5–10 minutes — not on every booking. The moment a customer starts a booking, price is locked. It cannot change mid-flow.

#### Commission model

FreshUp takes 20% on service and add-ons. 0% on delivery.

Commission is baked into the customer price before it is displayed — the provider only ever sees their net amount:

```
customer_price = provider_base_price / 0.80
```

Example: provider base price 300 kr → customer sees 375 kr → FreshUp keeps 75 kr.

Providers never feel money being "taken" — they accept based on what they will actually earn.

#### Base price discovery

FreshUp does not research prices. Providers supply them at signup.

Each provider enters a typical price per service in their area. The system calculates a trimmed mean (top and bottom 10% removed) across all providers in the same GPS-determined area. A base price becomes active once at least 5 providers have contributed.

Area is determined by GPS at signup — never manually selected. This prevents cross-market price contamination (Pakistani prices never mix with Norwegian prices).

---

### 6. Service mode — home visit vs. at provider

Every booking specifies a delivery mode:

- **At provider** — customer travels to the provider's location
- **Home visit** — provider travels to the customer

Delivery fee for home visit: `150 kr base + 10 kr/km` — 100% to the provider, 0% FreshUp commission. The fee compensates actual driving cost, not platform revenue.

Providers declare which modes they support per individual service.

---

### 7. Provider self-rating per service

At signup, each provider self-rates their competence per service (1–5 stars). This rating is never shown to customers. It feeds:

- Dispatch batch eligibility (determines which batch tier a provider enters)
- Matching engine weighting
- Internal analytics

A provider can be 5★ on Skin Fade and 3★ on Beard Trim. They are dispatched accordingly per request type — not as a single averaged entity.

---

### 8. Capacity heatmap

The engine maintains a live grid of used capacity per service per geographic cell, updated every 5–10 minutes. This drives:

- Dynamic pricing multipliers per area
- Visual demand overlay on the customer map
- Dispatch pool selection

---

## Database

51 tables across 8 domains:

| Domain | Key tables |
|--------|-----------|
| Service catalog | modes, targets, categories, services, service_modes, service_addons |
| Provider | provider_details, provider_skills, provider_verifications, provider_earnings_ledger |
| Dispatch | dispatch_batches, dispatch_attempts, order_offers, matching_runs, matching_candidates |
| Orders | orders, order_events, order_addons |
| Pricing | pricing_areas, provider_price_inputs, area_base_prices, booking_price_locks, demand_zones, area_capacity_snapshots |
| Payments | payments, payouts, payment_methods, refunds |
| Communication | conversations, messages, support_tickets |
| Real-time | provider_realtime_locations, customer_realtime_locations |

---

## Migrations

All schema changes must be committed as migration files — no direct Supabase dashboard edits:

```
supabase/migrations/YYYYMMDDHHMMSS_description.sql
```

---

## Project

- Owner: Munib Hadi / Fresh Up AS, Oslo, Norway
- Stack: Supabase · PostgreSQL · Stripe Connect · Mapbox · React Native (Expo) · Next.js · MCP
