TapEat MVP-1 SDD — Part 5 of 5

18. Delivery Plan
18.1 Build Philosophy
Build in phases. Each phase is independently testable before the next starts.
Ugly and working beats polished and broken.
Every phase ends with tsc --noEmit passing and a manual smoke test.
No phase starts until the founder confirms "proceed".

18.2 Phase Overview
Phase
Focus
Day
Confirms
0
Pre-build setup
Before Day 1
Env vars, Supabase, Razorpay test keys
1
Foundation
Day 1 AM
Folder structure, types, store, middleware
2
API routes
Day 1 PM
All routes created, health check passes
3
Customer menu + ordering
Day 2 AM
QR → menu → pay → order-status works end-to-end
4
Kitchen dashboard
Day 2 PM
New order appears live, status transitions work
5
Vendor dashboard
Day 3
Login, menu CRUD, tables, QR, orders, staff
6
Hardening + UAT
Day 3 EOD
All P0 tests pass, build succeeds


18.3 Phase 0 — Pre-Build Setup
Do before writing any code.
□ Create Supabase project
  └─ Copy: URL, anon key, service role key

□ Run 001_mvp1.sql in Supabase SQL Editor
  └─ Verify all tables exist
  └─ Verify triggers exist (generate_order_number, fn_updated_at)
  └─ Verify RLS policies exist

□ Enable Realtime for orders table
  └─ Supabase Dashboard → Database → Replication → orders ON

□ Create Supabase storage bucket
  └─ Name: menu-images | Public: yes | Max size: 5MB
  └─ (Empty in MVP-1, ready for MVP-2)

□ Create Razorpay test account
  └─ Get test key_id and key_secret

□ Configure Razorpay webhook
  └─ URL: https://{your-domain}/api/razorpay/webhook
  └─ Events: payment.captured, payment.failed
  └─ Copy webhook secret

□ Create GitHub repo
  └─ main branch = production

□ Connect repo to Vercel
  └─ Framework: Next.js (auto-detected)
  └─ Do NOT add env vars yet — wait for Phase 1

□ Confirm Razorpay test UPI IDs:
  └─ Success: success@razorpay
  └─ Failure: failure@razorpay

18.4 Phase 1 — Foundation
Prompt to send Windsurf:
Phase 1: Project foundation. No pages yet.

1. Initialise Next.js 14 with TypeScript strict and Tailwind CSS v3
2. Install packages:
   next react react-dom typescript
   @supabase/supabase-js @supabase/ssr
   zustand react-hook-form @hookform/resolvers zod
   qrcode.react sonner lucide-react
   clsx tailwind-merge date-fns razorpay

3. Create folder structure per SDD Section 14.3
   (all files as empty stubs — no implementation yet)

4. Create types/index.ts — all types from SDD Section 10.5

5. Create lib/supabase/client.ts and lib/supabase/server.ts
   per SDD Section 16.1

6. Create lib/razorpay.ts
   — verifyPaymentSignature()
   — verifyWebhookSignature()
   per SDD Section 16.2

7. Create lib/rateLimit.ts per SDD Section 16.3

8. Create lib/utils.ts:
   — cn(ClassValue[]): string
   — formatINR(amount: number): string  (e.g. ₹18.00)
   — sanitize(text: string): string  (strip HTML, trim, cap 500 chars)
   — timeAgo(date: string): string  (e.g. "3 mins ago")

9. Create lib/validations.ts — all Zod schemas from SDD Section 11.3

10. Create store/cartStore.ts — Zustand store per SDD Section 10.5 CartState

11. Create middleware.ts — role-based route protection per SDD Section 12.2

12. Create app/layout.tsx:
    — Sora font via next/font/google
    — Startup env var validation per SDD Section 17.2

13. Create tailwind.config.ts:
    — brand.purple: #3B0764
    — brand.amber: #F59E0B

14. Create .env.example with all keys from SDD Section 17.6

End: run npx tsc --noEmit
Report all errors. Fix all. Confirm "Phase 1 complete."
Phase 1 done when:
tsc --noEmit passes with 0 errors
All stub files exist at correct paths
cartStore has all CartState actions
Middleware correctly maps routes to roles

