<!-- Synced with vi version: 2026-05-07T10:30:00Z -->

# Period-Based V.Comm Rate Adjustment — Functional Specification

> **Status**: FINALIZED
> **Author**: Planning Plugin (Auto-generated)
> **Created**: 2026-04-16T10:00:00Z
> **Last Updated**: 2026-05-07T10:30:00Z
> **Source Documents**:
> - `ELLIS VComm Rate Period Adjustment Spec v1.0` (DOCX, April 2026)
> - Prototype simulator v2 (`remixed-ca0f0e7d.tsx`) — interactive 3-mode UX reference
> **Module**: Traders > Vendor Contract > Dynamic Rate > Net Provide
> **Subtitle**: Dynamic Commission Override by Booking Period & Stay Season

---

## 1. Overview

### 1.1 Purpose

The current ELLIS system supports only **a single fixed V.Comm Rate** per vendor contract (e.g., 15%), regardless of booking date or stay season. This causes:

- Inability to optimize margin by high/low season (Golden Week, Cherry Blossom, Year-End)
- No mechanism to incentivize early bookings or charge a premium for last-minute bookings
- SCM teams must create separate contracts or request manual overrides, increasing error risk
- Competitive disadvantage compared to global OTAs that all support period-based commissions

The **Period-Based V.Comm Rate Adjustment** feature allows commission rate overrides in 3 modes: Booking Date (Lead Time), Stay Date (season), or Combined (both combined), configured individually per vendor contract.

### 1.2 Target Users

| Role | Description |
|------|-------------|
| SCM Manager | Configure override rules for vendor contracts, manage seasons and Lead Time tiers |
| SCM Staff | View configurations, use the simulation panel to check rates |
| Admin (all) | Full access to the feature — no separate permission levels |

### 1.3 Success Metrics

| KPI | Target |
|-----|--------|
| Reduce commission configuration time | 50% reduction compared to manually creating separate contracts |
| Contract Override adoption rate | ≥ 30% of active contracts within the first 3 months |
| Peak season margin improvement | +2–5% commission margin compared to flat rate |
| Zero disruption | 100% of existing contracts operating normally after deploy |

---

## 2. User Stories

| ID | Role | Goal | Priority |
|----|------|------|----------|
| US-001 | As a SCM Manager | I want to enable commission override mode by Lead Time for a vendor contract to incentivize early bookings with lower rates and charge a premium for last-minute bookings | P0 |
| US-002 | As a SCM Manager | I want to configure commission by season (stay date) to optimize margin during peak seasons (Golden Week, Year-End) | P0 |
| US-003 | As a SCM Manager | I want to combine both booking and stay conditions with a resolution logic (Higher Wins, Lower Wins, etc.) for the most granular control | P0 |
| US-004 | As a SCM Staff | I want to use the simulation panel to check the resulting rate before saving the configuration | P0 |
| US-005 | As a SCM Manager | I want to view a rule change history (audit trail) to track who changed what and when | P1 |
| US-006 | As a SCM Manager | I want to manually recalculate the V.Comm Rate for existing bookings when rules change | P1 |
| US-007 | As a SCM Manager | I want to create cross-year seasons (e.g., Dec 15 – Jan 15) without needing to split them into 2 separate rules | P1 |
| US-008 | As a Admin | I want the system to fall back to Base Rate when no rule matches, to ensure every booking has a commission rate | P0 |

---

## 3. Functional Requirements

### FR-001: Override Mode Activation

**Description**: Each vendor contract has a "Period-Based Override" toggle that enables or disables commission override mode. When disabled, V.Comm Rate operates as it currently does (single flat rate).

**Business Rules**:
- BR-001: Override mode defaults to `'none'` (off) for all existing and new contracts
- BR-002: When override is enabled, the admin must select a mode: `'booking'`, `'stay'`, or `'combined'`
- BR-003: Only 1 mode is active at a time per contract
- BR-004: When override is disabled (reverting to `'none'`), rules **are still kept** in the database — they are not deleted
- BR-004a: **UI placement** (finalized per current ELLIS screenshots — Option B side drawer):
  - Within the `Company Master > Vendor Contract > Dynamic Rate sub-tab` modal: the Net Provide row has a "Period-Based Override" sub-card with a status badge + button [Configure/Edit Override]
  - Click button → opens Override Drawer (slides in from the right, 70% viewport, max 1200px)
  - Drawer contains the entire flow: ToggleOverride, Mode Selector, Rule Table, Visual Map, Calendar, Cross-Reference Matrix, Live Simulator, Audit Summary
  - Save coordination is atomic — see FR-011

**Acceptance Criteria**:
- [ ] AC-001: "Period-Based Override" sub-card is displayed in the parent modal (modal anchor) next to the V.Comm Rate field
- [ ] AC-002: When Override is OFF → badge "OFF — single flat rate" + button "[⚙ Configure]"
- [ ] AC-003: When Override is ON → badge "✓ ON | Mode: {X} | {N} active rules" + buttons [Edit] [History]
- [ ] AC-003a: Drawer slide-in animation ≤300ms (NFR-018), parent modal becomes inert when drawer is open

---

### FR-002: Booking Date Mode (Lead Time Tiers)

**Description**: Commission rate is determined by the number of days between the booking creation date and check-in date (Lead Time).

**Business Rules**:
- BR-005: **Lead Time formula (DST-safe)**: `Lead Time = DATE_DIFF(check_in_date, booking_date)` calculated in calendar days based on **DATE-only components** (year/month/day) at the **server timezone fixed (UTC or JST, project-wide constant)**. **Millisecond arithmetic is FORBIDDEN** (`Math.round((d1-d2)/86400000)`) — this will desync at DST boundaries. The frontend simulator must use the same helper as the backend (shared library)
- BR-006: Admin creates tiers with `lead_min_days` (inclusive) and `lead_max_days` (inclusive, NULL = unlimited)
- BR-007: Each tier has a `priority` (integer, lower = higher precedence), `comm_rate` (0–50%), and `is_active` toggle
- BR-008: When multiple tiers match, the tier with the **lowest priority number** is applied
- BR-009: **Tie-break (choose 1 single criterion)**: if 2 tiers share the same priority and both match → prefer `created_at DESC` (the more recently created rule wins). Secondary tie-break: `id DESC` (only when created_at is identical to the second). Removes ambiguity between "larger ID OR newer created_at" from the previous version
- BR-010: If no tier matches (booking falls in a gap between tiers), **Base Rate** is used
- BR-010a: **Boundary semantics (inclusive on both ends)**: Lead=`lead_min_days` → match. Lead=`lead_max_days` → match. Lead=`lead_max_days+1` → no match (check next tier)

**Acceptance Criteria**:
- [ ] AC-004: Display rule table with columns: Priority, Tier Name, Lead Time Range, V.Comm Rate, Active, Actions
- [ ] AC-005: Full CRUD for booking rate rules
- [ ] AC-006: Warning when a gap between Lead Time tiers is detected
- [ ] AC-007: Warning when a duplicate priority is detected

