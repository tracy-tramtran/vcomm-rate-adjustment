[vcomm-rate-adjustment-spec.md](https://github.com/user-attachments/files/27433126/vcomm-rate-adjustment-spec.md)
# vcomm-rate-adjustment# Period-Based V.Comm Rate Adjustment — Functional Specification

> **Status**: FINALIZED
> **Author**: Planning Plugin (Auto-generated)
> **Created**: 2026-04-16T10:00:00Z
> **Last Updated**: 2026-05-05T11:15:00Z
> **Source Documents**:
> - `ELLIS VComm Rate Period Adjustment Spec v1.0` (DOCX, April 2026)
> - Prototype simulator v2 (`remixed-ca0f0e7d.tsx`) — interactive 3-mode UX reference
> **Module**: Traders > Vendor Contract > Dynamic Rate > Net Provide
> **Subtitle**: Dynamic Commission Override by Booking Period & Stay Season

---

## 1. Overview

### 1.1 Purpose

Hệ thống ELLIS hiện tại chỉ hỗ trợ **một mức V.Comm Rate cố định** cho mỗi vendor contract (VD: 15%), bất kể thời điểm đặt phòng hay mùa lưu trú. Điều này gây ra:

- Không thể tối ưu margin theo mùa cao/thấp điểm (Golden Week, cherry blossom, year-end)
- Không có cơ chế khuyến khích đặt sớm hoặc thu premium cho booking last-minute
- Đội SCM phải tạo contract riêng hoặc yêu cầu override thủ công, tăng rủi ro sai sót
- Bất lợi cạnh tranh so với các OTA toàn cầu đều hỗ trợ commission theo kỳ

Tính năng **Period-Based V.Comm Rate Adjustment** cho phép override commission rate theo 3 chế độ: Booking Date (lead time), Stay Date (mùa), hoặc Combined (kết hợp cả hai), được cấu hình riêng cho từng vendor contract.

### 1.2 Target Users

| Vai trò | Mô tả |
|---------|-------|
| SCM Manager | Cấu hình override rules cho vendor contract, quản lý mùa và lead time tiers |
| SCM Staff | Xem cấu hình, sử dụng simulation panel để kiểm tra rate |
| Admin (tất cả) | Toàn quyền truy cập tính năng — không phân quyền riêng |

### 1.3 Success Metrics

| KPI | Mục tiêu |
|-----|----------|
| Giảm thời gian cấu hình commission | Giảm 50% so với tạo contract riêng thủ công |
| Tỷ lệ contract sử dụng override | ≥ 30% contract active trong 3 tháng đầu |
| Margin improvement mùa cao điểm | +2–5% commission margin so với flat rate |
| Zero disruption | 100% contract hiện tại hoạt động bình thường sau deploy |

---

## 2. User Stories

| ID | Role | Goal | Priority |
|----|------|------|----------|
| US-001 | As a SCM Manager | Tôi muốn bật chế độ override commission theo lead time cho vendor contract để khuyến khích booking sớm với rate thấp hơn và thu premium cho last-minute | P0 |
| US-002 | As a SCM Manager | Tôi muốn cấu hình commission theo mùa (stay date) để tối ưu margin vào mùa cao điểm (Golden Week, Year-End) | P0 |
| US-003 | As a SCM Manager | Tôi muốn kết hợp cả booking và stay conditions với logic giải quyết (Higher Wins, Lower Wins, v.v.) để kiểm soát granular nhất | P0 |
| US-004 | As a SCM Staff | Tôi muốn dùng simulation panel để kiểm tra rate kết quả trước khi lưu cấu hình | P0 |
| US-005 | As a SCM Manager | Tôi muốn xem lịch sử thay đổi rules (audit trail) để theo dõi ai đã thay đổi gì và khi nào | P1 |
| US-006 | As a SCM Manager | Tôi muốn recalculate V.Comm Rate thủ công cho booking đã tồn tại khi rules thay đổi | P1 |
| US-007 | As a SCM Manager | Tôi muốn tạo mùa qua năm (VD: Dec 15 – Jan 15) mà không cần tách thành 2 rules riêng | P1 |
| US-008 | As a Admin | Tôi muốn hệ thống fallback về Base Rate khi không có rule nào match để đảm bảo mọi booking đều có commission rate | P0 |

---

## 3. Functional Requirements

### FR-001: Override Mode Activation

**Description**: Mỗi vendor contract có một toggle "Period-Based Override" cho phép bật/tắt chế độ override commission. Khi tắt, V.Comm Rate hoạt động như hiện tại (single flat rate).

**Business Rules**:
- BR-001: Override mode mặc định là `'none'` (tắt) cho tất cả contract hiện tại và mới
- BR-002: Khi bật override, admin phải chọn mode: `'booking'`, `'stay'`, hoặc `'combined'`
- BR-003: Chỉ 1 mode active tại một thời điểm cho mỗi contract
- BR-004: Khi tắt override (chuyển về `'none'`), các rules **vẫn được giữ** trong database — không bị xóa
- BR-004a: **UI placement** (chốt theo screenshots ELLIS hiện tại — Option B side drawer):
  - Trong modal `Company Master > Vendor Contract > Dynamic Rate sub-tab`: dòng Net Provide có sub-card "Period-Based Override" với badge status + button [Configure/Edit Override]
  - Click button → mở Override Drawer (slide-in từ phải, 70% viewport, max 1200px)
  - Drawer chứa toàn bộ flow: ToggleOverride, Mode Selector, Rule Table, Visual Map, Calendar, Cross-Reference Matrix, Live Simulator, Audit Summary
  - Save coordination atomic — xem FR-011

**Acceptance Criteria**:
- [ ] AC-001: Sub-card "Period-Based Override" hiển thị trong modal cha (modal anchor) ngay cạnh V.Comm Rate field
- [ ] AC-002: Khi Override OFF → badge "OFF — single flat rate" + button "[⚙ Configure]"
- [ ] AC-003: Khi Override ON → badge "✓ ON | Mode: {X} | {N} active rules" + buttons [Edit] [History]
- [ ] AC-003a: Drawer slide-in animation ≤300ms (NFR-018), modal cha trở thành inert khi drawer mở

---

### FR-002: Booking Date Mode (Lead Time Tiers)

**Description**: Commission rate được xác định bởi số ngày giữa ngày tạo booking và ngày check-in (Lead Time).

**Business Rules**:
- BR-005: **Lead Time formula (DST-safe)**: `Lead Time = DATE_DIFF(check_in_date, booking_date)` tính bằng calendar days dựa trên **DATE-only components** (year/month/day) ở **server timezone fixed (UTC hoặc JST, project-wide constant)**. **CẤM dùng millisecond arithmetic** (`Math.round((d1-d2)/86400000)`) — sẽ desync ở DST boundaries. Frontend simulator phải dùng cùng helper với backend (shared library)
- BR-006: Admin tạo các tier với `lead_min_days` (inclusive) và `lead_max_days` (inclusive, NULL = unlimited)
- BR-007: Mỗi tier có `priority` (integer, lower = higher precedence), `comm_rate` (0–50%), `is_active` toggle
- BR-008: Khi nhiều tiers match, tier có **priority thấp nhất** được áp dụng
- BR-009: **Tie-break (chọn 1 tiêu chí duy nhất)**: nếu 2 tiers cùng priority đều match → ưu tiên `created_at DESC` (rule tạo sau thắng). Secondary tie-break: `id DESC` (chỉ khi created_at trùng đến giây). Loại bỏ ambiguity giữa "ID lớn hơn HOẶC created_at mới hơn" của bản trước
- BR-010: Nếu không có tier nào match (booking rơi vào gap giữa các tiers), **Base Rate** được sử dụng
- BR-010a: **Boundary semantics (inclusive cả 2 đầu)**: Lead=`lead_min_days` → match. Lead=`lead_max_days` → match. Lead=`lead_max_days+1` → không match (xét tier tiếp theo)

**Acceptance Criteria**:
- [ ] AC-004: Hiển thị rule table với cột: Priority, Tier Name, Lead Time Range, V.Comm Rate, Active, Actions
- [ ] AC-005: CRUD đầy đủ cho booking rate rules
- [ ] AC-006: Cảnh báo khi phát hiện gap giữa các lead time tiers
- [ ] AC-007: Cảnh báo khi phát hiện duplicate priority

**Default Tiers (Gợi ý)**:
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

**Description**: Commission rate được xác định bởi mùa/kỳ mà ngày check-in rơi vào. Mùa được định nghĩa bằng MM-DD ranges, lặp lại hàng năm.

**Business Rules**:
- BR-011: Mùa được lưu dạng `stay_from` (MM-DD) và `stay_to` (MM-DD), recurring annually
- BR-012: **Hỗ trợ cross-year season**: khi `stay_from > stay_to` (VD: `12-15` đến `01-15`), hệ thống hiểu đây là mùa qua năm
- BR-012a: **1-day season**: khi `stay_from == stay_to` (VD: `05-03` ~ `05-03`) → hiểu là season đúng 1 ngày, KHÔNG phải cross-year. Match khi check-in MM-DD == stay_from
- BR-013: Overlap giữa các mùa **được phép** — priority field giải quyết conflict (lowest priority number = highest precedence). Hỗ trợ N≥2 seasons overlap (3+ seasons cùng date phải sort `priority ASC, created_at DESC` và lấy first)
- BR-014: Mỗi mùa có `season_name` (user-defined label, max 100 ký tự, hỗ trợ Unicode), `priority`, `comm_rate`, `is_active`
- BR-015: **Logic match (so sánh lexicographical trên chuỗi MM-DD 5 ký tự)**: check-in MM-DD nằm trong range `stay_from–stay_to`. **Trường hợp**:
  - Normal (`stay_from < stay_to`): match khi `stay_from <= MM-DD <= stay_to` (cả 2 đầu inclusive)
  - Cross-year (`stay_from > stay_to`): match khi `MM-DD >= stay_from` HOẶC `MM-DD <= stay_to`
  - 1-day (`stay_from == stay_to`): match khi `MM-DD == stay_from`
- BR-015a: **Leap year handling (Feb 29)**:
  - Check-in = `02-29` (leap year) trong season Off-Peak (`01-01`~`03-31`) → match (vì `02-29` lexicographically nằm giữa `01-01` và `03-31`)
  - `stay_to = 02-29` ở năm không nhuận: vẫn lưu được trong DB; logic match dùng so sánh chuỗi nên `02-28 <= 02-29` (true) → check-in 02-28 năm không nhuận match. Năm nhuận: 02-29 == stay_to → match
  - `stay_from = 02-29` cho 1-day season: chỉ match đúng năm nhuận (date 02-29 không tồn tại ở năm thường nên check-in `02-29` không thể xảy ra ở năm thường)
  - Validate UI: cảnh báo (không block) khi admin nhập `stay_from` hoặc `stay_to` = `02-29`: "Ngày 02-29 chỉ tồn tại trong năm nhuận. Quy tắc match có thể khác mong đợi."

**Acceptance Criteria**:
- [ ] AC-008: Hiển thị season rule table với cột: Priority, Season Name, Stay Period, V.Comm Rate, Active, Actions
- [ ] AC-009: CRUD đầy đủ cho stay rate rules
- [ ] AC-010: Hỗ trợ nhập cross-year period (VD: Dec 15 – Jan 15)
- [ ] AC-011: Cảnh báo khi phát hiện overlap periods (informational, không block save)
- [ ] AC-012: Annual calendar view hiển thị visual map các mùa (tham khảo prototype TSX)

**Default Seasons (Gợi ý — JP Context)**:
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

**Description**: Cả hai điều kiện Booking và Stay được evaluate độc lập, tạo ra 2 candidate rates. Một resolution logic xác định rate cuối cùng.

**Business Rules**:
- BR-016: Admin chọn 1 trong 5 resolution logics khi bật Combined mode:

| Logic | Formula | Mô tả | Recommended For |
|-------|---------|-------|-----------------|
| Higher Wins (Recommended) | `MAX(B, S)` | Lấy rate cao hơn — tối đa revenue cho OMH | Default setting. Revenue-maximizing contracts |
| Lower Wins | `MIN(B, S)` | Lấy rate thấp hơn — hotel-friendly, xây dựng partnership | Strategic partners, volume-based contracts |
| Stay Priority | `S → B fallback` | Stay rate ưu tiên; Booking rate chỉ khi stay = Base Rate | Hotels with strong seasonal pricing structures |
| Booking Priority | `B → S fallback` | Booking rate ưu tiên; Stay rate chỉ khi booking = Base Rate | Hotels focused on yield management |
| Additive | `Base ± ΔB ± ΔS` | Cộng dồn delta từ cả 2 — cần rate_cap | Advanced contracts requiring granular margin control |

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

**Use Case**: Premium partners requiring granular control, high-volume contracts. CrossRefMatrix component trong drawer interactive — khi admin đổi LogicCard, tất cả Final Rate cells recalculate **trong ≤300ms** (xem NFR-017).

- BR-017: **Additive mode**: `Final = BASE_RATE + (BookingRate - BASE_RATE) + (StayRate - BASE_RATE)`. Nếu `rate_cap` không được set, áp dụng system default cap = 25%
- BR-017a: **Additive floor & cap rules**:
  - **Cap (upper bound)**: `Final = min(Final, rate_cap)`. Nếu `Final > rate_cap` → trigger E-VCR-011 informational warning
  - **Floor (lower bound)**: `Final = max(Final, 0%)` — không cho phép commission rate âm. Audit log ghi cả raw value (trước floor) và final value
  - Trường hợp `rate_cap < BASE_RATE` → trigger E-VCR-019 cảnh báo trước khi save (soft warning, cho phép save)
- BR-017b: **rate_cap persistence**: `rate_cap` được lưu trong `vendor_contract` independent với `combined_logic`. Khi user đổi từ Additive sang logic khác, `rate_cap` giữ nguyên trong DB. Khi quay lại Additive, RateCapInput pre-fill giá trị cũ
- BR-018: Nếu một trong hai mode không match rule nào, dùng Base Rate cho mode đó trước khi apply resolution logic
- BR-018a: **Cả 2 mode không match (cả Booking và Stay đều fallback Base Rate)**: tất cả 5 logics trả về Base Rate (verified):
  - `Higher Wins`: MAX(Base, Base) = Base
  - `Lower Wins`: MIN(Base, Base) = Base
  - `Stay Priority`: Stay = Base → fallback Booking = Base → Base
  - `Booking Priority`: Booking = Base → fallback Stay = Base → Base
  - `Additive`: Base + 0 + 0 = Base

**Acceptance Criteria**:
- [ ] AC-013: Hiển thị 5 resolution logic options dạng selectable cards/buttons
- [ ] AC-014: "Higher Wins" được đánh dấu "Recommended"
- [ ] AC-015: Cross-reference matrix hiển thị kết quả cho các tổ hợp stay × booking phổ biến
- [ ] AC-016: Khi chọn "Additive", hiển thị thêm field `rate_cap` input
- [ ] AC-017: Cả 5 logics đều nằm trong Phase 1

---

### FR-005: Live Simulation Panel

**Description**: Panel simulation cho phép admin nhập Booking Date và Stay Date, hiển thị kết quả commission rate cho cả 3 mode song song.

**Business Rules**:
- BR-019: Simulation chạy **real-time** khi user thay đổi input (debounce 300ms — NFR-004)
- BR-020: Hiển thị: Lead Time (D-XX), matched rule name cho từng mode, final rate, so sánh vs Base Rate
- BR-021: Simulation **không lưu dữ liệu** — chỉ để preview/verify trước khi save cấu hình
- BR-022: Response time target: **dưới 300ms** cho debounced re-compute (lower than user-perceptible threshold)
- BR-022a: **Default initial state khi mở drawer lần đầu cho contract đã có override active**:
  - Active mode: từ `vendor_contract.override_mode` (DB)
  - Booking Date: **today** (server time, formatted YYYY-MM-DD)
  - Stay Date: **today + 30 days** (default lead time = 30 → match "Standard" tier với default tiers)
  - Lead Time: D-30 → tier "Standard" trong default tiers
  - Combined Logic: từ `vendor_contract.combined_logic` hoặc `higher_wins` nếu NULL
  - **Full default expected results** (verify được trong test):
    | Mode | Matched Rule | Rate | Note |
    |------|--------------|------|------|
    | Booking-Based | Standard (D-30~59) | 13% | Lead=30 nằm boundary inclusive |
    | Stay-Based | Tùy stay_date thực tế (today+30) | tùy season | VD: today=2026-05-05 → stay=2026-06-04 → match "Shoulder" 14% |
    | Combined (higher_wins) | MAX(13, stayRate) | tùy | VD: MAX(13,14)=14% |
  - Pre-condition: contract đã có default seasons + tiers seeded (xem FR-001 BR-004a)
- BR-022b: **Lead Time formula DST-safe (xem BR-005)**: dùng helper `daysBetween(checkInDate, bookingDate)` shared giữa frontend/backend. Cấm Math.round milliseconds. Frontend simulator import từ `@ellis/rate-engine-shared` (npm package hoặc git submodule)
- BR-022c: **Simulation architecture quyết định = CLIENT-SIDE với shared evaluator**:
  - Simulation chạy thuần client-side (no API call) → response < 50ms typical, không có timeout
  - Sử dụng cùng evaluator code với backend (shared library `@ellis/rate-engine-shared`) để đảm bảo parity 100%
  - Unit test chạy chéo: cùng input → frontend evaluator output == backend evaluator output (parity test required)
  - **E-VCR-010 không còn là "API timeout"** mà chuyển thành "Calculation error" cho lỗi logic local
- BR-022d: **Pre-condition seed data** (BR-022a tham chiếu): Khi user lần đầu bật override toggle, hiện CTA "Apply default tiers/seasons?" để admin có quick start. Mặc định KHÔNG seed — admin phải tạo rule thủ công

**Acceptance Criteria**:
- [ ] AC-018: 3 ô kết quả song song: Booking-Based, Stay-Based, Combined
- [ ] AC-019: Mỗi ô hiển thị: mode name, matched rule, rate %, lead time info
- [ ] AC-020: Khi thay đổi date input → kết quả cập nhật trong vòng **300ms** (post-debounce)
- [ ] AC-020a: Hiển thị công thức Lead Time tham khảo gần input dates: "Lead Time = Check-in − Booking (calendar days)"
- [ ] AC-020b: Active mode tab tương ứng được highlight visual ở result card (border + background tint theo color token)
- [ ] AC-020c: Frontend evaluator import từ `@ellis/rate-engine-shared` cùng backend; parity test pass 100%

---

### FR-006: Rate Locking & Recalculation

**Description**: V.Comm Rate được **khóa** tại thời điểm booking được tạo. Khi rules thay đổi, booking hiện tại **không tự động cập nhật**. Admin có thể recalculate thủ công.

**Business Rules**:
- BR-023: Khi booking được **tạo mới**: evaluate rules tại thời điểm đó → khóa rate → ghi audit log
- BR-024: **Re-evaluate triggers** (auto): chỉ trigger khi 1 trong các trường thay đổi ảnh hưởng rate evaluation:
  - `check_in_date` thay đổi → re-evaluate (Lead Time + season match đều có thể thay đổi)
  - `booking_date` thay đổi (admin sửa trực tiếp) → re-evaluate
  - **KHÔNG re-evaluate** khi: chỉ `check_out_date` thay đổi (Phase 1), guest count thay đổi, room type thay đổi (Phase 1 — Phase 2 có thể), payment status thay đổi
  - Mỗi lần re-evaluate đều ghi audit log với reason
- BR-025: Khi rules thay đổi (admin edit rules): booking hiện tại **giữ nguyên** rate đã khóa — chỉ áp dụng cho bookings mới hoặc khi admin trigger recalculate thủ công
- BR-026: Admin có thể **recalculate thủ công** — chọn booking(s) và trigger re-evaluation với rules hiện tại. Hỗ trợ:
  - **Single**: từ Booking Detail page, button "[🔄 Recalculate V.Comm Rate]"
  - **Batch**: từ Booking List page, multi-select + toolbar action "[🔄 Recalculate Selected (N)]"
  - **Batch limits**: max 500 bookings per batch (Phase 1). Vượt → error E-VCR-021. Phase 2 sẽ tăng lên 5000 với background job
- BR-027: Cancel & rebook: booking mới được treat như booking mới hoàn toàn → lead time tính theo ngày booking mới

**Acceptance Criteria**:
- [ ] AC-021: Booking audit trail ghi rõ: applied rate, matched rule ID(s), mode, timestamp, evaluation source (creation/modify/manual_recalculate)
- [ ] AC-022: Nút "Recalculate" khả dụng cho cả single (Booking Detail) và batch (Booking List multi-select)
- [ ] AC-023: Recalculate hiển thị preview (rate cũ → rate mới) trước khi confirm; batch: bảng tóm tắt với pagination
- [ ] AC-023a: Batch recalculate có handling partial failure: nếu N/M bookings fail, hiển thị danh sách lỗi cụ thể, vẫn commit M-N succeed

---

### FR-007: Audit Trail

**Description**: Mọi thay đổi cấu hình override rules được ghi lịch sử và có UI để xem.

**Business Rules**:
- BR-028: **Unified audit log** — 1 stream duy nhất `vcomm_audit_log` table với các action types:
  - `RULE_CREATED`, `RULE_UPDATED`, `RULE_DELETED`, `RULE_TOGGLED_ACTIVE`, `RULE_TOGGLED_INACTIVE`
  - `OVERRIDE_MODE_CHANGED` (none → booking/stay/combined hoặc ngược lại)
  - `COMBINED_LOGIC_CHANGED`, `RATE_CAP_CHANGED`
  - `RATE_LOCKED_AT_BOOKING_CREATION` (mỗi booking mới)
  - `RATE_RE_EVALUATED_ON_BOOKING_MODIFY`
  - `RATE_RECALCULATED_MANUAL_SINGLE`, `RATE_RECALCULATED_MANUAL_BATCH`
  - `EVALUATION_FALLBACK_BASE_RATE` (khi không match rule)
  - `EVALUATION_ERROR` (E-VCR-016 trigger)
- BR-029: Mỗi log entry: `user_id`, `timestamp`, `action_type`, `entity_type` (rule/contract/booking), `entity_id`, `old_value` (JSON), `new_value` (JSON), `metadata` (JSON: matched_rule_ids, evaluation_mode, etc.)
- BR-029a: Audit log retention: 24 tháng online + archive sau 24 tháng (cold storage). Phase 1 chỉ implement online.

**Acceptance Criteria**:
- [ ] AC-024: **Inline summary** trong Override Drawer (hiển thị 5 thay đổi gần nhất, scope = override config + rules của contract)
- [ ] AC-025: **Tab "V.Comm Override"** trong History popup (chi tiết đầy đủ, filterable theo all action types ở trên)
- [ ] AC-025a: **Booking-level audit** truy cập từ Booking Detail page → "View V.Comm Audit Trail" — filter logs với entity_type=booking, entity_id=current booking
- [ ] AC-026: Cả hai view đều nằm trong Phase 1
- [ ] AC-026a: ActionTypeFilter trong History popup cho phép filter theo từng action type list ở BR-028

---

### FR-008: Validation Rules

**Description**: Các ràng buộc dữ liệu đảm bảo tính nhất quán và an toàn.

**Business Rules**:
- BR-030: Commission rate: 0.00% – 50.00% (DECIMAL(5,2), 2 chữ số thập phân). Reject giá trị ngoài range. Cho phép 0.00% (no commission) với optional business warning
- BR-031: Priority: integer ≥ 1. Cảnh báo (soft) khi duplicate priority trong cùng contract/mode — cho phép save sau confirmation
- BR-032: Period overlap: cho phép, priority giải quyết conflict. Hiển thị warning khi detect overlap (đặc biệt 3+ seasons cùng overlap → message liệt kê toàn bộ)
- BR-033: Lead time gap: cảnh báo khi có gap (VD: D-30~59 và D-61+, thiếu D-60). Booking rơi vào gap dùng Base Rate
- BR-034: Cross-year season: hỗ trợ khi `stay_from > stay_to` (xem BR-015)
- BR-035: Additive mode cap: nếu `rate_cap` = NULL → system default = 25%. Range cho `rate_cap`: 0.00% – 50.00% (cùng range với comm_rate)
- BR-035a: **rate_cap warning soft validations**:
  - `rate_cap < BASE_RATE` → E-VCR-019 warning: "Cap thấp hơn Base Rate, Additive có thể cap ngay cả khi cả delta = 0"
  - `rate_cap = 0%` → E-VCR-019 stronger warning: "Cap = 0% — mọi booking sẽ có rate 0%"
- BR-036: Khi `override_mode != 'none'` và tất cả rules bị inactive/xóa hết → **giữ nguyên mode**, tất cả booking fallback về Base Rate (không tự động switch về 'none'). UI hiện AllInactiveBanner trong drawer
- BR-037: Ít nhất 1 active rule: cảnh báo khi tất cả rules inactive — không block operation
- BR-038: **rule_name / season_name validation**:
  - Max 100 ký tự (DB schema VARCHAR(100)). Inline error E-VCR-020 khi exceed
  - Hỗ trợ Unicode (NFR-011): tiếng Nhật/Hàn/Việt/emoji
  - Server-side phải HTML-escape khi render audit log (chống XSS)
  - Không bắt unique trong cùng contract — duplicate name allowed (admin có thể vô tình tạo nhiều "Cherry Blossom"; soft warning)
- BR-039: **Optimistic locking**: mỗi rule và contract có `version` (INT, default 0). UPDATE statement luôn `WHERE id=? AND version=?` — version mismatch → trigger E-VCR-017 conflict dialog. Increment version sau mỗi successful save
- BR-040: **Authorization (NFR-007)**: mọi API endpoint (read rules, simulate, save rules, recalculate booking) đều validate `user_id` có quyền với `contract_id` trước khi execute. Cross-contract access → HTTP 403 + log security event (E-VCR-018)

**Acceptance Criteria**:
- [ ] AC-027: Validation messages hiển thị inline (không dùng alert popup)
- [ ] AC-028: Warning cho overlap/gap/duplicate priority — dạng informational, không block save
- [ ] AC-029: Error cho out-of-range rate — block save
- [ ] AC-029a: rule_name length validation triggered ở blur event hoặc trên submit
- [ ] AC-029b: Concurrent edit conflict (2 user save cùng rule): user thứ 2 thấy E-VCR-017 dialog
- [ ] AC-029c: Cross-contract API call (tampered contract_id) bị reject 403 + monitoring alert

---

### FR-009: Data Model

**Bảng hiện có — Mở rộng `vendor_contract`**:

| Field | Type | Constraint | Description |
|-------|------|-----------|-------------|
| override_mode | ENUM | NOT NULL, DEFAULT 'none' | 'none' \| 'booking' \| 'stay' \| 'combined' |
| combined_logic | ENUM | NULL (chỉ khi combined) | 'higher_wins' \| 'lower_wins' \| 'stay_priority' \| 'booking_priority' \| 'additive' |
| rate_cap | DECIMAL(5,2) | NULL | Max commission cap (cho additive mode); persist độc lập với combined_logic (BR-017b) |
| version | INT | NOT NULL, DEFAULT 0 | Optimistic locking (BR-039) |

**Bảng mới — `booking_rate_rules`**:

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
| extra_conditions | JSON | NULL | **Phase 2 forward compatibility** (room_type, day_of_week, channel). Phase 1 luôn NULL, không validate |
| version | INT | NOT NULL, DEFAULT 0 | Optimistic locking |
| created_at | DATETIME | NOT NULL | Dùng để tie-break cùng priority (BR-009) |
| updated_at | DATETIME | NOT NULL | |

**Bảng mới — `stay_rate_rules`**:

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
| extra_conditions | JSON | NULL | **Phase 2 forward compatibility**. Phase 1 luôn NULL, không validate |
| version | INT | NOT NULL, DEFAULT 0 | Optimistic locking |
| created_at | DATETIME | NOT NULL | |
| updated_at | DATETIME | NOT NULL | |

**Bảng mới — `vcomm_audit_log`** (FR-007 unified):

| Field | Type | Constraint | Description |
|-------|------|-----------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | |
| user_id | BIGINT | NOT NULL | Actor |
| timestamp | DATETIME | NOT NULL | Action time |
| action_type | ENUM | NOT NULL | List in BR-028 (RULE_CREATED, OVERRIDE_MODE_CHANGED, ...) |
| entity_type | ENUM | NOT NULL | 'rule' \| 'contract' \| 'booking' |
| entity_id | BIGINT | NOT NULL | ID của entity bị tác động |
| contract_id | BIGINT | NOT NULL | Để filter theo contract |
| old_value | JSON | NULL | Snapshot trước khi thay đổi |
| new_value | JSON | NULL | Snapshot sau thay đổi |
| metadata | JSON | NULL | matched_rule_ids, evaluation_mode, error_code, etc. |
| created_at | DATETIME | NOT NULL | |

Index: `(contract_id, timestamp DESC)`, `(entity_id, entity_type, timestamp DESC)`, `(action_type, timestamp DESC)`

---

### FR-010: Processing Logic

Khi booking được tạo hoặc sửa đổi:

| Step | Action | Detail |
|------|--------|--------|
| 0 | Authorization | Validate user có quyền với contract_id (BR-040). Cross-contract → 403 + log security event |
| 1 | Check override_mode | Nếu `'none'` → return Base Rate ngay. Dừng xử lý (short-circuit per NFR-015) |
| 2 | Calculate Lead Time | `Lead Time = daysBetween(check_in_date, booking_date)` dùng DATE-only components ở server timezone (UTC/JST fixed). DST-safe (BR-005, BR-022b). Always ≥ 0 |
| 3a | Booking mode | Query `booking_rate_rules` WHERE `contract_id` match AND `lead_min_days ≤ Lead Time ≤ lead_max_days` AND `is_active = true`. Order by `priority ASC, created_at DESC, id DESC`. Return first match |
| 3b | Stay mode | Query `stay_rate_rules` WHERE `contract_id` match AND check-in MM-DD trong range `stay_from–stay_to` (handle cross-year + 1-day per BR-015) AND `is_active = true`. Order by `priority ASC, created_at DESC, id DESC`. Return first match |
| 3c | Combined mode | Execute cả 3a và 3b (each fallback Base if no match). Apply `combined_logic`. Additive: enforce `rate_cap` (cap=25% nếu NULL, BR-035) + floor 0% (BR-017a) |
| 4 | Fallback | Nếu không match rule → return Base Rate |
| 5 | Lock & Log | Khóa rate trên booking. Ghi audit trail vào `vcomm_audit_log`: action_type=`RATE_LOCKED_AT_BOOKING_CREATION` (hoặc `RATE_RE_EVALUATED_ON_BOOKING_MODIFY` nếu update flow), applied rate, matched rule IDs (JSON array), mode, raw vs final value (cho Additive), timestamp |
| 6 | Error fallback | Nếu evaluation throw exception (DB timeout, corrupt rule) → fallback Base Rate, log E-VCR-016 + alert monitoring (không block booking creation) |

---

### FR-011: Save Coordination (Drawer Staging + Atomic Modal Commit)

**Description**: Override config được edit trong **OverrideDrawer** (slide-in từ modal cha Vendor Contract). Để đảm bảo data integrity giữa contract main fields + override config + rules con, hệ thống dùng **2-step staging save**.

**Business Rules**:
- BR-041: **Drawer Apply (staging)** — click [Apply Changes] trong drawer:
  - KHÔNG POST backend ngay
  - Lưu drawer state vào in-memory staging của modal cha (React state, Vue store, etc.)
  - Drawer đóng → OverrideStatusBadge update với suffix "Modified, not saved" (yellow)
  - Modal cha [Save] button hiển thị asterisk: `[Save *]` với tooltip
- BR-042: **Modal Save (atomic flush)** — click [Save] modal cha:
  - Validate cả contract main fields + override staging
  - 1 transaction POST `/api/vendor-contracts/{id}` với payload:
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
  - Backend mở DB transaction: UPDATE vendor_contract + INSERT/UPDATE/DELETE rules + INSERT audit logs
  - Trên success: clear staging, badge sync về DB, modal cha đóng, toast success
  - Trên fail: hiển thị E-VCR-014 hoặc E-VCR-017 (optimistic lock conflict). Modal cha và staging giữ nguyên để user retry
- BR-043: **Discard handling**:
  - Drawer [Discard Changes]: nếu drawer state khác staging → confirm E-VCR-013 → revert drawer state về staging
  - Modal cha [Cancel]: nếu có pending changes (modal fields hoặc override staging) → confirm E-VCR-013 với liệt kê các thay đổi → revert all
- BR-044: **Auto-save draft**:
  - Drawer staging được auto-save vào `localStorage` mỗi **10 giây** với key `vcomm_draft_{contract_id}_{user_id}`
  - Khi user mở lại modal trên cùng contract: detect draft → hỏi "Khôi phục pending override changes từ {timestamp}?" với 2 button [Restore] [Discard draft]
  - Sau khi modal cha Save thành công → clear draft
- BR-045: **Concurrent edit detection (BR-039)**: payload version từ frontend phải match `vendor_contract.version` ở backend; mismatch → 409 Conflict + E-VCR-017

**Acceptance Criteria**:
- [ ] AC-030: Drawer [Apply] không tạo network request đến rate-rules API
- [ ] AC-031: Modal cha Save trigger đúng 1 API call với payload merged
- [ ] AC-032: Đóng tab trình duyệt với pending changes → reload mở lại → restore prompt hiển thị
- [ ] AC-033: Optimistic lock conflict (2 tab edit cùng contract) → user thứ 2 thấy conflict dialog với option Reload/Override
- [ ] AC-034: Modal cha [Save] disabled khi không có changes; enabled khi có override staging hoặc contract field changes

---

### FR-012: Cross-Feature Display Synchronization

**Description**: Khi user thay đổi Base Rate (V.Comm Rate field) hoặc override config (mode/rules/logic), các screen ELLIS hiện có hiển thị dữ liệu pricing/commission liên quan phải đồng bộ. Đặc biệt, **bookings đã tồn tại** giữ nguyên rate locked tại thời điểm tạo (regression — BR-025).

**Affected Screens** (existing ELLIS — không phải feature mới):
- Vendor Contract edit form (chỗ override drawer mở từ — đã cover BR-004a)
- **Rate & Allotment Collection list** — header strip hiển thị `Comm Rate: X%` + indicator mới
- **Rate & Allotment Plan & Promotion modals** — pricing recalculate
- **Rate Detail Gross tab** — cột Vendor Net, Contract Amount, Margin Rate, Margin Amount mỗi row
- **Rate Detail "Apply All" action** — bulk recalculation cho rows tương lai
- **Reservation Hotel Price Detail (Vendor section)** — regression: rate locked, không đổi

**Business Rules**:

- BR-046: **CommRateHeaderIndicator** — component mới trên header strip của Rate & Allotment Collection list, sau label "Comm Rate: X%":
  - **Visibility**: ⓘ icon CHỈ hiển thị khi `override_mode !== 'none'`. Khi `override_mode = 'none'`, header chỉ render "Comm Rate: X%" (không icon)
  - **Trigger**: click ⓘ → mở Popover anchored đến icon (380px width, max 480px height)
  - **Popover sections** (3 cố định, **không có action buttons** — pure informational):
    1. **BASE RATE**: current value (15%) + "Last changed: {date} by {user} (was {prev}%)" — chỉ render section "Last changed" khi đổi trong 30 ngày gần nhất
    2. **OVERRIDE STATUS**: "✓ Active — {MODE} mode ({N} active rules)". Combined mode: thêm "Combined logic: {logic}". Additive logic: thêm "Rate cap: {cap}%"
    3. **ACTIVE RULES (N)**: scrollable list, sort `priority ASC`. Mỗi row: priority badge + mode color dot + name + period (`D-X~Y` cho booking, `MM-DD → MM-DD` cho stay) + rate + delta badge (xanh tăng / đỏ giảm theo BR mới đảo màu)
    4. Inactive rules ẩn mặc định, có toggle "Show inactive (N)" để hiện
  - **Close**: × button + Esc + click outside
  - **z-index**: cao hơn header strip nhưng thấp hơn modal/drawer

- BR-047: **Rate & Allotment Collection list** header — `Comm Rate: X%` label compute từ `vendor_contract.base_rate` của contract đang xem. Khi user save Base Rate change trong Vendor Contract modal → list refresh để hiển thị giá trị mới (next page view, không cần realtime websocket — xem OQ-004)

- BR-048: **Rate Detail Gross tab** — các cột Vendor Net, Contract Amount, Margin Rate (%), Margin Amount mỗi row được compute từ:
  - Sell Rate × applicable V.Comm Rate
  - Applicable rate = engine result (FR-010) cho row's stay_date / booking_date
  - Khi Base Rate hoặc rules thay đổi → list cần refresh display (next view)

- BR-049: **"Apply All" action** trong Rate Detail — sau khi user save Base Rate change, click "Apply All" trigger bulk recompute cho các rows tương lai (chưa book). Đây là action chủ động của user, không tự động. Existing bookings không bị tác động (BR-025)

- BR-050: **Reservation Hotel Price Detail regression** (BR-025 reinforce):
  - Existing booking rate được lock tại booking creation và lưu snapshot vào `bookings.contractList`
  - Khi Base Rate hoặc override rules thay đổi → Reservation Hotel Price Detail của booking cũ KHÔNG đổi
  - Chỉ thay đổi qua manual recalculate (FR-006 BR-026) hoặc booking modify (BR-024)

**Acceptance Criteria**:
- [ ] AC-035: ⓘ icon hidden khi `override_mode = 'none'`, visible khi != 'none'
- [ ] AC-036: Click ⓘ → Popover render với 3 sections (BASE RATE / OVERRIDE STATUS / ACTIVE RULES)
- [ ] AC-037: Active Rules list scrollable khi N > 5, không cần pagination (max ~50 rules per BR-007)
- [ ] AC-038: "Show inactive (N)" toggle hiển thị inactive rules trong cùng list
- [ ] AC-039: Popover đóng bằng × / Esc / click outside; không có navigation buttons (informational only)
- [ ] AC-040: Base Rate change save → Rate & Allotment Collection list label update on next view
- [ ] AC-041: Base Rate change save → Rate Detail Gross tab Vendor Net/Margin recompute on next view
- [ ] AC-042: Existing booking trong Reservation Hotel Price Detail KHÔNG đổi sau Base Rate / rules change

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
| OQ-001 | Booking Engine API: tạo endpoint mới hay mở rộng API hiện có? Fallback behavior khi API lỗi? | Cần quyết định architecture integration | TBD |
| OQ-002 | Migration strategy cho ALTER TABLE vendor_contract trên production? | Bảng có thể có nhiều records, cần đánh giá impact | TBD |
| OQ-003 | Rule creation UX: Modal dialog, inline row editing, hay side panel? | Ảnh hưởng đến UX flow. Resolved — side panel trong drawer (BR-004a) | RESOLVED |
| OQ-004 | Cross-feature display refresh strategy: realtime websocket vs reload-on-next-view? | FR-012 BR-047/048 — ảnh hưởng latency UX khi user toggle giữa Rate & Allotment và Vendor Contract modal | TBD — gợi ý reload-on-next-view (Phase 1) đơn giản, websocket defer Phase 2 |

---

## 9. Review History

| Round | Planner Score | Tester Score | Key Decisions | Date |
|-------|---------------|--------------|---------------|------|
| — | — | — | Draft created from DOCX spec + analyst requirements gathering | 2026-04-16 |
| Update | — | — | Synced with DOCX v1.0 + prototype v2: added Phase 2 Roadmap, Industry Benchmark, Use Case columns, Cross-Reference Matrix, prototype color tokens & components | 2026-05-05 |
| 1 | 8/10 | 5/10 | Reviews identified 7 critical + 14 major + 11 minor/suggestions issues | 2026-05-05 |
| 1 (resolved) | — | — | Applied all review fixes: FR-011 Save Coordination (Option B drawer), FR-005 client-side simulation + shared evaluator, BR-005/022b DST-safe Lead Time, BR-009 tie-break created_at DESC only, BR-015a Feb 29 leap year handling, BR-017a Additive floor 0%, BR-017b rate_cap persistence, BR-022a full default state, BR-024 re-evaluate scope explicit, BR-026 batch recalculate scope, BR-028 unified audit log, BR-035a rate_cap warnings, BR-038 rule_name validation, BR-039 optimistic locking, BR-040 cross-contract auth, BR-041~045 save coordination flow. AnnualCalendarGrid topRule = priority-based (sync with engine). Cross-Reference Matrix expanded to all 5 logics. New components: OverrideStatusBadge, OverrideDrawer, AllInactiveBanner. New error codes E-VCR-013~020. New NFRs 017~022. 24 new test scenarios TS-035 to TS-058 | 2026-05-05 |

---

## 10. Phase 2 Roadmap

Các enhancement sau được lên kế hoạch cho phase tiếp theo. Architecture của Phase 1 phải đảm bảo forward compatibility cho các tính năng này.

| Feature | Description | Priority |
|---------|-------------|----------|
| Day-of-Week Modifier | Áp dụng trọng số bổ sung cho cuối tuần vs ngày thường (VD: Sat/Sun +2%) | Medium |
| Room Type Override | Cho phép phân biệt commission theo loại phòng (VD: suite vs standard) | Medium |
| Bulk CSV Import/Export | Upload/download hàng loạt season rules cho nhiều contracts qua file CSV | High |
| HARE AI Recommendation | HARE market intelligence đề xuất commission tiers tối ưu dựa trên real-time demand data và competitive analysis | High |
| Simulation Report Export | Export simulation results so sánh 3 modes cho 1 date range, dạng shareable report | Low |
| Rate Parity Integration | Auto-flag khi commission override gây parity violation với competing channels | High |

**Forward Compatibility Notes**:
- `booking_rate_rules` và `stay_rate_rules` phải có column `extra_conditions` JSON nullable để Phase 2 thêm room_type, day_of_week mà không cần migration phức tạp
- Audit trail schema phải đủ flexible cho các action types mới (RECOMMENDATION_APPLIED, BULK_IMPORT, PARITY_FLAG)
- API contract cho rate evaluation cần extension point cho HARE recommendation hooks

---

## 11. Global Industry Benchmark

Tính năng này align với practices đã được áp dụng rộng rãi trên các nền tảng OTA toàn cầu. **Nguồn**: SCM team competitive analysis Q1/2026 + public documentation của các platform.

| Capability | Industry Standard | Reference / Example | OMH Implementation |
|-----------|-------------------|---------------------|--------------------|
| Stay Date Tiers | 100% adoption — tất cả nền tảng major hỗ trợ seasonal commission differentiation | Booking.com Genius seasonal rates, Expedia VIP Access, Agoda Smart Insights | Mode 2 (Stay). Full season definition với MM-DD recurring ranges |
| Lead Time Tiers | Advanced feature — chỉ một số platform cung cấp lead-time incentive | Booking.com Early Booker Deals, Hotelbeds Advance Purchase | Mode 1 (Booking). 5-tier structure từ Early Bird đến Urgent |
| Combined Logic | Rare — chỉ enterprise-grade platforms có dual-condition evaluation | Hotelbeds Dynamic Margin (custom contracts), enterprise CMS integrations | Mode 3 (Combined). 5 resolution logics. **Differentiator cho OMH** |
| Priority Resolution | Standard — tất cả platforms dùng priority ranking cho overlapping rules | Universal pattern (no specific reference) | Integer priority per rule. Lower = higher precedence |
| Base + Override | Universal — single base rate với exception overrides | All major OTAs | V.Comm Rate hiện có làm base. Override rules layered on top |
| Live Simulation | Uncommon — most platforms yêu cầu save-then-test workflow | Hotelbeds Rate Simulator (limited), most others save-and-test | Built-in client-side simulation panel. Immediate rate verification before save |

**Note**: Industry claims dựa trên internal review tại 2026-Q1. Cần update khi có competitive intel mới (định kỳ 6 tháng).