18.5 Phase 2 — API Routes
Prompt to send Windsurf:
Phase 2: All API routes. No UI work.

Build in this exact order:

1. app/api/health/route.ts
   — Supabase ping, return { status, db, latency_ms, timestamp }

2. app/api/menu/[vendorSlug]/route.ts
   — GET, resolve vendor by slug + service point by ?t= UUID
   — Return PublicMenuResponse
   — Cache-Control: no-store

3. app/api/razorpay/create-order/route.ts
   — POST, apply rateLimit(), create Razorpay order
   — Return { rzpOrderId, amount }

4. app/api/orders/route.ts
   — POST, ALL 6 steps from SDD Section 11.2 in exact order:
     1) rate limit
     2) Zod validate with CreateOrderSchema
     3) idempotency_key check → 409 if exists
     4) HMAC verify → 400 if invalid
     5) price re-fetch and compare → 422 if any diff > ₹0
     6) insert order + order_items + audit_log (service role)
   — Return { orderId, orderNumber }

5. app/api/orders/[id]/status/route.ts
   — GET (no auth): return { status, serviceMode, servicePointLabel, rejectionReason }
   — PATCH (auth required):
     a) verify session
     b) verify user has role for this vendor
     c) check cancel permission (403 for staff)
     d) validate transition per state machine (Section 9.1)
     e) require rejectionReason if status = rejected
     f) update order + write audit_log

6. app/api/menu-items/[id]/availability/route.ts
   — PATCH, Manager/Owner auth only
   — Toggle is_available
   — Return updated item

7. app/api/razorpay/webhook/route.ts
   — Full implementation per SDD Section 16.2
   — Read raw body before parsing
   — Verify webhook signature
   — Dedup via payment_events.event_id
   — Update payment_status on captured/failed
   — Always return 200

8. app/api/admin/invite-staff/route.ts
   — POST, Owner only
   — supabase.auth.admin.inviteUserByEmail()
   — Insert user_roles immediately
   — Write audit_log

9. app/api/cron/auto-cancel/route.ts
   — Full implementation per SDD Section 17.7
   — Check x-cron-secret header
   — Cancel placed orders > 30 min
   — Write audit_log per order

Each route:
— Zod validate all inputs
— Return { error, code } on all failures
— Use adminClient (service role) for all DB writes
— Never expose stack traces

End: test GET /api/health in browser
Run npx tsc --noEmit. Confirm "Phase 2 complete."
Phase 2 done when:
GET /api/health returns { status: 'ok' }
All routes exist and return correct shapes
tsc --noEmit passes

18.6 Phase 3 — Customer Menu and Ordering
Prompt to send Windsurf:
Phase 3: Customer ordering flow. Mobile-first, max-width 480px.

Build in this order:

1. components/shared/VegIndicator.tsx
   — 14px square, green border + dot (is_veg) or red (non-veg)

2. components/shared/SkeletonCard.tsx
   — Pulsing Tailwind skeleton for menu item loading

3. components/shared/ErrorBoundary.tsx
   — Catches React render errors
   — Shows "Something went wrong" + "Refresh Page" button
   — Logs to console.error only

4. components/shared/EmptyState.tsx
   — Lucide icon + message + optional CTA button prop

5. components/customer/MenuHeader.tsx
   — Vendor name, service point label, veg-only toggle switch

6. components/customer/CategoryTabs.tsx
   — Horizontal scroll, active state, filters list client-side

7. components/customer/RecommendedStrip.tsx
   — Horizontal scroll of items where is_recommended = true

8. components/customer/MenuItemCard.tsx
   — Veg dot, name, description, price in ₹
   — Add button (shows when qty = 0) or +/− stepper (qty > 0)
   — Sold-out: hide Add/stepper, show "Sold Out" pill
   — Connects to cartStore

9. components/customer/CartBar.tsx
   — Fixed bottom, hidden when cart is empty
   — Shows: item count | total | "View Cart" button
   — Connects to cartStore