**Default Tiers (Suggested)**:
| Tier | Lead Time | Rate | vs Base (15%) | Rationale |
|------|-----------|------|---------------|-----------|
| Early Bird | D-60+ | 10% | -5% | Guaranteed demand — incentivize early booking |
| Standard | D-30~59 | 13% | -2% | Normal planning window |
| Normal (Base) | D-15~29 | 15% | 0% | Base rate applies |
| Last Minute | D-7~14 | 18% | +3% | Higher fill value |
| Urgent | D-0~6 | 20% | +5% | Distressed inventory — last-minute fill |

**Use Case**: Yield-focused hotels, city hotels with volatile last-minute demand. Rationale (Global Standard): Last-minute bookings fill empty rooms (high incremental value → higher commission), while early bookings provide demand certainty (lower commission as competitive advantage).

---

### FR-003: Stay Date Mode (Season Tiers)

**Description**: Commission rate is determined by the season/period that the check-in date falls within. Seasons are defined using MM-DD ranges, repeating annually.

**Business Rules**:
- BR-011: Seasons are stored as `stay_from` (MM-DD) and `stay_to` (MM-DD), recurring annually
- BR-012: **Cross-year season support**: when `stay_from > stay_to` (e.g., `12-15` to `01-15`), the system understands this as a cross-year season
- BR-012a: **1-day season**: when `stay_from == stay_to` (e.g., `05-03` ~ `05-03`) → interpreted as a season of exactly 1 day, NOT cross-year. Matches when check-in MM-DD == stay_from
- BR-013: Overlap between seasons is **permitted** — the priority field resolves conflicts (lowest priority number = highest precedence). Supports N≥2 overlapping seasons (3+ seasons on the same date must sort `priority ASC, created_at DESC` and take the first)
- BR-014: Each season has a `season_name` (user-defined label, max 100 characters, Unicode support), `priority`, `comm_rate`, `is_active`
- BR-015: **Match logic (lexicographical comparison on 5-character MM-DD strings)**: check-in MM-DD falls within the range `stay_from–stay_to`. **Cases**:
  - Normal (`stay_from < stay_to`): match when `stay_from <= MM-DD <= stay_to` (both ends inclusive)
  - Cross-year (`stay_from > stay_to`): match when `MM-DD >= stay_from` OR `MM-DD <= stay_to`
  - 1-day (`stay_from == stay_to`): match when `MM-DD == stay_from`
- BR-015a: **Leap year handling (Feb 29)**:
  - Check-in = `02-29` (leap year) in Off-Peak season (`01-01`~`03-31`) → match (because `02-29` lexicographically falls between `01-01` and `03-31`)
  - `stay_to = 02-29` in a non-leap year: can still be stored in the DB; match logic uses string comparison so `02-28 <= 02-29` (true) → check-in 02-28 in a non-leap year matches. Leap year: 02-29 == stay_to → match
  - `stay_from = 02-29` for a 1-day season: only matches in a leap year (the date 02-29 does not exist in a non-leap year, so check-in `02-29` cannot occur in a non-leap year)
  - UI validation: warning (non-blocking) when admin enters `stay_from` or `stay_to` = `02-29`: "Date 02-29 only exists in leap years. The match rule may behave differently than expected."

**Acceptance Criteria**:
- [ ] AC-008: Display season rule table with columns: Priority, Season Name, Stay Period, V.Comm Rate, Active, Actions
- [ ] AC-009: Full CRUD for stay rate rules
- [ ] AC-010: Support entry of cross-year periods (e.g., Dec 15 – Jan 15)
- [ ] AC-011: Warning when overlapping periods are detected (informational, does not block save)
- [ ] AC-012: Annual calendar view displaying a visual map of seasons (see prototype TSX)

**Default Seasons (Suggested — JP Context)**:
| Season | Period | Rate | vs Base | JP Context |
|--------|--------|------|---------|------------|
| Off-Peak (Winter) | Jan 1 – Mar 31 | 12% | -3% | Low demand |
| Cherry Blossom | Apr 1 – Apr 28 | 16% | +1% | Hanami peak |
| Golden Week | Apr 29 – May 6 | 18% | +3% | National holiday |
| Shoulder (Early Summer) | May 7 – Jun 30 | 14% | -1% | Moderate demand |
| Summer Peak | Jul 1 – Aug 31 | 17% | +2% | School vacation |
| Autumn Foliage | Oct 1 – Nov 30 | 16% | +1% | Momiji season |
| Year-End / NYE | Dec 1 – Dec 31 | 18% | +3% | Oshogatsu prep |

**Use Case**: Resort properties, seasonal destinations (JP ski, beach, festivals). Rationale (Global Standard): Most widely adopted model across global OTAs. Aligns with hotel revenue management cycles — premium capture during peak demand, competitive positioning during off-peak.

---

### FR-004: Combined Mode (Booking + Stay)

**Description**: Both Booking and Stay conditions are evaluated independently, producing 2 candidate rates. A resolution logic determines the final rate.

**Business Rules**:
- BR-016: Admin selects 1 of 5 resolution logics when enabling Combined mode:

| Logic | Formula | Description | Recommended For |
|-------|---------|-------------|-----------------|
| Higher Wins (Recommended) | `MAX(B, S)` | Takes the higher rate — maximizes revenue for OMH | Default setting. Revenue-maximizing contracts |
| Lower Wins | `MIN(B, S)` | Takes the lower rate — hotel-friendly, builds partnership | Strategic partners, volume-based contracts |
| Stay Priority | `S → B fallback` | Stay rate takes priority; Booking rate only when stay = Base Rate | Hotels with strong seasonal pricing structures |
| Booking Priority | `B → S fallback` | Booking rate takes priority; Stay rate only when booking = Base Rate | Hotels focused on yield management |
| Additive | `Base ± ΔB ± ΔS` | Accumulates deltas from both — requires rate_cap | Advanced contracts requiring granular margin control |

**Cross-Reference Matrices** — Base Rate 15% (`*` denotes Base Rate fallback for that side):

**Higher Wins** — `MAX(B, S)`:
| Stay ↓ / Book → | Early Bird (10%) | Standard (13%) | Normal (15%)* | Last Min (20%) |
|-----------------|------------------|----------------|---------------|----------------|
| Off-Peak (12%) | **12%** | 13% | 15% | 20% |
| Golden Week (18%) | 18% | 18% | 18% | 20% |
| Summer Peak (17%) | 17% | 17% | 17% | 20% |
| Year-End (18%) | 18% | 18% | 18% | 20% |

**Lower Wins** — `MIN(B, S)`:
| Stay ↓ / Book → | Early Bird (10%) | Standard (13%) | Normal (15%)* | Last Min (20%) |
|-----------------|------------------|----------------|---------------|----------------|
| Off-Peak (12%) | **10%** | 12% | 12% | 12% |
| Golden Week (18%) | 10% | 13% | 15% | 18% |
| Summer Peak (17%) | 10% | 13% | 15% | 17% |
| Year-End (18%) | 10% | 13% | 15% | 18% |

