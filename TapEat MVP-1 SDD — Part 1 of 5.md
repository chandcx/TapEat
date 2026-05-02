TapEat MVP-1 SDD — Part 1 of 5

1. Product Overview
TapEat is a browser-based QR food ordering platform for restaurants, cafés, and cafeterias in India.
A customer scans a QR code on a table or counter. A mobile-optimised menu opens instantly in the browser — no app, no login. The customer orders and pays via UPI. The order appears on the kitchen staff's real-time dashboard. Vendors manage menus, tables, and orders from the same web application using role-based access.
MVP-1 exists for one reason: validate the core hypothesis with three real pilot vendors before building anything else.
Core hypothesis: QR-based browser ordering works in real restaurant conditions without an app, and vendors will adopt a digital kitchen dashboard over paper tickets.

2. MVP-1 Scope
In Scope
#
Feature
Notes
1
QR-to-browser menu
Server-rendered. No login. Vendor + table from URL params.
2
Mobile menu browsing
Category tabs, search, veg/non-veg filter, sold-out states.
3
Cart — add, remove, update qty
Zustand. Persisted in sessionStorage. Cleared after order.
4
UPI payment — Razorpay Standard Checkout
Pay-before-order only. Signature verified server-side.
5
Order creation — idempotent, price-validated
API-only. Customer never writes to DB directly.
6
Payment failure + cart-preserving retry
Toast + preserved cart. Retry works.
7
Customer order-status page
Polls every 15s. Animates on ready.
8
Real-time kitchen dashboard
Supabase WebSocket + 15s polling fallback.
9
Full order lifecycle
placed → accepted → preparing → ready → completed. Server-enforced state machine.
10
Order rejection with reason
Preset list. Stored. Audit log written.
11
Order cancellation
Manager + Owner only. Staff cannot cancel.
12
Auto-cancel stale orders
Vercel Cron: cancel placed orders older than 30 min.
13
Vendor + staff login
Supabase Auth. Email + password.
14
Role-based route protection
Owner: all. Manager: all except staff + settings. Staff: /kitchen only.
15
Menu CRUD + availability toggle
Owner and Manager. Toggle saves immediately — no form submit.
16
Category management
Create, rename, delete (only if empty).
17
Table + counter QR management
Create service points. Generate QR URL. Download PNG.
18
Dashboard home
Today's orders + revenue. Recent 10 orders.
19
Order history
Date filter, status filter, CSV export (client-side).
20
Staff management
Owner only. Invite by email, assign role, remove.
21
Vendor settings
Owner only. Name, description, address, phone.
22
Veg / non-veg indicator
Green/red dot. Required for India context.
23
Counter pickup token display
Token on customer screen + kitchen card.
24
Audit log
All order status changes. Written server-side.
25
Razorpay webhook handling
Idempotent. Sig verified before any DB write.

Out of Scope — MVP-1
Excluded Feature
When
Customer accounts, login, order history
MVP-2
Item modifiers / add-ons
MVP-2
Email / WhatsApp notifications
MVP-2
Self-serve vendor registration
MVP-2
Item image upload
MVP-2
Analytics charts, trend views
MVP-2
Refund UI in dashboard
MVP-2
Browser push notifications
MVP-2
Multi-location / outlet hierarchy
MVP-3
Subscription billing
MVP-3
White-label branding
MVP-3
COD / pay-after-order
Evaluate at MVP-2
Loyalty, AI recs, promo codes
MVP-4
Printer / ESC-POS
MVP-4
React Native or PWA app
Never — browser is the differentiator


3. Users and Roles
Role Definitions
Role
Auth
Device
What They Do
Customer
None — anonymous
Android Chrome (mobile)
Scan QR → order → pay → track status
Vendor Owner
Supabase Auth
Laptop or iPad
Full control: menu, staff, tables, orders, settings
Outlet Manager
Supabase Auth
Laptop or iPad
Operations: menu, tables, order history. No staff or settings.
Kitchen Staff
Supabase Auth
Android tablet 10" landscape
Accept, prepare, complete live orders. Nothing else.