10. components/customer/CartModal.tsx
    — Bottom sheet over menu page
    — Item list with qty steppers, optional name/phone/notes fields
    — Pay button triggers full Razorpay flow
    CRITICAL: idempotency_key generated before loadRazorpayScript()
    CRITICAL: load Razorpay script dynamically per SDD Section 16.2
    CRITICAL: POST /api/orders with all required fields
    CRITICAL: on 409 → navigate to /order-status with existing orderNumber
    CRITICAL: on success → clearCart() → navigate to /order-status

11. app/(customer)/menu/page.tsx
    — Server component, resolves URL params v and t
    — Fetches GET /api/menu/[vendorSlug]?t={uuid}
    — Passes data to client components
    — Shows ErrorBoundary and skeleton on load

12. app/(customer)/order-status/page.tsx
    — Reads ?id= from URL (order_number)
    — Polls GET /api/orders/{id}/status every 15 seconds
    — State-specific UI per SDD Section 9.2
    — On status = ready: full-screen green + audio alert
    — Audio: playOrderAlert() per SDD Section 9.4

Design:
— Font: Sora
— Primary: #3B0764 purple
— Accent: #F59E0B amber
— Skeleton loaders on all data-fetching states (not spinners)

End: Full test:
scan demo QR → menu loads → add items → cart bar shows →
open cart → pay (Razorpay test) → order-status shows placed
Run npx tsc --noEmit. Confirm "Phase 3 complete."
Phase 3 done when:
QR URL opens correct vendor menu on mobile
Add/remove/qty works, cart total always correct
Razorpay test payment creates order in DB
/order-status page polls and updates

18.7 Phase 4 — Kitchen Dashboard
Prompt to send Windsurf:
Phase 4: Real-time kitchen dashboard.

Build in this order:

1. components/kitchen/StatusBadge.tsx
   — Colour-coded pill per status:
     placed: amber | accepted: blue | preparing: purple
     ready: green | completed: gray | rejected: red | cancelled: gray

2. components/kitchen/OrderFilters.tsx
   — Tab bar: All | Pending | Preparing | Ready
   — "Pending" covers placed + accepted

3. components/kitchen/KitchenHeader.tsx
   — Vendor name, active order count badge
   — Connection status dot:
     green = SUBSCRIBED | amber = RECONNECTING | red = POLLING

4. components/kitchen/RejectionModal.tsx
   — shadcn Dialog
   — Preset rejection reasons as radio group:
     "Item unavailable"
     "Sold out for the day"
     "Technical issue"
     "Other"
   — Submit button disabled until reason selected

5. components/kitchen/OrderCard.tsx
   — All elements per SDD Section 6.4 (/kitchen layout)
   — Action buttons per status per SDD Section 9.3
   — Cancel button: render only when role === 'manager' || 'owner'
   — Cancel button invisible (not disabled) for staff role
   — Reject button: opens RejectionModal, passes reason on confirm
   — Optimistic UI: update local state before PATCH response
   — On PATCH error: revert local state + show toast

6. hooks/useOrders.ts
   — Full implementation per SDD Section 16.1
   — Supabase real-time subscription (INSERT + UPDATE)
   — 15-second polling fallback when WebSocket not SUBSCRIBED
   — Reconnect backoff: 5s → 10s → 30s
   — Returns { orders, connectionStatus }

7. app/(kitchen)/kitchen/page.tsx
   — Auth check via Supabase session
   — Get vendor_id from user_roles
   — Render KitchenHeader + OrderFilters + 2-column card grid
   — Amber "Reconnecting..." banner when connectionStatus !== 'SUBSCRIBED'
   — iOS audio prompt: "Tap to enable sound" overlay on first load
     (store confirmation in sessionStorage)

Audio alert:
— Exact implementation per SDD Section 9.4
— MP3 first (/public/sounds/order-alert.mp3)
— Web Audio API triple-beep fallback

Dark theme:
— Background: #0F0F0F
— All action buttons ≥ 44px touch target height
— Optimised for 10-inch tablet, landscape orientation

End:
Place a test order → watch card appear on kitchen dashboard < 1.5s
Accept → customer status page updates within 15s
Mark ready → customer sees full-screen green
Run npx tsc --noEmit. Confirm "Phase 4 complete."
Phase 4 done when:
New order appears in kitchen within 1.5 seconds
All status transitions work and are enforced
Staff cannot see cancel button
Reconnecting banner shows when WebSocket is simulated-dropped
Audio plays on new order