**Stay Priority** — `Stay if Stay≠Base else Booking`:
| Stay ↓ / Book → | Early Bird (10%) | Standard (13%) | Normal (15%)* | Last Min (20%) |
|-----------------|------------------|----------------|---------------|----------------|
| Off-Peak (12%) | 12% | 12% | 12% | 12% |
| Golden Week (18%) | 18% | 18% | 18% | 18% |
| No Season (=15%)* | 10% | 13% | 15% | 20% |

**Booking Priority** — `Booking if Booking≠Base else Stay`:
| Stay ↓ / Book → | Early Bird (10%) | Standard (13%) | Normal (15%)* | Last Min (20%) |
|-----------------|------------------|----------------|---------------|----------------|
| Off-Peak (12%) | 10% | 13% | 12% | 20% |
| Golden Week (18%) | 10% | 13% | 18% | 20% |
| No Season (=15%)* | 10% | 13% | 15% | 20% |

**Additive** — `15 + ΔB + ΔS`, cap=25%, floor=0%:
| Stay ↓ / Book → | Early Bird (Δ-5) | Standard (Δ-2) | Normal (Δ0)* | Last Min (Δ+5) |
|-----------------|------------------|----------------|--------------|----------------|
| Off-Peak (Δ-3) | 7% | 10% | 12% | 17% |
| Golden Week (Δ+3) | 13% | 16% | 18% | 23% |
| Summer Peak (Δ+2) | 12% | 15% | 17% | 22% |
| Year-End (Δ+3) | 13% | 16% | 18% | 23% |

**Use Case**: Premium partners requiring granular control, high-volume contracts. The CrossRefMatrix component in the drawer is interactive — when admin changes the LogicCard, all Final Rate cells recalculate **within ≤300ms** (see NFR-017).

- BR-017: **Additive mode**: `Final = BASE_RATE + (BookingRate - BASE_RATE) + (StayRate - BASE_RATE)`. If `rate_cap` is not set, the system default cap of 25% applies
- BR-017a: **Additive floor & cap rules**:
  - **Cap (upper bound)**: `Final = min(Final, rate_cap)`. If `Final > rate_cap` → trigger E-VCR-011 informational warning
  - **Floor (lower bound)**: `Final = max(Final, 0%)` — negative commission rates are not permitted. Audit log records both the raw value (before floor) and the final value
  - Case `rate_cap < BASE_RATE` → trigger E-VCR-019 warning before saving (soft warning, save is permitted)
- BR-017b: **rate_cap persistence**: `rate_cap` is stored in `vendor_contract` independently from `combined_logic`. When user switches from Additive to another logic, `rate_cap` is retained in the DB. When switching back to Additive, the RateCapInput pre-fills the old value
- BR-018: If one of the two modes does not match any rule, Base Rate is used for that mode before applying the resolution logic
- BR-018a: **Both modes do not match (both Booking and Stay fall back to Base Rate)**: all 5 logics return Base Rate (verified):
  - `Higher Wins`: MAX(Base, Base) = Base
  - `Lower Wins`: MIN(Base, Base) = Base
  - `Stay Priority`: Stay = Base → fallback Booking = Base → Base
  - `Booking Priority`: Booking = Base → fallback Stay = Base → Base
  - `Additive`: Base + 0 + 0 = Base

- BR-018b: **Combined mode Add Rule form layout** (UX rule):
  - In Combined mode, the Add Rule form displays **both sections stacked simultaneously** (Booking Tier + Stay Season) — NOT a tab toggle
  - The user sees both setups at once without needing to click back and forth to switch
  - **Active section** (clicked or default): border-2 mode-color + bg `mode-{X}-light/30~40` + label "● ACTIVE" + fields enabled + required `*` markers
  - **Inactive section**: border 1px grey + bg-card + opacity-60 + label "click to activate" + fields disabled (visible but not editable)
  - Clicking a section → toggles active state, the other section becomes inactive
  - Common fields (V.Comm Rate, Priority, Active toggle) are placed below — apply to the active section's rule type
  - Apply Rule button → creates a rule of the active type → toast "Booking tier added" or "Stay season added"
  - **Edit mode** (clicking Edit on an existing rule): only renders the section for the rule's type — NO chooser. Common fields below apply directly to rule.type (no selection needed). **Phase 1 does NOT support converting rule type via Edit** (delete + recreate if needed).
  - **Mode switch behavior** (FR-001 BR-004 reinforce): when the user switches override mode (booking ↔ stay ↔ combined), rules for the other mode are still retained in the DB with `is_active=true` (not automatically deactivated). However, within the drawer, **the rule table only displays rules belonging to the current mode**:
    - mode='booking': only the Booking Tiers table is visible; the Stay Seasons table is HIDDEN (rules remain in DB)
    - mode='stay': only the Stay Seasons table is visible
    - mode='combined': both tables are visible (see BR-018b layout)
    - When the user switches back to a previous mode, the corresponding table reappears with rules intact — UX makes clear "rules don't disappear, they just hide by mode"
  - **Inactive section preserves data**: when the user clicks to toggle the active section in the Combined Add form, fields of the section that just became inactive retain their values (not cleared) → user toggles back, data is still there

**Acceptance Criteria**:
- [ ] AC-013: Display 5 resolution logic options as selectable cards/buttons
- [ ] AC-014: "Higher Wins" is marked as "Recommended"
- [ ] AC-015: Cross-reference matrix displays results for common stay × booking combinations
- [ ] AC-016: When "Additive" is selected, additionally display the `rate_cap` input field
- [ ] AC-017: All 5 logics are within Phase 1
- [ ] AC-017a: Combined mode Add Rule form displays 2 stacked sections (Booking + Stay), active section highlighted, inactive section dimmed + disabled inputs (BR-018b)

---

### FR-005: Live Simulation Panel

**Description**: The simulation panel allows admins to enter a Booking Date and Stay Date, displaying commission rate results for all 3 modes in parallel.

