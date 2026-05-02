TapEat MVP-1 SDD — Part 2 of 5

6. Screen / Module List by Role
6.1 Customer Screens
Screen
Route
Purpose
Key Actions
Data Shown
Menu page
/menu?v={slug}&t={uuid}
QR landing. Browse and order.
Add/remove items, filter by category, veg toggle, search
Items, categories, prices, veg dot, availability, vendor name, table label
Cart modal
Overlay on menu
Review order and pay
Edit qty, enter optional name/phone/notes, tap Pay
Item list, subtotals, order total
Order status
/order-status?id={order_number}
Track live order after payment
Polls automatically
Status, token (pickup), ready animation + audio

Customer has no nav bar. Flow is linear: QR → menu → cart → payment → status.

6.2 Kitchen Staff Screens
Screen
Route
Purpose
Key Actions
Data Shown
Kitchen dashboard
/kitchen
Live order queue management
Accept, Start Preparing, Mark Ready, Complete, Reject (with reason)
Order #, table/token, service mode badge, items × qty, total, elapsed time

Staff have no nav bar. /kitchen is their only screen. All /dashboard/* routes redirect to /kitchen via middleware.
Cancel button: Invisible for Staff. Visible only for Manager and Owner.

6.3 Vendor Owner Screens
Screen
Route
Access
Purpose
Key Actions
Dashboard home
/dashboard
Owner, Manager
Live snapshot
View today stats, navigate
Menu management
/dashboard/menu
Owner, Manager
Full item + category CRUD
Add/edit/delete items, toggle availability, manage categories
Tables & QR
/dashboard/tables
Owner, Manager
Service points + QR codes
Add table/counter, download QR PNG, copy link, deactivate
Order history
/dashboard/orders
Owner, Manager
Review all past orders
Filter by date/status, expand row, export CSV
Staff management
/dashboard/staff
Owner only
Invite and manage staff
Invite by email, assign role, remove
Settings
/dashboard/settings
Owner only
Edit vendor profile
Save name, description, address, phone
Kitchen view
/kitchen (new tab)
Owner, Manager, Staff
Operational order queue
Same as Staff kitchen screen


6.4 Screen-Level Detail
/menu — Customer Menu Page
Layout (top to bottom):
├── Header: vendor name | service point label | veg-only toggle
├── Search bar (client-side filter, no server call)
├── Recommended strip (horizontal scroll, is_recommended = true)
├── Category tabs (horizontal scroll, filters list)
├── Item list (grouped by active category)
│   └── Item card: veg dot | name | description | price | Add button / qty stepper
│   └── Sold-out state: "Sold Out" pill | Add button hidden (not disabled)
└── Cart bar (fixed bottom, hidden when empty)
    └── Item count | total | "View Cart" button
Rendering: Next.js server component fetches all data. Client components handle cart interaction. FCP target: < 2 seconds on 4G India. Vendor not found: Full-screen — "Menu unavailable. Please ask a staff member." Service point not found: Menu loads without table context. Order proceeds as walk-in.

/kitchen — Kitchen Dashboard
Layout:
├── Header: vendor name | active order count badge | connection status dot
│   └── Dot: green (SUBSCRIBED) | amber (reconnecting) | red (polling only)
├── Filter tabs: All | Pending | Preparing | Ready
└── Order card grid (2 columns, tablet landscape)
    └── Order card:
        ├── Order number (large) | service point label | DINE-IN or PICKUP badge
        ├── Items: name × qty (one per line)
        ├── Total amount
        ├── Elapsed time (updates every 30s)
        └── Action buttons (vary by status — see Section 7)
New order behaviour: Audio alert fires → card slides into Pending column. Reconnect banner: Amber banner when WebSocket is not SUBSCRIBED. Cancel button: Rendered only when role === 'manager' || role === 'owner'.

/dashboard — Dashboard Home
Stats row (4 cards):
├── Today's orders (COUNT, refreshes every 5 min)
├── Today's revenue (SUM paid, refreshes every 5 min)
├── Pending orders (COUNT placed+accepted+preparing, real-time)
└── Active menu items (COUNT is_available=true, on load)

Recent orders table:
└── Last 10 orders | columns: Order # | Service Point | Status badge | Amount | Time ago
└── "View all" → /dashboard/orders

/dashboard/menu — Menu Management
Layout:
├── Category accordion or tab strip
│   └── Per category: item cards with toggle switch
└── "+ Add Item" button → Dialog form
    └── Fields: name | description | price | category (select) |
               is_veg (toggle) | is_available (toggle) | is_recommended (toggle)
    └── Validation: name ≥ 2 chars, price ≥ 0 (client + server)

Per-item actions:
├── Toggle availability → PATCH /api/menu-items/{id}/availability (immediate, no form)
├── Edit → same Dialog pre-filled
└── Delete → confirmation Dialog (blocked if has active orders? No — delete is allowed)

Category actions:
├── Add category → inline input
└── Delete category → only if zero items assigned (server returns 422 otherwise)

/dashboard/tables — Tables and QR
Add service point form:
└── Fields: label (e.g. T1, Counter A) | service_mode (dine_in / counter_pickup) | section (optional)

Service point card grid:
└── Per card: label | section | mode badge | QR code preview
    └── Actions: Download PNG | Copy link | Deactivate (with confirmation)

QR URL format: {APP_URL}/menu?v={vendor_slug}&t={service_point_uuid}

/dashboard/orders — Order History
Filters:
├── Date range: Today | Last 7 days | Last 30 days | Custom
└── Status: All | Placed | Accepted | Preparing | Ready | Completed | Rejected | Cancelled

Table (20 per page):
└── Columns: Order # | Service Point | Status badge | Payment Status | Total | Created At
└── Expandable row: items list for that order

Export: CSV button → client-side generation from current query result
Columns in CSV: Order #, Service Point, Status, Payment Status, Amount, Created At

/dashboard/staff — Staff Management (Owner Only)
Invite form:
└── Email input | Role select (Manager / Staff) | Invite button
└── Server: supabase.auth.admin.inviteUserByEmail() + user_roles insert

Staff table:
└── Columns: Email | Role badge | Status (Active / Invited) | Remove button
└── Remove: deletes user_roles record → user loses access immediately

7. Functional Requirements
FR-01: Menu Loading
ID
Requirement
FR-01.1
Menu MUST achieve FCP < 2 seconds on 4G India. Fetched server-side on initial load.
FR-01.2
Vendor resolved from v= slug. Service point resolved from t= UUID. Both resolved before page renders.
FR-01.3
Items with is_available = false MUST show Sold Out pill. Add button MUST be hidden — not disabled.
FR-01.4
Cart state MUST persist in sessionStorage. Survives page refresh within session.
FR-01.5
Cart total MUST update instantly on every add/remove. No server call for total.
FR-01.6
Veg-only toggle MUST filter items client-side with no server request.
FR-01.7
Search MUST filter items by name client-side in real time. No server call.

FR-02: Payment and Order Creation
ID
Requirement
FR-02.1
Frontend MUST generate idempotency_key (UUID v4) before Razorpay opens. Stored in sessionStorage.
FR-02.2
/api/razorpay/create-order MUST be rate limited: 10 req / IP / 60 sec.
FR-02.3
Server MUST verify HMAC-SHA256 payment signature. Return 400 if invalid. Order not created.
FR-02.4
Server MUST re-fetch all item prices from DB. Return 422 if any price differs by > ₹0.
FR-02.5
Server MUST check idempotency_key uniqueness before insert. Return 409 + { orderNumber } if duplicate.
FR-02.6
On payment failure or timeout, cart MUST be preserved. Toast shown. Retry available.
FR-02.7
On successful order creation, frontend MUST navigate to /order-status and clear cart.

FR-03: Kitchen Operations
ID
Requirement
FR-03.1
Kitchen dashboard MUST show new orders within 1.5 seconds via Supabase real-time.
FR-03.2
Audio alert MUST fire on every new order INSERT. MP3 first, Web Audio API oscillator fallback.
FR-03.3
All status transitions MUST be validated server-side. Invalid transitions return 422.
FR-03.4
Rejection MUST require a preset reason. rejection_reason stored. Audit log written.
FR-03.5
Cancel button MUST be invisible (not disabled) for Staff role. PATCH also returns 403 for Staff attempting cancel.
FR-03.6
WebSocket drop MUST trigger amber banner + 15-second polling fallback automatically.

FR-04: Vendor Operations
ID
Requirement
FR-04.1
Availability toggle MUST persist to DB immediately on click via PATCH /api/menu-items/{id}/availability.
FR-04.2
Availability change MUST reflect on customer menu page on next request (no stale cache).
FR-04.3
Item form MUST validate: name ≥ 2 chars, price ≥ 0. Validated client AND server.
FR-04.4
Category delete MUST return 422 server-side if items are assigned. Client shows explanation.
FR-04.5
Staff invite MUST be Owner-only. Server checks role before calling Supabase admin API. Returns 403 for non-owners.

FR-05: Auto-Cancel Cron
ID
Requirement
FR-05.1
Vercel Cron fires GET /api/cron/auto-cancel every 10 minutes.
FR-05.2
Route checks x-cron-secret header against CRON_SECRET env var. Returns 401 if wrong or missing.
FR-05.3
Cancels all orders WHERE status = placed AND created_at < now() - interval 30 min.
FR-05.4
Each auto-cancelled order gets an audit_log entry with action = auto_cancel.


8. Non-Functional Requirements
NFR
Target
Approach
Menu FCP
< 2s on 4G India
SSR via Next.js App Router. All menu data fetched server-side on first load.
Order placement (post-payment)
< 3s total
API route target < 500ms. Razorpay checkout excluded from measurement.
Kitchen real-time latency
< 1.5s new order to card
Supabase WebSocket. Monitor created_at vs card render delta.
Concurrent orders (MVP-1)
20 simultaneous across 3 vendors
Supabase free tier sufficient. Monitor and upgrade if needed.
Mobile browser support
Android Chrome 90+, iOS Safari 15+
No CSS :has(). No grid subgrid. Test on real device — not emulator.
Kitchen tablet
10" Android, 1280×800, landscape
All interactive elements ≥ 44px touch target. Test on real hardware.
TypeScript
Strict, zero any
tsc --noEmit must pass with 0 errors before every deploy.
Build
Must succeed
npm run build zero errors. No server-only keys in client bundle.
Uptime
99% monthly (best effort)
Vercel + Supabase managed infra. No formal SLA in MVP-1.
Accessibility
Basic WCAG 2.1 AA on customer menu
shadcn/ui components are accessible. Add aria-label to icon-only buttons.


9. Notifications and Order States
9.1 Order Status State Machine
                   ┌─────────────────────────────────────┐
                    │              PLACED                  │
                    │  (payment_status = paid required)    │
                    └──────┬──────────┬────────────────────┘
                           │          │
                      [Accept]    [Reject]──────────────────→ REJECTED (terminal)
                           │      (reason required)
                           ▼
                       ACCEPTED
                           │
                    [Start Preparing]     [Cancel] ← Manager/Owner only
                           │                  │
                           ▼                  ▼
                       PREPARING          CANCELLED (terminal)
                           │
                     [Mark Ready]
                           │
                           ▼
                         READY
                           │
                      [Complete]
                           │
                           ▼
                       COMPLETED (terminal)
Valid transitions (server-enforced):
From
Allowed Next States
placed
accepted, rejected, cancelled
accepted
preparing, cancelled
preparing
ready
ready
completed
completed
— terminal
rejected
— terminal
cancelled
— terminal

Rules:
cancelled transition: 403 for Staff. Manager and Owner only.
rejected transition: rejectionReason required. 400 if missing.
Invalid transition: 422 { error: "Invalid status transition", code: "INVALID_TRANSITION" }.
Every transition: writes audit_log entry with user_id, old_value, new_value, timestamp.

9.2 Customer-Facing Status Messages
Status
Message Shown
placed
"Order placed. Preparing shortly." + spinner
accepted
"Confirmed! Your order is being prepared."
preparing
"In the kitchen now."
ready (dine_in)
Full-screen green. Audio alert. "Your order is ready at Table T3."
ready (counter_pickup)
Full-screen green. Large token. "Token 42 — collect at counter."
rejected
Red banner. "Could not be fulfilled. Reason: [reason]. Refund in 3–5 business days."
completed
"Done! Enjoy your meal."
cancelled
"Order cancelled. If payment was made, refund in 3–5 business days."


9.3 Kitchen Action Buttons Per Status
Current Status
Buttons Shown
Cancel Visible?
placed
Accept · Reject
Manager/Owner only
accepted
Start Preparing · Reject
Manager/Owner only
preparing
Mark Ready
Manager/Owner only
ready
Complete
Manager/Owner only
completed
— (archived)
—
rejected
— (archived)
—
cancelled
— (archived)
—


9.4 Notification Architecture
Event
Channel
Mechanism
Fallback
New order → kitchen
In-browser audio
MP3 file → Web Audio API oscillator
Polling picks up missed orders every 15s
Order status → customer
Polling
GET /api/orders/{id}/status every 15s
N/A — polling IS the mechanism
WebSocket drop → kitchen
Connection banner
Monitor channel state, show reconnecting UI
15s polling activates automatically
Order ready → customer
Page animation + audio
Triggered when poll returns ready
N/A

Audio alert implementation:
typescript
function playOrderAlert() {
  try {
    const a = new Audio('/sounds/order-alert.mp3');
    a.volume = 0.8;
    a.play();
    return;
  } catch {}
  // Web Audio API triple-beep fallback
  try {
    const ctx = new AudioContext();
    [880, 1100, 880].forEach((freq, i) => {
      const osc = ctx.createOscillator();
      const gain = ctx.createGain();
      osc.connect(gain);
      gain.connect(ctx.destination);
      osc.frequency.value = freq;
      gain.gain.setValueAtTime(0.3, ctx.currentTime + i * 0.18);
      gain.gain.exponentialRampToValueAtTime(
        0.001, ctx.currentTime + i * 0.18 + 0.12
      );
      osc.start(ctx.currentTime + i * 0.18);
      osc.stop(ctx.currentTime + i * 0.18 + 0.15);
    });
  } catch {}
}
iOS note: AudioContext requires a user gesture before play. Show a one-time "Tap to enable sound" overlay on first load. Store gesture confirmation in sessionStorage.
WebSocket reconnect strategy:
Disconnect detected
  └─ Wait 5s → reconnect attempt 1
  └─ Wait 10s → reconnect attempt 2
  └─ Wait 30s → reconnect attempt 3+
  └─ Polling fallback active throughout
  └─ Banner dismissed automatically on SUBSCRIBED