18.8 Phase 5 — Vendor Dashboard
Prompt to send Windsurf:
Phase 5: Vendor auth and full management dashboard.

Build in this order:

1. app/(vendor)/login/page.tsx
   — Email + password via supabase.auth.signInWithPassword()
   — On success: redirect to /dashboard
   — Show error inline on failure

2. components/vendor/VendorNav.tsx
   — Accepts role prop ('owner' | 'manager' | 'staff')
   — Desktop: left sidebar 240px fixed
   — Mobile/tablet: bottom tab bar
   — Nav items per SDD Section 8.3
   — Staff and Settings links: render ONLY when role === 'owner'
   — "Kitchen View" opens /kitchen in new tab

3. components/shared/StatsCard.tsx
   — Lucide icon + large number + label
   — Optional trend label (unused in MVP-1)

4. app/(vendor)/dashboard/page.tsx
   — 4 stats cards (today orders, revenue, pending, active items)
   — Recent 10 orders table
   — "View all" → /dashboard/orders

5. components/vendor/MenuItemForm.tsx
   — shadcn Dialog
   — React Hook Form + Zod MenuItemSchema
   — Fields: name, description, price, category (select),
     isVeg (Switch), isAvailable (Switch), isRecommended (Switch)
   — Disable submit while in flight
   — Works for both create (POST) and edit (PATCH)

6. app/(vendor)/dashboard/menu/page.tsx
   — Category accordion or tab strip
   — Per item: name, price, veg dot, availability Switch
   — Switch: PATCH /api/menu-items/{id}/availability (immediate, no form)
   — "+ Add Item" → MenuItemForm Dialog
   — Edit icon → MenuItemForm pre-filled
   — Delete: confirmation Dialog → DELETE /api/menu-items/{id}
   — Category management: inline add, delete (blocked if has items)

7. components/vendor/QRDownload.tsx
   — qrcode.react QRCodeCanvas component
   — "Download PNG" button:
     canvas.toDataURL('image/png') → trigger download link

8. components/vendor/ServicePointCard.tsx
   — Label, section, service mode badge (DINE-IN / PICKUP)
   — QRDownload component
   — Copy link button (navigator.clipboard.writeText)
   — Deactivate button with confirmation Dialog

9. app/(vendor)/dashboard/tables/page.tsx
   — Add service point form:
     label, service_mode (select), section (optional)
   — Grid of ServicePointCards

10. app/(vendor)/dashboard/orders/page.tsx
    — Date filter: Today | Last 7 days | Last 30 days | Custom
    — Status filter chips
    — Paginated table (20/page): per SDD Section 6.4
    — Expandable rows showing order_items
    — CSV export: client-side from current query result
      columns: Order #, Service Point, Status, Payment, Amount, Created At

11. app/(vendor)/dashboard/staff/page.tsx
    — Owner only (middleware blocks Manager)
    — Invite form: email + role select → POST /api/admin/invite-staff
    — Staff table: email, role badge, status, remove button

12. app/(vendor)/dashboard/settings/page.tsx
    — Owner only (middleware blocks Manager)
    — Fields: name, description, address, phone
    — PATCH /api/vendors/{id}

All forms: React Hook Form + Zod
Use shadcn: Dialog, Table, Select, Switch, Badge, Button
Manager nav: Staff + Settings tabs not rendered

End:
Login → add item → toggle availability → add table → download QR →
view orders → invite staff (as owner)
Run npx tsc --noEmit. Confirm "Phase 5 complete."
Phase 5 done when:
Owner sees all nav items; Manager does not see Staff or Settings
Menu item toggling reflects immediately on customer menu
QR PNG downloads with correct URL
CSV exports open correctly in Excel
Staff invite flow sends email and creates user_roles record

18.9 Phase 6 — Hardening and UAT
Prompt to send Windsurf:
Phase 6: Security hardening, error audit, and UAT.

Step 1 — Security audit
Verify every item in SDD Section 17.1 (S-01 through S-17).
For each item, confirm it is implemented or report what is missing.
Fix all missing items before continuing.