**Business Rules**:
- BR-019: Simulation runs **real-time** when the user changes input (debounce 300ms — NFR-004)
- BR-020: Displays: Lead Time (D-XX), matched rule name for each mode, final rate, comparison vs Base Rate
- BR-021: Simulation **does not save data** — for preview/verify purposes only before saving the configuration
- BR-022: Response time target: **under 300ms** for debounced re-compute (lower than user-perceptible threshold)
- BR-022a: **Default initial state when drawer is first opened for a contract with an active override**:
  - Active mode: from `vendor_contract.override_mode` (DB)
  - Booking Date: **today** (server time, formatted YYYY-MM-DD)
  - Stay Date: **today + 30 days** (default lead time = 30 → matches "Standard" tier with default tiers)
  - Lead Time: D-30 → "Standard" tier in default tiers
  - Combined Logic: from `vendor_contract.combined_logic` or `higher_wins` if NULL
  - **Full default expected results** (verifiable in tests):
    | Mode | Matched Rule | Rate | Note |
    |------|--------------|------|------|
    | Booking-Based | Standard (D-30~59) | 13% | Lead=30 is boundary inclusive |
    | Stay-Based | Depends on actual stay_date (today+30) | depends on season | E.g.: today=2026-05-05 → stay=2026-06-04 → matches "Shoulder" 14% |
    | Combined (higher_wins) | MAX(13, stayRate) | depends | E.g.: MAX(13,14)=14% |
  - Pre-condition: contract already has default seasons + tiers seeded (see FR-001 BR-004a)
- BR-022b: **Lead Time formula DST-safe (see BR-005)**: use helper `daysBetween(checkInDate, bookingDate)` shared between frontend/backend. Millisecond Math.round is forbidden. Frontend simulator imports from `@ellis/rate-engine-shared` (npm package or git submodule)
- BR-022c: **Simulation architecture decision = CLIENT-SIDE with shared evaluator**:
  - Simulation runs entirely client-side (no API call) → response < 50ms typical, no timeout
  - Uses the same evaluator code as the backend (shared library `@ellis/rate-engine-shared`) to ensure 100% parity
  - Cross-run unit tests: same input → frontend evaluator output == backend evaluator output (parity test required)
  - **E-VCR-010 is no longer an "API timeout"** but changed to "Calculation error" for local logic errors
- BR-022d: **Pre-condition seed data** (referenced by BR-022a): When user first enables the override toggle, show CTA "Apply default tiers/seasons?" for a quick start. No seeding by default — admin must create rules manually

**Acceptance Criteria**:
- [ ] AC-018: 3 parallel result panels: Booking-Based, Stay-Based, Combined
- [ ] AC-019: Each panel displays: mode name, matched rule, rate %, lead time info
- [ ] AC-020: When date input changes → results update within **300ms** (post-debounce)
- [ ] AC-020a: Display the Lead Time reference formula near the date inputs: "Lead Time = Check-in − Booking (calendar days)"
- [ ] AC-020b: The active mode tab is visually highlighted on the result card (border + background tint per color token)
- [ ] AC-020c: Frontend evaluator imports from `@ellis/rate-engine-shared` same as backend; parity test passes 100%

---

### FR-006: Rate Locking & Recalculation

**Description**: V.Comm Rate is **locked** at the time a booking is created. When rules change, existing bookings **do not automatically update**. Admins can manually recalculate.

**Business Rules**:
- BR-023: When a booking is **newly created**: evaluate rules at that moment → lock rate → write audit log
- BR-024: **Re-evaluate triggers** (auto): only triggered when 1 of the following fields changes in a way that affects rate evaluation:
  - `check_in_date` changes → re-evaluate (both Lead Time and season match may change)
  - `booking_date` changes (admin edits directly) → re-evaluate
  - **Do NOT re-evaluate** when: only `check_out_date` changes (Phase 1), guest count changes, room type changes (Phase 1 — Phase 2 may add), payment status changes
  - Each re-evaluation writes an audit log with a reason
- BR-025: When rules change (admin edits rules): existing bookings **retain** their locked rate — only applies to new bookings or when admin triggers a manual recalculate
- BR-026: Admin can **manually recalculate** — select booking(s) and trigger re-evaluation with current rules. Supports:
  - **Single**: from the Booking Detail page, button "[🔄 Recalculate V.Comm Rate]"
  - **Batch**: from the Booking List page, multi-select + toolbar action "[🔄 Recalculate Selected (N)]"
  - **Batch limits**: max 500 bookings per batch (Phase 1). Exceeding limit → error E-VCR-021. Phase 2 will increase to 5000 with a background job
- BR-027: Cancel & rebook: the new booking is treated as a completely new booking → lead time is calculated from the new booking date

**Acceptance Criteria**:
- [ ] AC-021: Booking audit trail records clearly: applied rate, matched rule ID(s), mode, timestamp, evaluation source (creation/modify/manual_recalculate)
- [ ] AC-022: "Recalculate" button is available for both single (Booking Detail) and batch (Booking List multi-select)
- [ ] AC-023: Recalculate displays a preview (old rate → new rate) before confirming; batch: summary table with pagination
- [ ] AC-023a: Batch recalculate handles partial failures: if N/M bookings fail, displays a specific error list, still commits M-N successes

---

### FR-007: Audit Trail

**Description**: All override rule configuration changes are logged and viewable via UI.

**Business Rules**:
- BR-028: **Unified audit log** — a single stream `vcomm_audit_log` table with action types:
  - `RULE_CREATED`, `RULE_UPDATED`, `RULE_DELETED`, `RULE_TOGGLED_ACTIVE`, `RULE_TOGGLED_INACTIVE`
  - `OVERRIDE_MODE_CHANGED` (none → booking/stay/combined or vice versa)
  - `COMBINED_LOGIC_CHANGED`, `RATE_CAP_CHANGED`
  - `RATE_LOCKED_AT_BOOKING_CREATION` (each new booking)
  - `RATE_RE_EVALUATED_ON_BOOKING_MODIFY`
  - `RATE_RECALCULATED_MANUAL_SINGLE`, `RATE_RECALCULATED_MANUAL_BATCH`
  - `EVALUATION_FALLBACK_BASE_RATE` (when no rule matches)
  - `EVALUATION_ERROR` (E-VCR-016 trigger)
- BR-029: Each log entry: `user_id`, `timestamp`, `action_type`, `entity_type` (rule/contract/booking), `entity_id`, `old_value` (JSON), `new_value` (JSON), `metadata` (JSON: matched_rule_ids, evaluation_mode, etc.)
- BR-029a: Audit log retention: 24 months online + archive after 24 months (cold storage). Phase 1 implements online only.

**Acceptance Criteria**:
- [ ] AC-024: **Inline summary** in the Override Drawer (displays the 5 most recent changes, scoped to override config + rules for the contract)
- [ ] AC-025: **"V.Comm Override" tab** in History popup (full details, filterable by all action types listed above)
- [ ] AC-025a: **Booking-level audit** accessible from Booking Detail page → "View V.Comm Audit Trail" — filter logs with entity_type=booking, entity_id=current booking
- [ ] AC-026: Both views are within Phase 1
- [ ] AC-026a: ActionTypeFilter in the History popup allows filtering by each action type listed in BR-028

---

### FR-008: Validation Rules

**Description**: Data constraints ensure consistency and safety.