Role Architecture Decision
Owner and Manager share one dashboard application (/dashboard). There is no separate Manager app.
The difference is enforced in two places only:
middleware.ts blocks /dashboard/staff and /dashboard/settings for the Manager role
VendorNav conditionally renders those two links only when role === 'owner'
Staff see only /kitchen. Middleware redirects every /dashboard/* attempt by a Staff user to /kitchen. The kitchen page has no nav bar — it is a full-screen operational view.
Role Access Summary
Route
Customer
Owner
Manager
Staff
/menu
✅
✅
✅
✅
/order-status
✅
—
—
—
/kitchen
—
✅
✅
✅
/dashboard
—
✅
✅
—
/dashboard/menu
—
✅
✅
—
/dashboard/tables
—
✅
✅
—
/dashboard/orders
—
✅
✅
—
/dashboard/staff
—
✅ only
❌ 403
—
/dashboard/settings
—
✅ only
❌ 403
—


4. Core User Journeys
Journey 1 — Customer Dine-In Order
Scan QR on table
  └─ Browser opens: /menu?v={vendor_slug}&t={service_point_uuid}
  └─ Menu loads (SSR, < 2s on 4G)
  └─ Customer browses, adds items
  └─ Cart bar appears at bottom
  └─ Taps "View Cart" → bottom sheet opens
  └─ Optional: enters name, phone, notes
  └─ Taps "Pay ₹X"
      └─ Frontend generates idempotency_key (UUID v4) → sessionStorage
      └─ POST /api/razorpay/create-order → gets rzpOrderId
      └─ Razorpay checkout opens
      └─ Customer pays via UPI
      └─ POST /api/orders (sig verify + price validate + idempotency check)
      └─ Navigate to /order-status?id={order_number}
      └─ Cart cleared
  └─ Status page polls every 15s
  └─ On status = ready → full-screen green + audio alert
Journey 2 — Customer Counter Pickup
Scan QR at counter stand
  └─ /menu?v={slug}&t={counter_uuid}
  └─ Same ordering flow
  └─ Order created with service_mode = counter_pickup
  └─ /order-status shows token number (e.g. Token 42)
  └─ On ready → "Token 42 — collect at counter"
Journey 3 — Kitchen Staff Order Flow
Staff logs in → /kitchen (only destination)
  └─ Real-time order queue (Supabase WebSocket)
  └─ New order → audio alert + card slides in
  └─ Staff taps "Accept" → status = accepted (optimistic UI)
  └─ Staff taps "Start Preparing" → status = preparing
  └─ Staff taps "Mark Ready" → status = ready
      └─ Customer status page updates on next poll
  └─ Staff taps "Complete" → status = completed → card archived
  └─ To reject → opens modal → selects preset reason → confirms
Journey 4 — Vendor Owner Manages Menu
Owner logs in → /dashboard
  └─ /dashboard/menu
  └─ Toggles availability → PATCH /api/menu-items/{id}/availability (immediate)
  └─ Adds item → Dialog form → POST /api/menu-items
  └─ Edits item → same Dialog pre-filled → PATCH
  └─ Deletes item → confirmation → DELETE (blocked if category has items)
Journey 5 — Owner Invites Staff
Owner → /dashboard/staff
  └─ Enters email + selects role (Manager or Staff)
  └─ POST /api/admin/invite-staff (Owner-only route)
  └─ Server calls supabase.auth.admin.inviteUserByEmail()
  └─ user_roles record inserted immediately
  └─ Invited user receives magic link → sets password → logs in → routed by role
Journey 6 — Payment Failure Recovery
Customer pays → network drops after payment
  └─ Retry: POST /api/orders with same idempotency_key
  └─ Server finds existing order → returns 409 + { orderNumber }
  └─ Frontend navigates to /order-status for existing order
  └─ No duplicate charge. No duplicate order.

5. Business Rules
Payment Rules
Rule
Detail
Pay-before-order only
No COD. No pay-later. No exceptions in MVP-1.
Price re-validation
Server re-fetches all prices from DB. Rejects if any price differs by > ₹0.
HMAC signature required
`createHmac('sha256', KEY_SECRET).update(rzpOrderId + '
Idempotency
idempotency_key (UUID v4) unique per order. 409 returned if duplicate. Frontend navigates to existing order.
Webhook dedup
payment_events.event_id unique constraint prevents double processing.

Order Rules
Rule
Detail
Auto-cancel
placed orders with no action after 30 minutes are cancelled by cron. Audit log written.
State machine
Transitions enforced server-side. Invalid transitions return 422.
Reject requires reason
rejection_reason from preset list. Required field. Cannot reject without selecting.
Cancel: Manager + Owner only
Staff cannot cancel. Cancel button is invisible (not disabled) for Staff role. Programmatic PATCH also returns 403 for Staff.
Price denormalisation
order_items.price stores price at time of order. Immune to future price changes.
Service point denormalised
orders.service_point_label stored at creation. Correct even if service point is later renamed or deleted.

Menu Rules
Rule
Detail
Availability toggle
Saves immediately on click. No form submit. Customer menu reflects change on next request.
Sold-out display
is_available = false hides the Add button entirely. Item is shown with Sold Out pill.
Category delete guard
Categories with items assigned cannot be deleted. Server returns 422. Client shows explanation.
Veg indicator
Every item must have is_veg value. Displayed as green dot (veg) or red dot (non-veg) on all surfaces.

Vendor Rules
Rule
Detail
Manual onboarding
Founder creates all vendor records for pilot. No self-serve registration.
Single location only
No outlet hierarchy in MVP-1. Each vendor = one location.
Staff invite
Owner-only. Server checks role before calling Supabase admin API.
QR URL uses UUID
/menu?v={slug}&t={service_point_uuid} — UUID prevents table ID enumeration.
One Razorpay account
Single platform Razorpay account. Vendor payments settled to platform. No marketplace split in MVP-1.

Notification Rules
Rule
Detail
Kitchen alert
Audio on new order INSERT. MP3 first, Web Audio API oscillator fallback.
iOS audio
Requires one user gesture before AudioContext plays. Show "Tap to enable sound" prompt on first load.
Customer notification
Polling only. 15-second interval. No push notifications in MVP-1.
WebSocket failure
Show amber "Reconnecting..." banner. Activate 15s polling fallback. Reconnect: 5s → 10s → 30s backoff.