Step 2 — Error handling audit
□ Every page has ErrorBoundary wrapping
□ Every API route returns { error, code } with correct HTTP code
□ Every data-fetching page shows skeleton loader while loading
□ Every list shows EmptyState when result has 0 items
□ Every form disables submit button while request is in flight
Fix any gaps found.

Step 3 — Build verification
Run: npm run build
Run: npx tsc --noEmit
Both must pass with zero errors.
Report any remaining errors and fix.

Step 4 — Final files
□ .env.example — all keys from SDD Section 17.6 present (no values)
□ vercel.json — cron config from SDD Section 17.7 present
□ README.md — 10-step setup guide:
    1. Clone repo
    2. npm install
    3. Create Supabase project
    4. Run 001_mvp1.sql
    5. Enable Realtime for orders
    6. Create Razorpay account + webhook
    7. Copy .env.example to .env.local, fill values
    8. npm run dev to test locally
    9. Push to GitHub → Vercel auto-deploys
    10. Run seed SQL, create vendor owner in Supabase Auth
□ supabase/migrations/001_mvp1.sql — matches SDD Section 10.2 exactly

Step 5 — Run all P0 UAT tests from SDD Section 19.2
Report PASS or FAIL for each.
Fix all FAIL results before declaring done.

Confirm with:
"Phase 6 complete. All P0 tests pass. Build succeeds.
Zero TypeScript errors. Ready for pilot vendor go-live."

19. QA / UAT Checklist
19.1 Test Environment Setup
□ Razorpay in TEST mode
□ Demo vendor seeded (SDD Section 13.4)
□ Owner user created in Supabase Auth + linked in user_roles
□ Manager user created + linked
□ Staff user created + linked
□ QR codes generated for Counter A, T1, T2
□ Two devices available: mobile (Android Chrome) + tablet (kitchen)

19.2 P0 Tests — Must Pass Before Go-Live
ID
Test
Steps
Expected
Status
T01
QR scan on real Android Chrome
Scan printed QR code
Menu loads < 2s. Correct vendor and table shown.
□
T02
Cart accuracy
Add 3 items, vary qty
Item count and total always correct.
□
T03
Successful payment
Complete Razorpay test UPI (success@razorpay)
Order in DB: status=placed, payment_status=paid.
□
T04
Kitchen real-time
Place order, watch kitchen on separate device
Card appears < 1.5s via WebSocket. Correct items + service point.
□
T05
Status propagation
Accept order on kitchen
Customer /order-status shows accepted within 15s.
□
T06
Price tamper protection
Manually change price in browser devtools, submit
Server returns 422. No order created. Cart intact.
□
T07
Double-tap protection
Tap Pay twice rapidly
Exactly one order created. Second call returns 409.
□
T08
Webhook signature check
Send POST to webhook with wrong signature
Returns 400. No DB changes.
□
T09
Cross-vendor RLS
Log in as Vendor A, query /dashboard/orders
Zero rows from Vendor B. RLS enforced.
□
T10
Auto-cancel cron
Leave order at placed for 35 min
Status = cancelled. Audit log entry present.
□
T11
Payment timeout recovery
Close Razorpay modal without paying
Cart preserved. Toast shown. Retry works.
□
T12
Rejection flow
Kitchen taps Reject, selects reason
status=rejected, rejection_reason stored, audit log written.
□
T13
Manager route block
Log in as Manager, navigate to /dashboard/staff
403 redirect. Staff list never rendered.
□
T14
Staff cancel block
Log in as Staff, try to cancel an order
Cancel button not visible. Direct PATCH returns 403.
□
T15
Availability toggle
Vendor toggles item to unavailable
Customer menu shows Sold Out on next load. Add button hidden.
□
T16
TypeScript
Run npx tsc --noEmit
Zero errors.
□
T17
Build
Run npm run build
Zero errors. No secrets in client bundle.
□