**Business Rules**:
- BR-030: Commission rate: 0.00% – 50.00% (DECIMAL(5,2), 2 decimal places). Reject values outside this range. Allow 0.00% (no commission) with an optional business warning
- BR-031: Priority: integer ≥ 1. Soft warning when duplicate priority within the same contract/mode — save is permitted after confirmation
- BR-032: Period overlap: permitted, priority resolves conflicts. Display warning when overlap is detected (especially 3+ seasons overlapping → message lists all of them)
- BR-033: Lead time gap: warning when a gap exists (e.g., D-30~59 and D-61+, missing D-60). Bookings falling in the gap use Base Rate
- BR-034: Cross-year season: supported when `stay_from > stay_to` (see BR-015)
- BR-035: Additive mode cap: if `rate_cap` = NULL → system default = 25%. Range for `rate_cap`: 0.00% – 50.00% (same range as comm_rate)
- BR-035a: **rate_cap warning soft validations**:
  - `rate_cap < BASE_RATE` → E-VCR-019 warning: "Cap is lower than Base Rate, Additive may cap even when both deltas = 0"
  - `rate_cap = 0%` → E-VCR-019 stronger warning: "Cap = 0% — all bookings will have a rate of 0%"
- BR-036: When `override_mode != 'none'` and all rules are inactive/deleted → **mode is retained**, all bookings fall back to Base Rate (does not automatically switch back to 'none'). UI displays AllInactiveBanner in the drawer
- BR-037: At least 1 active rule: warning when all rules are inactive — does not block the operation
- BR-038: **rule_name / season_name validation**:
  - Max 100 characters (DB schema VARCHAR(100)). Inline error E-VCR-020 when exceeded
  - Supports Unicode (NFR-011): Japanese/Korean/Vietnamese/emoji
  - Server-side must HTML-escape when rendering audit log (XSS protection)
  - Not required to be unique within the same contract — duplicate names are allowed (admin may accidentally create multiple "Cherry Blossom"; soft warning)
- BR-039: **Optimistic locking**: each rule and contract has a `version` (INT, default 0). UPDATE statement always uses `WHERE id=? AND version=?` — version mismatch → trigger E-VCR-017 conflict dialog. Increment version after each successful save
- BR-040: **Authorization (NFR-007)**: all API endpoints (read rules, simulate, save rules, recalculate booking) must validate that `user_id` has permission for `contract_id` before executing. Cross-contract access → HTTP 403 + log security event (E-VCR-018)

**Acceptance Criteria**:
- [ ] AC-027: Validation messages display inline (do not use alert popups)
- [ ] AC-028: Warnings for overlap/gap/duplicate priority — informational, does not block save
- [ ] AC-029: Error for out-of-range rate — blocks save
- [ ] AC-029a: rule_name length validation triggered on blur event or on submit
- [ ] AC-029b: Concurrent edit conflict (2 users saving the same rule): the second user sees the E-VCR-017 dialog
- [ ] AC-029c: Cross-contract API call (tampered contract_id) is rejected with 403 + monitoring alert

---

### FR-009: Data Model

**Existing Table — Extend `vendor_contract`**:

| Field | Type | Constraint | Description |
|-------|------|-----------|-------------|
| override_mode | ENUM | NOT NULL, DEFAULT 'none' | 'none' \| 'booking' \| 'stay' \| 'combined' |
| combined_logic | ENUM | NULL (only when combined) | 'higher_wins' \| 'lower_wins' \| 'stay_priority' \| 'booking_priority' \| 'additive' |
| rate_cap | DECIMAL(5,2) | NULL | Max commission cap (for Additive mode); persists independently from combined_logic (BR-017b) |
| version | INT | NOT NULL, DEFAULT 0 | Optimistic locking (BR-039) |

**New Table — `booking_rate_rules`**:

| Field | Type | Constraint | Description |
|-------|------|-----------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | |
| contract_id | FK → vendor_contract | NOT NULL | |
| rule_name | VARCHAR(100) | NOT NULL | User-defined label, Unicode (BR-038) |
| lead_min_days | INT | NOT NULL | Min lead time (inclusive) |
| lead_max_days | INT | NULL = unlimited | Max lead time (inclusive) |
| comm_rate | DECIMAL(5,2) | NOT NULL, 0–50 | Override rate (%) |
| priority | INT | NOT NULL, ≥ 1 | Lower = higher precedence |
| is_active | BOOLEAN | DEFAULT true | Enable/disable |
| extra_conditions | JSON | NULL | **Phase 2 forward compatibility** (room_type, day_of_week, channel). Phase 1 always NULL, not validated |
| version | INT | NOT NULL, DEFAULT 0 | Optimistic locking |
| created_at | DATETIME | NOT NULL | Used for tie-breaking same priority (BR-009) |
| updated_at | DATETIME | NOT NULL | |

**New Table — `stay_rate_rules`**:

| Field | Type | Constraint | Description |
|-------|------|-----------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | |
| contract_id | FK → vendor_contract | NOT NULL | |
| season_name | VARCHAR(100) | NOT NULL | User-defined label, Unicode (BR-038) |
| stay_from | CHAR(5) | NOT NULL | MM-DD format |
| stay_to | CHAR(5) | NOT NULL | MM-DD format |
| comm_rate | DECIMAL(5,2) | NOT NULL, 0–50 | Override rate (%) |
| priority | INT | NOT NULL, ≥ 1 | Lower = higher precedence |
| is_active | BOOLEAN | DEFAULT true | Enable/disable |
| extra_conditions | JSON | NULL | **Phase 2 forward compatibility**. Phase 1 always NULL, not validated |
| version | INT | NOT NULL, DEFAULT 0 | Optimistic locking |
| created_at | DATETIME | NOT NULL | |
| updated_at | DATETIME | NOT NULL | |

**New Table — `vcomm_audit_log`** (FR-007 unified):

| Field | Type | Constraint | Description |
|-------|------|-----------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | |
| user_id | BIGINT | NOT NULL | Actor |
| timestamp | DATETIME | NOT NULL | Action time |
| action_type | ENUM | NOT NULL | List in BR-028 (RULE_CREATED, OVERRIDE_MODE_CHANGED, ...) |
| entity_type | ENUM | NOT NULL | 'rule' \| 'contract' \| 'booking' |
| entity_id | BIGINT | NOT NULL | ID of the affected entity |
| contract_id | BIGINT | NOT NULL | For filtering by contract |
| old_value | JSON | NULL | Snapshot before the change |
| new_value | JSON | NULL | Snapshot after the change |
| metadata | JSON | NULL | matched_rule_ids, evaluation_mode, error_code, etc. |
| created_at | DATETIME | NOT NULL | |

Index: `(contract_id, timestamp DESC)`, `(entity_id, entity_type, timestamp DESC)`, `(action_type, timestamp DESC)`

---

### FR-010: Processing Logic

When a booking is created or modified:

