TapEat MVP-1 SDD — Part 4 of 5

14. Architecture
14.1 System Overview
┌─────────────────────────────────────────────────────────────────┐
│                        VERCEL (Edge + Serverless)               │
│                                                                 │
│  ┌─────────────────┐    ┌──────────────────────────────────┐   │
│  │  Next.js 14      │    │         API Routes               │   │
│  │  App Router      │    │  /api/menu/[vendorSlug]          │   │
│  │                 │    │  /api/orders                     │   │
│  │  SSR pages:     │    │  /api/orders/[id]/status         │   │
│  │  /menu          │    │  /api/razorpay/create-order      │   │
│  │                 │    │  /api/razorpay/webhook           │   │
│  │  Client pages:  │    │  /api/menu-items/[id]/availability│  │
│  │  /kitchen       │    │  /api/admin/invite-staff         │   │
│  │  /dashboard/*   │    │  /api/cron/auto-cancel           │   │
│  │  /order-status  │    │  /api/health                     │   │
│  └────────┬────────┘    └──────────────┬───────────────────┘   │
│           │                            │                        │
└───────────┼────────────────────────────┼────────────────────────┘
            │                            │
            │         ┌──────────────────▼──────────────────┐
            │         │         SUPABASE                     │
            │         │                                      │
            │         │  ┌──────────┐  ┌──────────────────┐ │
            │         │  │PostgreSQL│  │  Supabase Auth   │ │
            │         │  │  + RLS   │  │  (email/password)│ │
            │         │  └──────────┘  └──────────────────┘ │
            │         │                                      │
            │         │  ┌──────────┐  ┌──────────────────┐ │
            │         │  │Realtime  │  │  Storage         │ │
            │         │  │WebSocket │  │  (menu-images    │ │
            │         │  │(orders)  │  │   ready for v2)  │ │
            │         │  └──────────┘  └──────────────────┘ │
            │         └─────────────────────────────────────┘
            │
            │         ┌──────────────────────────────────────┐
            └────────►│         RAZORPAY                     │
                      │  Standard Checkout JS                │
                      │  Orders API                          │
                      │  Webhooks (payment.captured/failed)  │
                      └──────────────────────────────────────┘

14.2 Request Flow by Scenario
Customer Orders (Critical Path)
1. Customer scans QR
   └─ GET /menu?v=my-chai&t={uuid}
   └─ Next.js server component fetches:
      - vendors WHERE slug = 'my-chai'
      - service_points WHERE id = {uuid}
      - categories WHERE vendor_id = X
      - menu_items WHERE vendor_id = X AND is_available = true
   └─ HTML rendered server-side → sent to browser
   └─ Hydration: Zustand cart initialised, params stored in store

2. Customer pays
   └─ POST /api/razorpay/create-order → Razorpay API → rzpOrderId
   └─ Razorpay Checkout JS opens (loaded dynamically)
   └─ Customer pays via UPI
   └─ Razorpay → handler(paymentId, orderId, signature)
   └─ POST /api/orders (service role, all 6 steps)
   └─ Order row inserted → Supabase Realtime fires INSERT event
   └─ Kitchen WebSocket receives event → card rendered
   └─ Customer navigates to /order-status (polls every 15s)

3. Razorpay webhook (async)
   └─ POST /api/razorpay/webhook
   └─ Raw body read → HMAC verified → event_id dedup check
   └─ UPDATE orders SET payment_status = 'paid'
Kitchen Status Update
Staff taps "Mark Ready"
   └─ Optimistic UI: local state updated immediately
   └─ PATCH /api/orders/{id}/status { status: 'ready' }
   └─ Server: session check → role check → transition check
   └─ UPDATE orders SET status = 'ready'
   └─ INSERT audit_log
   └─ Supabase Realtime fires UPDATE event
   └─ All connected kitchen tablets update in-place
   └─ Customer /order-status poll (next 15s tick) returns 'ready'
   └─ Customer page: full-screen green + audio alert

14.3 Folder Structure
tapeat/
├── app/
│   ├── (customer)/
│   │   ├── menu/
│   │   │   └── page.tsx              # SSR — server component
│   │   └── order-status/
│   │       └── page.tsx              # CSR — polling
│   ├── (kitchen)/
│   │   └── kitchen/
│   │       └── page.tsx              # CSR — real-time
│   ├── (vendor)/
│   │   ├── login/
│   │   │   └── page.tsx
│   │   └── dashboard/
│   │       ├── page.tsx              # home stats
│   │       ├── menu/
│   │       │   └── page.tsx
│   │       ├── tables/
│   │       │   └── page.tsx
│   │       ├── orders/
│   │       │   └── page.tsx
│   │       ├── staff/
│   │       │   └── page.tsx          # Owner only
│   │       └── settings/
│   │           └── page.tsx          # Owner only
│   ├── api/
│   │   ├── menu/[vendorSlug]/route.ts
│   │   ├── orders/
│   │   │   ├── route.ts
│   │   │   └── [id]/status/route.ts
│   │   ├── menu-items/
│   │   │   └── [id]/availability/route.ts
│   │   ├── razorpay/
│   │   │   ├── create-order/route.ts
│   │   │   └── webhook/route.ts
│   │   ├── admin/
│   │   │   └── invite-staff/route.ts
│   │   ├── cron/
│   │   │   └── auto-cancel/route.ts
│   │   └── health/route.ts
│   ├── layout.tsx                    # Sora font + env validation
│   └── globals.css
│
├── components/
│   ├── customer/
│   │   ├── MenuHeader.tsx
│   │   ├── CategoryTabs.tsx
│   │   ├── RecommendedStrip.tsx
│   │   ├── MenuItemCard.tsx
│   │   ├── CartBar.tsx
│   │   └── CartModal.tsx             # Razorpay + POST /api/orders
│   ├── kitchen/
│   │   ├── KitchenHeader.tsx
│   │   ├── OrderFilters.tsx
│   │   ├── OrderCard.tsx
│   │   ├── StatusBadge.tsx
│   │   └── RejectionModal.tsx
│   ├── vendor/
│   │   ├── VendorNav.tsx             # role prop → conditional links
│   │   ├── StatsCard.tsx
│   │   ├── MenuItemForm.tsx
│   │   ├── ServicePointCard.tsx
│   │   └── QRDownload.tsx
│   └── shared/
│       ├── VegIndicator.tsx
│       ├── SkeletonCard.tsx
│       ├── ErrorBoundary.tsx
│       └── EmptyState.tsx
│
├── hooks/
│   ├── useOrders.ts                  # Supabase real-time + polling fallback
│   └── useVendorSession.ts           # vendor_id + role from session
│
├── lib/
│   ├── supabase/
│   │   ├── client.ts                 # browser client
│   │   └── server.ts                 # server client (SSR + API routes)
│   ├── razorpay.ts                   # verifyPaymentSignature, verifyWebhookSig
│   ├── rateLimit.ts                  # in-memory 10/IP/60s
│   ├── validations.ts                # all Zod schemas
│   └── utils.ts                      # cn(), formatINR(), sanitize(), timeAgo()
│
├── store/
│   └── cartStore.ts                  # Zustand cart + session state
│
├── types/
│   └── index.ts                      # all TypeScript types
│
├── middleware.ts                     # role-based route protection
├── supabase/
│   └── migrations/
│       └── 001_mvp1.sql
├── public/
│   └── sounds/
│       └── order-alert.mp3
├── vercel.json                       # cron config
├── tailwind.config.ts
├── next.config.ts
├── .env.local
└── .env.example

15. Recommended Tech Stack
15.1 Stack (Locked — No Substitutions)
Layer
Technology
Version
Reason
Framework
Next.js App Router
14+
SSR for menu + API routes + client components in one project
Language
TypeScript
5.4+ strict
Zero any. Zero untyped assertions.
Styling
Tailwind CSS
v3+
No CSS modules, no styled-components
UI primitives
shadcn/ui
Latest
Dialogs, tables, forms, badges — not a full design system
Database
Supabase (PostgreSQL 15)
Latest
RLS + Realtime + Auth + Storage in one service
Auth
Supabase Auth
Built-in
Email+password + magic link for staff invites
Payments
Razorpay Standard Checkout
v2
UPI-first, India-native
State
Zustand
4.5+
Cart only. No Redux. No Context API for global state.
Forms
React Hook Form + Zod
Latest
All forms. All validation.
QR
qrcode.react
3.1+
Client-side generation + PNG download
Toasts
Sonner
1.4+
In-app notifications only
Icons
Lucide React
0.378+
Single icon library
Font
Sora (Google Fonts)
All weights
All UI surfaces
Hosting
Vercel
Hobby/Pro
Auto-deploy + Cron support

15.2 Required Packages
bash
# Core
next react react-dom typescript

# Supabase
@supabase/supabase-js @supabase/ssr

# State + forms
zustand react-hook-form @hookform/resolvers zod

# UI
sonner lucide-react qrcode.react

# Utilities
clsx tailwind-merge date-fns

# Server types only
razorpay
15.3 What Not to Add
❌ Do Not Add
Use Instead
Prisma or Drizzle
Supabase client directly
Redux or Jotai
Zustand (cart only)
Axios
Native fetch
Formik
React Hook Form
Stripe or PayU
Razorpay only
Firebase
Supabase only
Any analytics SDK
None in MVP-1
Any email SDK
None in MVP-1


16. Integrations
16.1 Supabase
Client Setup
typescript
// FILE: lib/supabase/client.ts
import { createBrowserClient } from '@supabase/ssr';

export const createClient = () =>
  createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
typescript
// FILE: lib/supabase/server.ts
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';

export const createClient = () => {
  const cookieStore = cookies();
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        get: (name) => cookieStore.get(name)?.value,
        set: (name, value, options) =>
          cookieStore.set({ name, value, ...options }),
        remove: (name, options) =>
          cookieStore.set({ name, value: '', ...options }),
      },
    }
  );
};
Service Role Client (API routes and webhooks only)
typescript
// Used only in server-side API routes
// Never import this in any component or client-side file
import { createClient as createAdminClient } from '@supabase/supabase-js';

export const adminClient = createAdminClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!  // server-only, never NEXT_PUBLIC_
);
Real-Time Subscription
typescript
// FILE: hooks/useOrders.ts (key parts)

export function useOrders(vendorId: string) {
  const [orders, setOrders] = useState<Order[]>([]);
  const [connectionStatus, setConnectionStatus] =
    useState<'SUBSCRIBED' | 'RECONNECTING' | 'POLLING'>('RECONNECTING');
  const pollingRef = useRef<NodeJS.Timeout>();

  useEffect(() => {
    const supabase = createClient();

    // Initial fetch
    supabase
      .from('orders')
      .select('*, order_items(*)')
      .eq('vendor_id', vendorId)
      .not('status', 'in', '(completed,rejected,cancelled)')
      .order('created_at', { ascending: true })
      .then(({ data }) => data && setOrders(data));

    const channel = supabase
      .channel('kitchen-' + vendorId)
      .on('postgres_changes', {
        event: 'INSERT', schema: 'public', table: 'orders',
        filter: `vendor_id=eq.${vendorId}`,
      }, (payload) => {
        playOrderAlert();
        setOrders(prev => [payload.new as Order, ...prev]);
      })
      .on('postgres_changes', {
        event: 'UPDATE', schema: 'public', table: 'orders',
        filter: `vendor_id=eq.${vendorId}`,
      }, (payload) => {
        setOrders(prev => prev.map(o =>
          o.id === payload.new.id
            ? { ...o, ...(payload.new as Order) }
            : o
        ));
      })
      .subscribe((status) => {
        if (status === 'SUBSCRIBED') {
          setConnectionStatus('SUBSCRIBED');
          clearInterval(pollingRef.current);
        } else {
          setConnectionStatus('RECONNECTING');
          startPollingFallback();
        }
      });

    return () => { supabase.removeChannel(channel); };
  }, [vendorId]);

  // Polling fallback: refetch active orders every 15s
  function startPollingFallback() {
    clearInterval(pollingRef.current);
    pollingRef.current = setInterval(async () => {
      const supabase = createClient();
      const { data } = await supabase
        .from('orders')
        .select('*, order_items(*)')
        .eq('vendor_id', vendorId)
        .not('status', 'in', '(completed,rejected,cancelled)')
        .order('created_at', { ascending: true });
      if (data) setOrders(data);
    }, 15_000);
  }

  return { orders, connectionStatus };
}

16.2 Razorpay
Signature Verification
typescript
// FILE: lib/razorpay.ts
import crypto from 'crypto';

export function verifyPaymentSignature(
  razorpayOrderId: string,
  razorpayPaymentId: string,
  signature: string
): boolean {
  const body = razorpayOrderId + '|' + razorpayPaymentId;
  const expected = crypto
    .createHmac('sha256', process.env.RAZORPAY_KEY_SECRET!)
    .update(body)
    .digest('hex');
  return expected === signature;
}

export function verifyWebhookSignature(
  rawBody: string,
  signature: string
): boolean {
  const expected = crypto
    .createHmac('sha256', process.env.RAZORPAY_WEBHOOK_SECRET!)
    .update(rawBody)
    .digest('hex');
  return expected === signature;
}
Checkout Integration (CartModal)
typescript
// Key steps in CartModal.tsx

async function handlePayment() {
  setLoading(true);

  // Step 1: generate idempotency key before anything else
  const idempotencyKey = crypto.randomUUID();
  sessionStorage.setItem('tapeat_idem_key', idempotencyKey);

  // Step 2: load Razorpay script
  const loaded = await loadRazorpayScript();
  if (!loaded) {
    toast.error('Payment could not load. Check your connection.');
    setLoading(false);
    return;
  }

  // Step 3: create Razorpay order
  const res = await fetch('/api/razorpay/create-order', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ amount: total * 100, vendorId }),
  });
  const { rzpOrderId, amount } = await res.json();

  // Step 4: open checkout
  const options = {
    key: process.env.NEXT_PUBLIC_RAZORPAY_KEY_ID,
    amount,
    currency: 'INR',
    order_id: rzpOrderId,
    name: vendorName,
    description: servicePointLabel ?? 'Food Order',
    prefill: { name: customerName, contact: customerPhone },
    handler: async (response: {
      razorpay_order_id: string;
      razorpay_payment_id: string;
      razorpay_signature: string;
    }) => {
      // Step 5: create order in DB
      const orderRes = await fetch('/api/orders', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          vendorId,
          servicePointId,
          servicePointLabel,
          serviceMode,
          items: cart.items.map(i => ({
            menuItemId: i.menuItemId, name: i.name,
            price: i.price, quantity: i.quantity,
          })),
          totalAmount: total,
          idempotencyKey,
          razorpayOrderId: response.razorpay_order_id,
          razorpayPaymentId: response.razorpay_payment_id,
          razorpaySignature: response.razorpay_signature,
          customerName, customerPhone, notes,
        }),
      });

      if (orderRes.status === 409) {
        // Duplicate — navigate to existing order
        const { orderNumber } = await orderRes.json();
        cart.clearCart();
        router.push(`/order-status?id=${orderNumber}`);
        return;
      }

      const { orderNumber } = await orderRes.json();
      cart.clearCart();
      router.push(`/order-status?id=${orderNumber}`);
    },
    modal: {
      ondismiss: () => {
        setLoading(false);
        toast.info('Payment cancelled. Your cart is saved.');
      },
    },
  };

  const rzp = new (window as any).Razorpay(options);
  rzp.open();
}
Webhook Handler
typescript
// FILE: app/api/razorpay/webhook/route.ts

export async function POST(req: Request) {
  // 1. Read raw body — must happen before any parsing
  const rawBody = await req.text();
  const signature = req.headers.get('x-razorpay-signature') ?? '';

  // 2. Verify signature
  if (!verifyWebhookSignature(rawBody, signature)) {
    return Response.json(
      { error: 'Invalid signature', code: 'INVALID_SIGNATURE' },
      { status: 400 }
    );
  }

  const event = JSON.parse(rawBody);

  // 3. Check for duplicate event
  const { data: existing } = await adminClient
    .from('payment_events')
    .select('id')
    .eq('event_id', event.id)
    .single();

  if (existing) {
    return Response.json({ received: true }); // idempotent 200
  }

  // 4. Log event
  await adminClient.from('payment_events').insert({
    event_id: event.id,
    event_type: event.event,
    payload: event,
  });

  // 5. Handle event types
  const payment = event.payload?.payment?.entity;

  if (event.event === 'payment.captured') {
    await adminClient
      .from('orders')
      .update({ payment_status: 'paid' })
      .eq('razorpay_order_id', payment.order_id);
  }

  if (event.event === 'payment.failed') {
    await adminClient
      .from('orders')
      .update({ payment_status: 'failed' })
      .eq('razorpay_order_id', payment.order_id);
  }

  // Always return 200 — Razorpay retries on non-200
  return Response.json({ received: true });
}

16.3 Rate Limiter
typescript
// FILE: lib/rateLimit.ts
// In-memory. Resets on Vercel cold start.
// Sufficient for MVP-1 load. Upgrade to Redis at scale.

const store = new Map<string, { count: number; resetAt: number }>();

export function rateLimit(
  ip: string,
  limit = 10,
  windowMs = 60_000
): { allowed: boolean } {
  const now = Date.now();
  const entry = store.get(ip);

  if (!entry || now > entry.resetAt) {
    store.set(ip, { count: 1, resetAt: now + windowMs });
    return { allowed: true };
  }

  if (entry.count >= limit) {
    return { allowed: false };
  }

  entry.count++;
  return { allowed: true };
}

// Usage in API route:
// const ip = req.headers.get('x-forwarded-for') ?? '127.0.0.1';
// const { allowed } = rateLimit(ip);
// if (!allowed) return Response.json(
//   { error: 'Too many requests', code: 'RATE_LIMITED' },
//   { status: 429 }
// );

17. Security and Audit Requirements
17.1 Security Checklist
#
Requirement
Implementation
S-01
Service role key server-only
Never in NEXT_PUBLIC_ vars. Only in API routes + webhook.
S-02
Razorpay secrets server-only
RAZORPAY_KEY_SECRET and RAZORPAY_WEBHOOK_SECRET never in client code.
S-03
All API inputs Zod-validated
Every POST / PATCH route has a Zod schema. Unknown fields stripped.
S-04
Payment signature verified
HMAC-SHA256 in POST /api/orders. Mandatory before any DB write.
S-05
Webhook signature verified on raw body
Raw body read before JSON.parse. Verified before any processing.
S-06
Price re-validation server-side
Re-fetch DB prices in POST /api/orders. Reject 422 if any diff > ₹0.
S-07
Idempotency key uniqueness
Unique constraint on idempotency_key. 409 returned on duplicate.
S-08
Webhook event dedup
payment_events.event_id unique constraint. Duplicate returns 200 silently.
S-09
RLS on all tables
No direct client DB writes for orders. Vendor data scoped by my_vendor_ids().
S-10
Rate limiting on public routes
10 req/IP/60s on /api/orders and /api/razorpay/create-order.
S-11
Input sanitisation
Strip HTML from notes, customerName, customerPhone before DB insert.
S-12
Cancel role enforcement
PATCH returns 403 for Staff attempting cancel. Button invisible in UI.
S-13
Staff invite Owner-only
Server checks role === 'owner' before calling Supabase admin API.
S-14
HTTPS enforced
Vercel enforces HTTPS. No HTTP in production.
S-15
No PII in logs
Never log customerPhone, signatures, secrets, or service role key.
S-16
Service point UUID in QR URL
UUID prevents table ID enumeration by incrementing.
S-17
Env var validation on startup
app/layout.tsx throws at boot if any required env var is missing.


17.2 Startup Env Validation
typescript
// FILE: app/layout.tsx (server component)

const REQUIRED_ENV = [
  'NEXT_PUBLIC_SUPABASE_URL',
  'NEXT_PUBLIC_SUPABASE_ANON_KEY',
  'SUPABASE_SERVICE_ROLE_KEY',
  'NEXT_PUBLIC_RAZORPAY_KEY_ID',
  'RAZORPAY_KEY_SECRET',
  'RAZORPAY_WEBHOOK_SECRET',
  'NEXT_PUBLIC_APP_URL',
  'CRON_SECRET',
] as const;

REQUIRED_ENV.forEach((key) => {
  if (!process.env[key]) {
    throw new Error(
      `[TapEat] Missing required environment variable: ${key}`
    );
  }
});

17.3 Input Sanitisation
typescript
// FILE: lib/utils.ts

export function sanitize(input: string): string {
  return input
    .replace(/<[^>]*>/g, '')    // strip HTML tags
    .replace(/&[^;]+;/g, '')    // strip HTML entities
    .trim()
    .slice(0, 500);             // hard cap
}

// Apply before any DB insert on:
// - orders.notes
// - orders.customer_name
// - orders.customer_phone
// - orders.rejection_reason (from preset list — still sanitise)

17.4 Audit Log
Every order status change must write to audit_log. Written server-side using the service role client. Never skipped.
typescript
// Pattern used in PATCH /api/orders/[id]/status

await adminClient.from('audit_log').insert({
  vendor_id:  order.vendor_id,
  user_id:    session.user.id,
  action:     'status_change',
  entity:     'orders',
  entity_id:  order.id,
  old_value:  order.status,
  new_value:  newStatus,
  ip_address: req.headers.get('x-forwarded-for') ?? null,
});
Audit log entries written for:
Action
action value
Order status change
status_change
Auto-cancel by cron
auto_cancel
Menu item created
item_created
Menu item updated
item_updated
Menu item deleted
item_deleted
Staff invited
staff_invited
Staff removed
staff_removed


17.5 Error Response Standard
typescript
// Every API failure returns this shape only.
// No stack traces. No raw DB errors. No internal details.

type ErrorResponse = {
  error: string;   // human-readable
  code:  string;   // machine-readable constant
};

// HTTP codes used:
// 400 — Bad request (Zod fail, bad signature)
// 401 — Unauthenticated (no session, wrong CRON_SECRET)
// 403 — Forbidden (wrong role for route or action)
// 404 — Not found
// 409 — Conflict (duplicate idempotency key)
// 422 — Unprocessable (price mismatch, invalid transition, blocked delete)
// 429 — Rate limited
// 500 — Internal (log server-side, return generic message to client)

17.6 Environment Variables
bash
# FILE: .env.local (never commit)
# FILE: .env.example (commit — values empty)

NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=

# Server-only — never NEXT_PUBLIC_
SUPABASE_SERVICE_ROLE_KEY=

NEXT_PUBLIC_RAZORPAY_KEY_ID=

# Server-only — never NEXT_PUBLIC_
RAZORPAY_KEY_SECRET=
RAZORPAY_WEBHOOK_SECRET=

NEXT_PUBLIC_APP_URL=https://tapeat.in

# Server-only — used to authenticate Vercel Cron calls
CRON_SECRET=

17.7 Vercel Configuration
json
// FILE: vercel.json

{
  "crons": [
    {
      "path": "/api/cron/auto-cancel",
      "schedule": "*/10 * * * *"
    }
  ]
}
Cron handler pattern:
typescript
// FILE: app/api/cron/auto-cancel/route.ts

export async function GET(req: Request) {
  const secret = req.headers.get('x-cron-secret');
  if (secret !== process.env.CRON_SECRET) {
    return Response.json(
      { error: 'Unauthorized', code: 'INVALID_SECRET' },
      { status: 401 }
    );
  }

  const cutoff = new Date(Date.now() - 30 * 60 * 1000).toISOString();

  const { data: staleOrders } = await adminClient
    .from('orders')
    .select('id, vendor_id, order_number')
    .eq('status', 'placed')
    .lt('created_at', cutoff);

  if (!staleOrders?.length) {
    return Response.json({ cancelled: 0 });
  }

  for (const order of staleOrders) {
    await adminClient
      .from('orders')
      .update({ status: 'cancelled' })
      .eq('id', order.id);

    await adminClient.from('audit_log').insert({
      vendor_id:  order.vendor_id,
      action:     'auto_cancel',
      entity:     'orders',
      entity_id:  order.id,
      old_value:  'placed',
      new_value:  'cancelled',
    });
  }

  return Response.json({ cancelled: staleOrders.length });
}