19.3 P1 Tests — Must Pass Before Week 2
ID
Test
Expected
Status
T18
Add menu item
New item appears in correct category on customer menu.
□
T19
QR code download
PNG downloads. URL contains correct vendor slug + service point UUID.
□
T20
Counter pickup order
Token number shown on customer screen and kitchen card.
□
T21
WebSocket disconnect
Amber banner shown. Polling activates. Auto-reconnects and banner clears.
□
T22
iOS Safari 15+
Menu loads. Cart works. Razorpay checkout opens.
□
T23
CSV export
CSV downloads. Correct columns. Opens in Excel without errors.
□
T24
Manager access
Manager sees Home, Menu, Tables, Orders, Kitchen View. NOT Staff or Settings.
□
T25
Staff invite
Owner invites by email. user_roles record created. Invited user logs in and reaches /kitchen.
□
T26
Sold-out display
is_available=false item: Sold Out pill visible, Add button hidden, item still visible.
□
T27
Order history filter
Date filter + status filter return correct results.
□


20. Launch Readiness Checklist
20.1 Technical Readiness
Infrastructure
□ Supabase project on paid plan (or free tier confirmed stable)
□ Realtime enabled for orders table
□ Daily automated backups enabled (Supabase Pro)
□ Vercel project live on custom domain (tapeat.in or similar)
□ HTTPS confirmed active
□ vercel.json cron job confirmed active in Vercel dashboard
□ All environment variables set in Vercel (not just local)

Razorpay
□ Webhook URL set and confirmed active
□ Webhook events: payment.captured + payment.failed both enabled
□ Webhook secret copied to RAZORPAY_WEBHOOK_SECRET env var
□ Test mode: all P0 tests pass
□ Live mode: KYC approved, live keys set, one test order completed

Code
□ npm run build passes with zero errors
□ npx tsc --noEmit passes with zero errors
□ No SUPABASE_SERVICE_ROLE_KEY in client bundle (inspect build output)
□ No RAZORPAY_KEY_SECRET in client bundle
□ .env.local not committed to repo (.gitignore confirmed)
□ /api/health returns { status: 'ok' } on production URL

20.2 Vendor Readiness (Per Vendor)
Vendor: ________________

Setup
□ Vendor record created in vendors table
□ Vendor slug confirmed URL-safe (lowercase, hyphens only)
□ Owner user created in Supabase Auth
□ Owner user linked in user_roles with role = 'owner'
□ Menu items entered (all categories, prices, veg flags)
□ Service points created (tables and/or counters)
□ QR codes downloaded as PNG

Physical
□ QR codes printed and laminated
□ QR codes placed at correct tables/counters
□ Kitchen device (tablet) logged in to /kitchen
□ Kitchen device: audio test passed (tap screen once for iOS)
□ Connection status dot shows green (SUBSCRIBED)

Training
□ Owner walked through /dashboard (10 min demo)
□ Kitchen staff walked through /kitchen (5 min demo)
□ Owner knows how to toggle item availability
□ Owner knows how to view order history
□ Owner has founder's WhatsApp for first-day support

Validation
□ One test order placed on each service point
□ Test order received on kitchen dashboard
□ Test order accepted, prepared, marked ready, completed
□ Customer status page updated correctly on each transition
□ Razorpay test payment confirmed in Razorpay dashboard

20.3 Operational Readiness (Founder)
Monitoring
□ Supabase dashboard bookmarked (orders table direct link)
□ Razorpay dashboard bookmarked (live payments)
□ Vercel deployment log bookmarked
□ /api/health bookmarked for quick status check

Known manual tasks for MVP-1
□ Refund process: Razorpay Dashboard → Payments → Refund
□ New vendor setup: Supabase Studio SQL insert
□ Menu import for new vendor: SQL insert or CSV in Studio
□ Deactivate vendor: UPDATE vendors SET is_active = false

Incident response
□ If kitchen WebSocket drops: polling fallback activates automatically
   — Vendor knows to refresh if banner persists > 2 min
□ If payment webhook fails: check Razorpay → Webhooks → Failed events
   — Retry from Razorpay dashboard
□ If order stuck: UPDATE orders SET status = 'cancelled' in Studio
□ If duplicate order appears: check idempotency_key in orders table