| Step | Action | Detail |
|------|--------|--------|
| 0 | Authorization | Validate that the user has permission for the contract_id (BR-040). Cross-contract → 403 + log security event |
| 1 | Check override_mode | If `'none'` → return Base Rate immediately. Stop processing (short-circuit per NFR-015) |
| 2 | Calculate Lead Time | `Lead Time = daysBetween(check_in_date, booking_date)` using DATE-only components at server timezone (UTC/JST fixed). DST-safe (BR-005, BR-022b). Always ≥ 0 |
| 3a | Booking mode | Query `booking_rate_rules` WHERE `contract_id` matches AND `lead_min_days ≤ Lead Time ≤ lead_max_days` AND `is_active = true`. Order by `priority ASC, created_at DESC, id DESC`. Return first match |
| 3b | Stay mode | Query `stay_rate_rules` WHERE `contract_id` matches AND check-in MM-DD is within range `stay_from–stay_to` (handle cross-year + 1-day per BR-015) AND `is_active = true`. Order by `priority ASC, created_at DESC, id DESC`. Return first match |
| 3c | Combined mode | Execute both 3a and 3b (each falls back to Base if no match). Apply `combined_logic`. Additive: enforce `rate_cap` (cap=25% if NULL, BR-035) + floor 0% (BR-017a) |
| 4 | Fallback | If no rule matches → return Base Rate |
| 5 | Lock & Log | Lock rate on booking. Write audit trail to `vcomm_audit_log`: action_type=`RATE_LOCKED_AT_BOOKING_CREATION` (or `RATE_RE_EVALUATED_ON_BOOKING_MODIFY` for update flow), applied rate, matched rule IDs (JSON array), mode, raw vs final value (for Additive), timestamp |
| 6 | Error fallback | If evaluation throws an exception (DB timeout, corrupt rule) → fall back to Base Rate, log E-VCR-016 + alert monitoring (does not block booking creation) |

---

### FR-011: Save Coordination (Drawer Staging + Atomic Modal Commit)

**Description**: Override config is edited in the **OverrideDrawer** (slides in from the parent Vendor Contract modal). To ensure data integrity between the contract main fields + override config + child rules, the system uses a **2-step staging save**.

**Business Rules**:
- BR-041: **Drawer Apply (staging)** — click [Apply Changes] in the drawer:
  - Does NOT POST to the backend immediately
  - Saves drawer state to the parent modal's in-memory staging (React state, Vue store, etc.)
  - Drawer closes → OverrideStatusBadge updates with suffix "Modified, not saved" (yellow)
  - Parent modal [Save] button displays an asterisk: `[Save *]` with tooltip
- BR-042: **Modal Save (atomic flush)** — click [Save] on the parent modal:
  - Validate both contract main fields + override staging
  - 1 transaction POST `/api/vendor-contracts/{id}` with payload:
    ```json
    {
      "contractFields": { ... },
      "override": {
        "mode": "stay",
        "combined_logic": "higher_wins",
        "rate_cap": 25,
        "version": 12
      },
      "booking_rate_rules": [ ... ],
      "stay_rate_rules": [ ... ],
      "version": 7
    }
    ```
  - Backend opens DB transaction: UPDATE vendor_contract + INSERT/UPDATE/DELETE rules + INSERT audit logs
  - On success: clear staging, badge syncs to DB, parent modal closes, success toast
  - On failure: display E-VCR-014 or E-VCR-017 (optimistic lock conflict). Parent modal and staging are retained for user to retry
- BR-043: **Discard handling (smart cancel — skip confirmation when no pending changes)**:
  - **Drawer [Discard Changes]**: detect pending — `hasPendingChanges || mode/toggle/logic/rate_cap changed vs DB` →
    - Has pending → confirm E-VCR-013 → on confirm, **reset the entire drawer state** to the DB initial values (overrideEnabled, stagingMode, combinedLogic, rateCap, bookingList, stayList) + clear localStorage draft + toast "Changes discarded"
    - No pending → close drawer immediately, no confirmation
  - **Drawer [×] / Esc / backdrop click**: triggers the same flow as [Discard Changes]
  - **Parent modal [Cancel] / [×] / Esc / backdrop click** (smart):
    - Has pending changes → confirm E-VCR-013 → on confirm, reset all + close modal (allows user to navigate to sidebar/other page)
    - No pending → close modal immediately, no confirmation (smooth UX, no jankiness when user is merely browsing the modal)
  - **Parent modal Save success**: also auto-closes the modal (returns user to Traders list)
- BR-044: **Auto-save draft**:
  - Drawer staging is auto-saved to `localStorage` every **10 seconds** with key `vcomm_draft_{contract_id}_{user_id}`
  - **Conditional save**: the timer only ticks when `hasPendingChanges = true` (state ≠ DB initial). After a Discard resets state → hasPendingChanges=false → auto-save pauses until the user edits again
  - **Cancel on discard** (interaction with BR-043): when the user clicks [Discard Changes] and confirms → `clearInterval(autoSaveTimerId)` is called BEFORE clearing localStorage to avoid a race condition (timer fires just after clear → writes a stale draft back)
  - When the user reopens the modal on the same contract: detect draft → prompt "Restore pending override changes from {timestamp}?" with 2 buttons [Restore] [Discard draft]
  - **Restore behavior**: the draft contains only **UI state** (mode, toggle, rules list, combined_logic, rate_cap) — **no `version` field**. On restore, `version` is re-fetched fresh from the server via `GET /api/contracts/{id}` to avoid a stale version causing a spurious 409 conflict (see BR-039 + TS-068)
  - **Restore prompt skip**: the prompt is only shown when the draft state ≠ DB initial state. After a Discard reset, if an auto-save tick writes state == DB (due to a bug), the restore prompt is skipped to avoid false positives
  - **Corrupt draft handling**: if `JSON.parse(localStorage.getItem(draftKey))` throws → silently discard the corrupt draft (no prompt shown), clear the localStorage key, and open the drawer fresh. Do not crash the UI
  - **Quota exceeded**: if `localStorage.setItem` throws `QuotaExceededError` → silent fail (do not interrupt user workflow), optional console.warn. Do not retry
  - After the parent modal Save succeeds → clear draft
- BR-045: **Concurrent edit detection (BR-039)**: the version in the frontend payload must match `vendor_contract.version` on the backend; mismatch → 409 Conflict + E-VCR-017

**Acceptance Criteria**:
- [ ] AC-030: Drawer [Apply] does not create a network request to the rate-rules API
- [ ] AC-031: Parent modal Save triggers exactly 1 API call with the merged payload
- [ ] AC-032: Closing the browser tab with pending changes → reloading → restore prompt is displayed
- [ ] AC-033: Optimistic lock conflict (2 tabs editing the same contract) → the second user sees a conflict dialog with Reload/Override options
- [ ] AC-034: Parent modal [Save] disabled when there are no changes; enabled when there is override staging or contract field changes

---

### FR-012: Cross-Feature Display Synchronization

**Description**: When a user changes the Base Rate (V.Comm Rate field) or the override config (mode/rules/logic), existing ELLIS screens that display pricing/commission-related data must reflect the update. In particular, **existing bookings** retain the rate locked at the time of creation (regression — BR-025).

**Affected Screens** (existing ELLIS — not new feature screens):
- Vendor Contract edit form (where the override drawer is opened from — already covered by BR-004a)
- **Rate & Allotment Collection list** — header strip displaying `Comm Rate: X%` + new indicator
- **Rate & Allotment Plan & Promotion modals** — pricing recalculation
- **Rate Detail Gross tab** — Vendor Net, Contract Amount, Margin Rate, Margin Amount columns per row
- **Rate Detail "Apply All" action** — bulk recalculation for future rows
- **Reservation Hotel Price Detail (Vendor section)** — regression: rate is locked, does not change

**Business Rules**:

- BR-046: **CommRateHeaderIndicator** — a new component on the header strip of the Rate & Allotment Collection list, placed after the "Comm Rate: X%" label:
  - **Header strip layout** (per existing ELLIS — split into 2 cells to align with the grid below):
    - **Left cell** (fixed width 260px = sticky room column in the grid): "Dynamic Rate : {ID}" — white bold text, padding 4
    - **Right cell** (flex-1, subtle border-left): metadata cluster `Contract Status: A | Curr: VND | Gross | Comm Rate: X% ⓘ | Rate: HotelLink | Allotment: HotelLink` — labels white/85%, values coral `#FF8A65` bold
    - **Right end**: "Stop or Start Selling" link in coral underline (action only, not part of the metadata cluster)
    - Background: `#3A3A3A` (dark, ELLIS pattern)
  - **Visibility**: ⓘ icon is displayed ONLY when `override_mode !== 'none'`. When `override_mode = 'none'`, the header renders only "Comm Rate: X%" (no icon)
  - **Trigger**: click ⓘ → opens a Popover anchored to the icon (380px width, max 480px height)
  - **Popover sections** (3 fixed sections, **no action buttons** — purely informational):
    1. **BASE RATE**: current value (15%) + "Last changed: {date} by {user} (was {prev}%)" — the "Last changed" line only renders if the change occurred within the past 30 days
    2. **OVERRIDE STATUS**: "✓ Active — {MODE} mode ({N} active rules)" with **mode-color check icon** (booking-primary indigo, stay-primary orange, combined-primary green depending on mode). Combined mode: adds "Combined logic: {logic}". Additive logic: adds "Rate cap: {cap}%".
       - **Currently applying sub-section** (new — sub-card below the mode line, with a border-t separator): describes the rule currently in effect if a user were to create a booking right now:
         - Compute simulation: `Booking Date = today (server time)`, `Stay Date = today + 30 days` (default sim per BR-022a)
         - Header: "Currently applying (today {MM-DD}, +30d sim):"
         - **Booking mode**: displays 1 row with BOOKING chip + matched tier name + period (D-X~Y) + rate% + delta vs base
         - **Stay mode**: displays 1 row with STAY chip + matched season name + period (MM-DD → MM-DD) + rate% + delta
         - **Combined mode**: displays 2 rows (BOOKING + STAY) + 1 line "→ {Logic} applied: {finalRate}%" per BR-016 logic resolution
         - **No match**: displays "No rule matches → using Base Rate {X}%" (rule fallback)
         - Purpose: SCM Manager can glance-check the currently effective rule without opening the Override Drawer
    3. **ACTIVE RULES (N)**: scrollable list, sorted `priority ASC`. Each row: priority badge `P{N}` + **mode label chip** (uppercase 9px, bordered, bg `mode-color +22 alpha`, border `mode-color +55 alpha`, text `mode-color`) + rule name + period sub-line (`D-X~Y` for booking, `MM-DD → MM-DD` for stay) + rate% + delta badge.
       - **Chip values**: only "BOOKING" (indigo `#6366F1`) or "STAY" (orange `#FF6000`) — **NO "COMBINED" chip** (each rule is individually always one of the two types). In Combined mode, the list mixes rules with BOOKING + STAY chips.
       - **Delta badge convention**: **rate-up green** (increase = positive for OMH revenue), **rate-down red** (decrease = negative). Convention is the reverse of the initial version to be intuitive: increase = good = green, decrease = bad = red.
       - **Chip a11y**: `aria-label="Mode: BOOKING"` or `"Mode: STAY"` for screen readers (NFR-009)
       - **Sub-feature — Inactive rules toggle**: hidden by default; a "Show inactive (N)" toggle within the same section 3 (NOT a separate section) — click to expand inactive rules with opacity-60 + line-through
  - **Close**: × button + Esc + click outside
  - **z-index**: higher than header strip but lower than modal/drawer

- BR-047: **Rate & Allotment Collection list** header — the `Comm Rate: X%` label is computed from `vendor_contract.base_rate` of the contract currently being viewed. When the user saves a Base Rate change in the Vendor Contract modal → the list refreshes to show the new value on the next page view (no realtime websocket needed — see OQ-004)

- BR-048: **Rate Detail Gross tab** — the Vendor Net, Contract Amount, Margin Rate (%), and Margin Amount columns per row are computed from:
  - Sell Rate × applicable V.Comm Rate
  - Applicable rate = engine result (FR-010) for the row's stay_date / booking_date
  - When Base Rate or rules change → list needs a display refresh (on next view)

- BR-049: **"Apply All" action** in Rate Detail — after the user saves a Base Rate change, clicking "Apply All" triggers a bulk recompute for future rows (not yet booked). This is a user-initiated action, not automatic. Existing bookings are not affected (BR-025)

- BR-050: **Reservation Hotel Price Detail regression** (BR-025 reinforce):
  - The existing booking rate is locked at booking creation and a snapshot is stored in `bookings.contractList`
  - When Base Rate or override rules change → the Reservation Hotel Price Detail for old bookings does NOT change
  - Changes only occur via manual recalculate (FR-006 BR-026) or booking modify (BR-024)

**Acceptance Criteria**:
- [ ] AC-035: ⓘ icon hidden when `override_mode = 'none'`, visible when != 'none'
- [ ] AC-036: Click ⓘ → Popover renders with 3 sections (BASE RATE / OVERRIDE STATUS / ACTIVE RULES)
- [ ] AC-037: Active Rules list is scrollable when N > 5; no pagination needed (max ~50 rules per BR-007)
- [ ] AC-038: "Show inactive (N)" toggle displays inactive rules within the same list
- [ ] AC-039: Popover closes via × / Esc / click outside; no navigation buttons (informational only)
- [ ] AC-040: Base Rate change save → Rate & Allotment Collection list label updates on next view
- [ ] AC-041: Base Rate change save → Rate Detail Gross tab Vendor Net/Margin recomputes on next view
- [ ] AC-042: Existing booking in Reservation Hotel Price Detail does NOT change after Base Rate / rules change

---

## Spec Files

| File | Contents |
|------|----------|
| `screens.md` | Screen Definitions, Error Handling |
| `test-scenarios.md` | Non-Functional Requirements, Test Scenarios |

---