21. Known Limitations and MVP-2 Scope
21.1 Known Limitations of MVP-1
Limitation
Impact
Workaround
No item modifiers
Cannot serve vendors with "spice level", "extra cheese" menus
Choose simple-menu pilot vendors only
Manual vendor onboarding
Not scalable beyond ~10 vendors
Founder handles directly
No customer accounts
Customers cannot retrieve past orders
Acceptable — customers are on-site
Refunds manual
Founder initiates in Razorpay dashboard
Expected rare
No email/SMS confirmation
Browser-only confirmation
Sufficient for on-site ordering
In-memory rate limiter
Resets on Vercel cold start
Adequate for MVP-1 load
No item images
Text menus only
Avoids storage + upload complexity
Single location per vendor
No multi-branch support
Choose single-location pilots
No COD
UPI-only
Avoid COD-preference vendors
iOS audio requires gesture
Staff must tap once on first load
Show one-time prompt
No analytics
No charts or trend data
Founder queries Supabase directly


21.2 What Moves to MVP-2
Feature
Build Trigger
Customer accounts + order history
Vendors report customers wanting to reorder
Item modifier groups
A pilot vendor with add-ons validates the core
Email / WhatsApp order confirmation
Vendors report customer confusion about order status
Self-serve vendor registration
10+ vendors and founder time becomes the bottleneck
Item image upload
Vendors request it and pilot is validated
Revenue analytics charts
Vendors ask for trend data after 2+ weeks live
Refund UI in dashboard
> 5 refunds per week across all pilots
Browser push notifications
Vendors report kitchen missing orders
COD / pay-after-order
A validated vendor loses > 20% customers over UPI-only
Multi-location outlet hierarchy
A pilot vendor opens a second location


22. Complete Windsurf / Cursor Instructions Reference
System prompt to paste before any build phase:
You are implementing TapEat MVP-1.
This SDD is the single source of truth.
Every decision in this document is final.

BEFORE ANY CODE:

1. Ask for these env vars. Do not proceed without all of them:
   NEXT_PUBLIC_SUPABASE_URL
   NEXT_PUBLIC_SUPABASE_ANON_KEY
   SUPABASE_SERVICE_ROLE_KEY
   NEXT_PUBLIC_RAZORPAY_KEY_ID
   RAZORPAY_KEY_SECRET
   RAZORPAY_WEBHOOK_SECRET
   NEXT_PUBLIC_APP_URL
   CRON_SECRET

2. Confirm:
   - Next.js 14 App Router (NOT Pages Router)
   - TypeScript strict mode ON
   - Supabase project created and 001_mvp1.sql already run
   - Razorpay test keys available
   - You will wait for "proceed" before each new phase

3. Confirm you have read SDD sections:
   3 (scope), 4 (out-of-scope), 9 (functional requirements),
   12 (permissions), 13 (schema), 15 (APIs), 17 (payment flow),
   18 (real-time), 19 (UAT checklist)

Reply READY when confirmed.

STACK RULES (non-negotiable):
- Next.js 14 App Router only
- TypeScript strict — zero any types
- Tailwind CSS only — no CSS modules
- Supabase only — no Firebase, no Prisma
- Razorpay only — no Stripe, no PayU
- Zustand for cart only — no Redux
- React Hook Form + Zod for all forms
- No new packages without founder approval

BUILD RULES:
- One phase at a time. Wait for "proceed."
- End every phase with tsc --noEmit. Report and fix errors.
- Working > polished. Functional flows first.
- Section 4 is the out-of-scope list. Check it before building anything.
- No speculative features. No unsolicited additions.

SECURITY RULES (never skip):
- Customers NEVER write to DB directly
- HMAC signature verified in POST /api/orders
- Prices re-validated from DB in POST /api/orders
- idempotency_key checked before every order insert
- Rate limit: 10/IP/60s on /api/orders and /api/razorpay/create-order
- State machine enforced in PATCH /api/orders/[id]/status
- Cancel returns 403 for staff role
- SUPABASE_SERVICE_ROLE_KEY: server only, never NEXT_PUBLIC_

WHEN TO STOP AND ASK:
- Missing env var
- New package needed not in SDD Section 15.2
- SDD appears contradictory
- Request is in Section 4 (out-of-scope)
- Schema change needed from Section 13
- Never silently add deps or change schema