## 8. Open Questions

| ID | Question | Context | Status |
|----|----------|---------|--------|
| OQ-001 | Booking Engine API: create a new endpoint or extend the existing API? Fallback behavior when API errors? | Architecture integration decision required | TBD |
| OQ-002 | Migration strategy for ALTER TABLE vendor_contract in production? | The table may have many records; impact assessment needed | TBD |
| OQ-003 | Rule creation UX: Modal dialog, inline row editing, or side panel? | Affects UX flow. Resolved — side panel within the drawer (BR-004a) | RESOLVED |
| OQ-004 | Cross-feature display refresh strategy: realtime websocket vs reload-on-next-view? | FR-012 BR-047/048 — affects latency UX when user toggles between Rate & Allotment and the Vendor Contract modal | TBD — recommendation: reload-on-next-view (Phase 1) for simplicity; defer websocket to Phase 2 |

---

## 9. Review History

| Round | Planner Score | Tester Score | Key Decisions | Date |
|-------|---------------|--------------|---------------|------|
| — | — | — | Draft created from DOCX spec + analyst requirements gathering | 2026-04-16 |
| Update | — | — | Synced with DOCX v1.0 + prototype v2: added Phase 2 Roadmap, Industry Benchmark, Use Case columns, Cross-Reference Matrix, prototype color tokens & components | 2026-05-05 |
| 1 | 8/10 | 5/10 | Reviews identified 7 critical + 14 major + 11 minor/suggestions issues | 2026-05-05 |
| 1 (resolved) | — | — | Applied all review fixes: FR-011 Save Coordination (Option B drawer), FR-005 client-side simulation + shared evaluator, BR-005/022b DST-safe Lead Time, BR-009 tie-break created_at DESC only, BR-015a Feb 29 leap year handling, BR-017a Additive floor 0%, BR-017b rate_cap persistence, BR-022a full default state, BR-024 re-evaluate scope explicit, BR-026 batch recalculate scope, BR-028 unified audit log, BR-035a rate_cap warnings, BR-038 rule_name validation, BR-039 optimistic locking, BR-040 cross-contract auth, BR-041~045 save coordination flow. AnnualCalendarGrid topRule = priority-based (sync with engine). Cross-Reference Matrix expanded to all 5 logics. New components: OverrideStatusBadge, OverrideDrawer, AllInactiveBanner. New error codes E-VCR-013~020. New NFRs 017~022. 24 new test scenarios TS-035 to TS-058 | 2026-05-05 |
| Sync | — | — | Prototype iteration sync (post-finalize): (1) Color convention SWAP — rate-up = green (#16A34A) increase positive, rate-down = red (#DC2626) decrease negative — applied across spec/DSL/design-tokens/DESIGN.md/shadcn-mapping. (2) BR-018b new — Combined mode Add Rule form: 2 sections stacked (Booking Tier + Stay Season) visible simultaneously, click-to-activate, AC-017a added. (3) FR-011 BR-043 clarified — smart cancel: skip confirm dialog when no pending changes (drawer + modal both); Drawer Discard resets state to DB initial. (4) BR-046 expanded — header strip 260px room-col + flex-1 metadata layout aligned with date columns; mode label chip (BOOKING/STAY uppercase bordered) replaces mode color dot in ActiveRulesList. (5) RuleForm — added X close button + state reset on Cancel/X/Esc. Files synced: vi/en/ko spec + DSL booking-recalculate + design-tokens.json + DESIGN.md + shadcn-mapping.json | 2026-05-06 |
| Sync | — | — | Post-finalize incremental sync: BR-046 FR-012 enhancement — added "Currently applying sub-section" to OVERRIDE STATUS popover section. Compute simulation using today (server time) + 30d stay date per BR-022a. Per-mode display (Booking/Stay/Combined with logic resolution). No-match fallback "No rule matches → using Base Rate {X}%". New test scenario TS-073 (6 sub-cases). screens.md OverrideStatusSection + CommRateHeaderIndicator ASCII updated. | 2026-05-07 |

---

## 10. Phase 2 Roadmap

The following enhancements are planned for the next phase. The Phase 1 architecture must ensure forward compatibility for these features.

| Feature | Description | Priority |
|---------|-------------|----------|
| Day-of-Week Modifier | Apply additional weighting for weekends vs weekdays (e.g., Sat/Sun +2%) | Medium |
| Room Type Override | Allow differentiating commission by room type (e.g., suite vs standard) | Medium |
| Bulk CSV Import/Export | Batch upload/download season rules for multiple contracts via CSV file | High |
| HARE AI Recommendation | HARE market intelligence suggests optimal commission tiers based on real-time demand data and competitive analysis | High |
| Simulation Report Export | Export simulation results comparing 3 modes for a date range, as a shareable report | Low |
| Rate Parity Integration | Auto-flag when commission override causes a parity violation with competing channels | High |

**Forward Compatibility Notes**:
- `booking_rate_rules` and `stay_rate_rules` must have a nullable JSON column `extra_conditions` so that Phase 2 can add room_type, day_of_week without complex migrations
- Audit trail schema must be flexible enough for new action types (RECOMMENDATION_APPLIED, BULK_IMPORT, PARITY_FLAG)
- API contract for rate evaluation needs an extension point for HARE recommendation hooks

---

## 11. Global Industry Benchmark

This feature aligns with practices widely adopted across global OTA platforms. **Source**: SCM team competitive analysis Q1/2026 + public documentation of the platforms.

| Capability | Industry Standard | Reference / Example | OMH Implementation |
|-----------|-------------------|---------------------|--------------------|
| Stay Date Tiers | 100% adoption — all major platforms support seasonal commission differentiation | Booking.com Genius seasonal rates, Expedia VIP Access, Agoda Smart Insights | Mode 2 (Stay). Full season definition with MM-DD recurring ranges |
| Lead Time Tiers | Advanced feature — only a few platforms offer lead-time incentives | Booking.com Early Booker Deals, Hotelbeds Advance Purchase | Mode 1 (Booking). 5-tier structure from Early Bird to Urgent |
| Combined Logic | Rare — only enterprise-grade platforms have dual-condition evaluation | Hotelbeds Dynamic Margin (custom contracts), enterprise CMS integrations | Mode 3 (Combined). 5 resolution logics. **Differentiator for OMH** |
| Priority Resolution | Standard — all platforms use priority ranking for overlapping rules | Universal pattern (no specific reference) | Integer priority per rule. Lower = higher precedence |
| Base + Override | Universal — single base rate with exception overrides | All major OTAs | Existing V.Comm Rate serves as base. Override rules layered on top |
| Live Simulation | Uncommon — most platforms require a save-then-test workflow | Hotelbeds Rate Simulator (limited), most others save-and-test | Built-in client-side simulation panel. Immediate rate verification before save |

**Note**: Industry claims are based on an internal review at 2026-Q1. Update when new competitive intelligence is available (every 6 months).
