TapEat MVP-1 SDD — Part 3 of 5

10. Data Model
10.1 Entity Relationship Overview
vendors
  ├── user_roles          (vendor ↔ auth.users, role scoping)
  ├── categories          (menu grouping)
  ├── menu_items          (belongs to vendor + category)
  ├── service_points      (tables and counters, QR targets)
  └── orders
        ├── order_items   (line items, price denormalised)
        ├── audit_log     (status change history)
        └── payment_events (webhook dedup log)

10.2 Full Schema
sql
-- ─────────────────────────────────────────────
-- FILE: supabase/migrations/001_mvp1.sql
-- Run this in Supabase SQL Editor before dev starts.
-- Do not rename columns — referenced directly in code.
-- ─────────────────────────────────────────────

create extension if not exists "uuid-ossp";

-- ── VENDORS ──────────────────────────────────
create table vendors (
  id          uuid primary key default uuid_generate_v4(),
  name        text not null,
  slug        text not null unique,   -- URL-safe e.g. "my-chai"
  description text,
  phone       text,
  address     text,
  is_active   boolean not null default true,
  created_at  timestamptz not null default now()
);

-- ── USER ROLES ────────────────────────────────
create table user_roles (
  id          uuid primary key default uuid_generate_v4(),
  user_id     uuid not null references auth.users(id) on delete cascade,
  vendor_id   uuid not null references vendors(id) on delete cascade,
  role        text not null default 'staff',
  -- role: owner | manager | staff
  created_at  timestamptz not null default now(),
  unique(user_id, vendor_id)
);

-- ── CATEGORIES ────────────────────────────────
create table categories (
  id          uuid primary key default uuid_generate_v4(),
  vendor_id   uuid not null references vendors(id) on delete cascade,
  name        text not null,
  sort_order  int not null default 0,
  created_at  timestamptz not null default now()
);

-- ── MENU ITEMS ────────────────────────────────
create table menu_items (
  id              uuid primary key default uuid_generate_v4(),
  vendor_id       uuid not null references vendors(id) on delete cascade,
  category_id     uuid references categories(id) on delete set null,
  name            text not null,
  description     text,
  price           numeric(10,2) not null check (price >= 0),
  is_veg          boolean not null default true,
  is_available    boolean not null default true,
  is_recommended  boolean not null default false,
  sort_order      int not null default 0,
  created_at      timestamptz not null default now(),
  updated_at      timestamptz not null default now()
);

-- ── SERVICE POINTS (tables + counters) ────────
create table service_points (
  id            uuid primary key default uuid_generate_v4(),
  vendor_id     uuid not null references vendors(id) on delete cascade,
  label         text not null,         -- "T1", "T2", "Counter A"
  service_mode  text not null default 'dine_in',
  -- service_mode: dine_in | counter_pickup
  section       text,                  -- "Ground Floor", "Rooftop"
  is_active     boolean not null default true,
  created_at    timestamptz not null default now(),
  unique(vendor_id, label)
);

-- ── ORDERS ────────────────────────────────────
create table orders (
  id                   uuid primary key default uuid_generate_v4(),
  order_number         text not null,
  vendor_id            uuid not null references vendors(id),
  service_point_id     uuid references service_points(id),
  service_point_label  text,           -- denormalised at creation
  service_mode         text not null default 'dine_in',
  status               text not null default 'placed',
  -- placed|accepted|preparing|ready|completed|rejected|cancelled
  payment_status       text not null default 'pending',
  -- pending|paid|failed|refunded
  total_amount         numeric(10,2) not null,
  razorpay_order_id    text unique,
  razorpay_payment_id  text,
  razorpay_signature   text,
  idempotency_key      text unique not null,
  customer_name        text,
  customer_phone       text,
  notes                text,
  rejection_reason     text,
  created_at           timestamptz not null default now(),
  updated_at           timestamptz not null default now()
);

-- ── ORDER ITEMS ───────────────────────────────
create table order_items (
  id            uuid primary key default uuid_generate_v4(),
  order_id      uuid not null references orders(id) on delete cascade,
  menu_item_id  uuid references menu_items(id) on delete set null,
  name          text not null,        -- denormalised at creation
  price         numeric(10,2) not null, -- price at time of order
  quantity      int not null check (quantity > 0),
  subtotal      numeric(10,2) not null, -- price × quantity
  created_at    timestamptz not null default now()
);

-- ── PAYMENT EVENTS (webhook dedup) ────────────
create table payment_events (
  id           uuid primary key default uuid_generate_v4(),
  event_id     text not null unique,   -- Razorpay event ID
  event_type   text not null,
  order_id     uuid references orders(id),
  payload      jsonb not null,
  processed    boolean not null default false,
  created_at   timestamptz not null default now()
);

-- ── AUDIT LOG ─────────────────────────────────
create table audit_log (
  id           uuid primary key default uuid_generate_v4(),
  vendor_id    uuid references vendors(id),
  user_id      uuid references auth.users(id),
  action       text not null,
  -- status_change | auto_cancel | item_update | staff_invite | staff_remove
  entity       text not null,          -- orders | menu_items | user_roles
  entity_id    uuid,
  old_value    text,
  new_value    text,
  ip_address   text,
  created_at   timestamptz not null default now()
);

10.3 Triggers
sql
-- Auto-generate order_number per vendor
create or replace function generate_order_number()
returns trigger as $$
declare n int;
begin
  select coalesce(
    max(cast(split_part(order_number, '-', 2) as int)), 1000
  ) + 1
  into n
  from orders
  where vendor_id = new.vendor_id;
  new.order_number = 'ORD-' || n;
  return new;
end;
$$ language plpgsql;

create trigger trg_order_number
  before insert on orders
  for each row execute function generate_order_number();

-- Auto-update updated_at
create or replace function fn_updated_at()
returns trigger as $$
begin
  new.updated_at = now();
  return new;
end;
$$ language plpgsql;

create trigger trg_menu_updated
  before update on menu_items
  for each row execute function fn_updated_at();

create trigger trg_orders_updated
  before update on orders
  for each row execute function fn_updated_at();

10.4 Row-Level Security
sql
-- Enable RLS on all tables
alter table vendors          enable row level security;
alter table user_roles       enable row level security;
alter table categories       enable row level security;
alter table menu_items       enable row level security;
alter table service_points   enable row level security;
alter table orders           enable row level security;
alter table order_items      enable row level security;
alter table payment_events   enable row level security;
alter table audit_log        enable row level security;

-- Helper: return vendor IDs for current authenticated user
create or replace function my_vendor_ids()
returns setof uuid as $$
  select vendor_id
  from user_roles
  where user_id = auth.uid();
$$ language sql security definer;

-- PUBLIC READ (customer menu page — no auth)
create policy "menu_public_read"
  on menu_items for select using (true);

create policy "cats_public_read"
  on categories for select using (true);

create policy "sp_public_read"
  on service_points for select using (true);

create policy "vendors_public_read"
  on vendors for select using (true);

-- VENDOR WRITE (authenticated staff — own vendor only)
create policy "menu_vendor_all"
  on menu_items for all
  using  (vendor_id in (select my_vendor_ids()))
  with check (vendor_id in (select my_vendor_ids()));

create policy "cats_vendor_all"
  on categories for all
  using  (vendor_id in (select my_vendor_ids()))
  with check (vendor_id in (select my_vendor_ids()));

create policy "sp_vendor_all"
  on service_points for all
  using  (vendor_id in (select my_vendor_ids()))
  with check (vendor_id in (select my_vendor_ids()));

-- ORDERS — no direct client write
-- All order inserts go through service role in API routes
-- Vendor staff can SELECT their own orders only
create policy "orders_vendor_read"
  on orders for select
  using (vendor_id in (select my_vendor_ids()));

-- AUDIT LOG — vendor staff read their own
create policy "audit_vendor_read"
  on audit_log for select
  using (vendor_id in (select my_vendor_ids()));

-- Enable Realtime for orders:
-- Supabase Dashboard → Database → Replication → Toggle orders ON

10.5 TypeScript Types
typescript
// FILE: types/index.ts

export type ServiceMode    = 'dine_in' | 'counter_pickup';
export type UserRole       = 'owner' | 'manager' | 'staff';
export type OrderStatus    = 'placed' | 'accepted' | 'preparing' |
                             'ready' | 'completed' | 'rejected' | 'cancelled';
export type PaymentStatus  = 'pending' | 'paid' | 'failed' | 'refunded';

export interface Vendor {
  id: string; name: string; slug: string;
  description: string | null; phone: string | null;
  address: string | null; is_active: boolean; created_at: string;
}

export interface Category {
  id: string; vendor_id: string;
  name: string; sort_order: number;
}

export interface MenuItem {
  id: string; vendor_id: string; category_id: string | null;
  name: string; description: string | null; price: number;
  is_veg: boolean; is_available: boolean;
  is_recommended: boolean; sort_order: number;
  categories?: Category;
}

export interface ServicePoint {
  id: string; vendor_id: string; label: string;
  service_mode: ServiceMode; section: string | null; is_active: boolean;
}

export interface Order {
  id: string; order_number: string; vendor_id: string;
  service_point_id: string | null; service_point_label: string | null;
  service_mode: ServiceMode; status: OrderStatus;
  payment_status: PaymentStatus; total_amount: number;
  razorpay_order_id: string | null; razorpay_payment_id: string | null;
  idempotency_key: string; customer_name: string | null;
  customer_phone: string | null; notes: string | null;
  rejection_reason: string | null;
  created_at: string; updated_at: string;
  order_items?: OrderItem[];
}

export interface OrderItem {
  id: string; order_id: string; menu_item_id: string | null;
  name: string; price: number; quantity: number; subtotal: number;
}

export interface CartItem {
  menuItemId: string; name: string;
  price: number; quantity: number; isVeg: boolean;
}

export interface CartState {
  items: CartItem[];
  vendorId: string | null;
  servicePointId: string | null;
  servicePointLabel: string | null;
  serviceMode: ServiceMode | null;
  addItem: (item: Omit<CartItem, 'quantity'>) => void;
  removeItem: (menuItemId: string) => void;
  updateQty: (menuItemId: string, qty: number) => void;
  clearCart: () => void;
  setSession: (
    vendorId: string,
    servicePointId: string | null,
    servicePointLabel: string | null,
    serviceMode: ServiceMode
  ) => void;
  total: () => number;
  itemCount: () => number;
}

11. API List
11.1 Route Reference
Method
Route
Auth
Purpose
GET
/api/menu/[vendorSlug]?t={uuid}
None
Public menu: vendor + service point + categories + items
POST
/api/razorpay/create-order
None (rate limited)
Create Razorpay payment order
POST
/api/orders
None (server-validated)
Create order: sig verify + price validate + idempotency
GET
/api/orders/[id]/status
None
Customer polls order status
PATCH
/api/orders/[id]/status
Staff / Manager / Owner
Update status via state machine
PATCH
/api/menu-items/[id]/availability
Manager / Owner
Toggle is_available immediately
POST
/api/razorpay/webhook
Sig verified
Handle payment.captured / payment.failed
POST
/api/admin/invite-staff
Owner only
Invite user + insert user_roles
GET
/api/health
None
DB ping + timestamp
GET
/api/cron/auto-cancel
x-cron-secret header
Cancel stale placed orders


11.2 API Contracts
GET /api/menu/[vendorSlug]
typescript
// Query params: t = service_point_uuid (optional)

// Response 200:
{
  vendor: Vendor;
  servicePoint: ServicePoint | null;
  categories: Category[];
  items: MenuItem[];  // all available items, ordered by sort_order
}

// Response 404: vendor slug not found
{ error: 'Vendor not found', code: 'VENDOR_NOT_FOUND' }

POST /api/razorpay/create-order
typescript
// Request:
{
  amount: number;    // in paise (INR × 100)
  vendorId: string;  // uuid
}

// Response 200:
{ rzpOrderId: string; amount: number; }

// Response 429: rate limit exceeded
{ error: 'Too many requests', code: 'RATE_LIMITED' }

POST /api/orders — All 6 Steps Mandatory
typescript
// Request (CreateOrderSchema):
{
  vendorId:          string;  // uuid
  servicePointId:    string | null;
  servicePointLabel: string | null;
  serviceMode:       'dine_in' | 'counter_pickup';
  items: Array<{
    menuItemId: string;
    name:       string;
    price:      number;
    quantity:   number;
  }>;
  totalAmount:         number;
  idempotencyKey:      string;  // uuid v4
  razorpayOrderId:     string;
  razorpayPaymentId:   string;
  razorpaySignature:   string;
  customerName?:       string;
  customerPhone?:      string;
  notes?:              string;
}

// Server steps (all mandatory, in order):
// 1. Rate limit: 10/IP/60s → 429
// 2. Zod validate → 400
// 3. idempotency_key check → 409 + { orderNumber } if exists
// 4. HMAC verify: sha256(KEY_SECRET, rzpOrderId + '|' + paymentId) → 400
// 5. Price re-fetch from DB, compare submitted → 422 if any diff > ₹0
// 6. Insert order + order_items + audit_log (service role)

// Response 200:
{ orderId: string; orderNumber: string; }

// Response 409 (duplicate):
{ error: 'Order already exists', code: 'DUPLICATE_ORDER', orderNumber: string }

GET /api/orders/[id]/status
typescript
// [id] = order_number (e.g. ORD-1042)

// Response 200:
{
  status:            OrderStatus;
  serviceMode:       ServiceMode;
  servicePointLabel: string | null;
  rejectionReason:   string | null;
}

// Response 404:
{ error: 'Order not found', code: 'ORDER_NOT_FOUND' }

PATCH /api/orders/[id]/status
typescript
// Request:
{
  status:           OrderStatus;
  rejectionReason?: string;  // required when status = 'rejected'
}

// Server checks:
// 1. Session valid → 401
// 2. User has role for vendor → 403
// 3. status = 'cancelled' and role = 'staff' → 403
// 4. Transition valid per state machine → 422
// 5. status = 'rejected' and no rejectionReason → 400
// 6. Update order + write audit_log

// Response 200:
{ status: OrderStatus; updatedAt: string; }

PATCH /api/menu-items/[id]/availability
typescript
// Request:
{ isAvailable: boolean; }

// Auth: Manager or Owner only → 403 otherwise

// Response 200:
{ id: string; isAvailable: boolean; updatedAt: string; }

POST /api/razorpay/webhook
typescript
// Headers: x-razorpay-signature (verified against raw body)

// Server steps:
// 1. Read raw body (before JSON.parse)
// 2. Verify HMAC-SHA256 with RAZORPAY_WEBHOOK_SECRET → 400 if invalid
// 3. Check payment_events.event_id for duplicate → return 200 silently
// 4. Insert payment_events record
// 5. On payment.captured: update orders.payment_status = 'paid'
// 6. On payment.failed: update orders.payment_status = 'failed'
// Always return 200 (Razorpay retries on non-200)

POST /api/admin/invite-staff
typescript
// Request:
{ email: string; role: 'manager' | 'staff'; }

// Auth: Owner only → 403 for manager or staff

// Server:
// 1. Verify caller is owner for this vendor
// 2. supabase.auth.admin.inviteUserByEmail(email)
// 3. Insert user_roles { user_id, vendor_id, role } immediately
// 4. Write audit_log entry

// Response 200:
{ message: 'Invitation sent' }

// Response 409: email already has a role for this vendor
{ error: 'User already has access', code: 'DUPLICATE_ROLE' }

11.3 Zod Validation Schemas
typescript
// FILE: lib/validations.ts

import { z } from 'zod';

export const CreateOrderSchema = z.object({
  vendorId:          z.string().uuid(),
  servicePointId:    z.string().uuid().nullable(),
  servicePointLabel: z.string().max(50).nullable(),
  serviceMode:       z.enum(['dine_in', 'counter_pickup']),
  items: z.array(z.object({
    menuItemId: z.string().uuid(),
    name:       z.string().max(200),
    price:      z.number().positive().max(100_000),
    quantity:   z.number().int().min(1).max(99),
  })).min(1).max(50),
  totalAmount:        z.number().positive().max(1_000_000),
  idempotencyKey:     z.string().uuid(),
  razorpayOrderId:    z.string().min(1),
  razorpayPaymentId:  z.string().min(1),
  razorpaySignature:  z.string().min(1),
  customerName:       z.string().max(100).optional(),
  customerPhone:      z.string().max(15).optional(),
  notes:              z.string().max(500).optional(),
});

export const UpdateOrderStatusSchema = z.object({
  status:          z.enum([
    'accepted','preparing','ready','completed','rejected','cancelled'
  ]),
  rejectionReason: z.string().max(200).optional(),
});

export const MenuItemSchema = z.object({
  name:          z.string().min(2).max(200),
  description:   z.string().max(500).optional(),
  price:         z.coerce.number().min(0).max(100_000),
  categoryId:    z.string().uuid().nullable().optional(),
  isVeg:         z.boolean().default(true),
  isAvailable:   z.boolean().default(true),
  isRecommended: z.boolean().default(false),
});

export const InviteStaffSchema = z.object({
  email: z.string().email(),
  role:  z.enum(['manager', 'staff']),
});

export const CreateRazorpayOrderSchema = z.object({
  amount:   z.number().int().positive(),  // paise
  vendorId: z.string().uuid(),
});

12. Role Permissions Matrix
12.1 Complete Action-Level Permissions
Action
Customer
Owner
Manager
Staff
Ordering








View public menu
✅
✅
✅
✅
Place order
✅
—
—
—
Pay via UPI
✅
—
—
—
Track order status
✅
—
—
—
Kitchen








View live order queue
—
✅
✅
✅
Accept order
—
✅
✅
✅
Mark preparing
—
✅
✅
✅
Mark ready
—
✅
✅
✅
Mark complete
—
✅
✅
✅
Reject order (reason required)
—
✅
✅
✅
Cancel order
—
✅
✅
❌ 403
Menu








Add / edit / delete items
—
✅
✅
—
Update prices
—
✅
✅
—
Toggle availability (live)
—
✅
✅
—
Manage categories
—
✅
✅
—
Tables








Create service points
—
✅
✅
—
Download QR codes
—
✅
✅
—
Deactivate service points
—
✅
✅
—
Orders








View order history
—
✅
✅
—
View daily revenue
—
✅
✅
—
View weekly / monthly revenue
—
✅
✅
—
Export CSV
—
✅
✅
—
Staff and Settings








Invite staff
—
✅
❌ 403
❌ 403
Remove staff
—
✅
❌ 403
❌ 403
Assign roles
—
✅
❌ 403
❌ 403
Edit vendor settings
—
✅
❌ 403
❌ 403


12.2 Middleware Route Guards
typescript
// FILE: middleware.ts

const ROLE_ROUTES: Record<string, UserRole[]> = {
  '/kitchen':               ['owner', 'manager', 'staff'],
  '/dashboard':             ['owner', 'manager'],
  '/dashboard/menu':        ['owner', 'manager'],
  '/dashboard/tables':      ['owner', 'manager'],
  '/dashboard/orders':      ['owner', 'manager'],
  '/dashboard/staff':       ['owner'],          // Owner ONLY
  '/dashboard/settings':    ['owner'],          // Owner ONLY
};

// Per request:
// 1. Get session from Supabase SSR cookie
// 2. No session → redirect /login
// 3. Query user_roles for user_id → { role, vendor_id }
// 4. Role not in ROLE_ROUTES[pathname] → redirect /unauthorized
// 5. Set x-user-role + x-vendor-id headers for downstream

13. Admin and Outlet Controls
13.1 Vendor Operational Controls
Control
Route
Role
How
Toggle item availability
/dashboard/menu
Owner, Manager
Switch per item. PATCH /api/menu-items/{id}/availability. Immediate save.
Add / edit / delete menu item
/dashboard/menu
Owner, Manager
Dialog form. POST / PATCH / DELETE /api/menu-items.
Manage categories
/dashboard/menu
Owner, Manager
Inline input. Delete blocked if items assigned.
Create service point
/dashboard/tables
Owner, Manager
Form → POST /api/service-points.
Deactivate service point
/dashboard/tables
Owner, Manager
Confirmation → PATCH /api/service-points/{id} { isActive: false }.
Download QR PNG
/dashboard/tables
Owner, Manager
Client-side canvas → PNG download.
Accept / prepare / complete orders
/kitchen
Owner, Manager, Staff
PATCH /api/orders/{id}/status.
Reject order
/kitchen
Owner, Manager, Staff
Modal → preset reason → PATCH.
Cancel order
/kitchen
Owner, Manager only
Button invisible for Staff. PATCH returns 403 for Staff.
View live stats
/dashboard
Owner, Manager
Client-side query. 5-minute refresh.
Filter + export orders
/dashboard/orders
Owner, Manager
Date + status filter. CSV client-side.
Invite staff
/dashboard/staff
Owner only
POST /api/admin/invite-staff.
Remove staff
/dashboard/staff
Owner only
Delete user_roles record.
Edit vendor profile
/dashboard/settings
Owner only
PATCH /api/vendors/{id}.


13.2 System (Automated) Controls
Control
Trigger
Action
Auto-cancel stale orders
Vercel Cron every 10 min
Cancel placed orders > 30 min old. Write audit log per order.
Webhook payment update
Razorpay payment.captured
Update orders.payment_status = 'paid'. Idempotent.
Webhook failure update
Razorpay payment.failed
Update orders.payment_status = 'failed'. Idempotent.
Order number generation
DB trigger on INSERT
ORD-{n} per vendor sequence starting at 1001.
updated_at maintenance
DB trigger on UPDATE
Auto-set on menu_items and orders.


13.3 Founder Admin Controls (MVP-1)
Founder uses Supabase Studio directly. No admin UI in MVP-1.
Task
How (Supabase Studio)
Create new vendor
INSERT into vendors
Create vendor owner account
Supabase Auth → Add User → INSERT into user_roles
Import menu items
SQL INSERT or CSV import via Studio
Create service points
SQL INSERT into service_points
Process refund
Razorpay Dashboard → manual refund
View any vendor's data
Direct SQL query (no RLS restriction for Studio)
Deactivate vendor
UPDATE vendors SET is_active = false WHERE id = '...'


13.4 Demo Seed Data
sql
-- Run after schema migration

-- Vendor
insert into vendors (name, slug, description, address)
values (
  'My Chai', 'my-chai',
  'Best chai and snacks on campus',
  'HITEC City, Hyderabad'
);

-- Categories
insert into categories (vendor_id, name, sort_order)
select id, 'Beverages', 1 from vendors where slug = 'my-chai';

insert into categories (vendor_id, name, sort_order)
select id, 'Snacks', 2 from vendors where slug = 'my-chai';

-- Menu items
insert into menu_items
  (vendor_id, category_id, name, description, price, is_veg, is_recommended)
select v.id, c.id, 'Green Tea With Honey',
  '100ml · light and refreshing', 18, true, true
from vendors v
join categories c on c.vendor_id = v.id and c.name = 'Beverages'
where v.slug = 'my-chai';

insert into menu_items
  (vendor_id, category_id, name, description, price, is_veg, is_recommended)
select v.id, c.id, 'Coffee',
  '100ml · strong filter coffee', 16, true, true
from vendors v
join categories c on c.vendor_id = v.id and c.name = 'Beverages'
where v.slug = 'my-chai';

insert into menu_items
  (vendor_id, category_id, name, description, price, is_veg)
select v.id, c.id, 'Tea',
  '100ml · classic masala chai', 14, true
from vendors v
join categories c on c.vendor_id = v.id and c.name = 'Beverages'
where v.slug = 'my-chai';

insert into menu_items
  (vendor_id, category_id, name, description, price, is_veg, is_recommended)
select v.id, c.id, 'Vada Pav',
  'with green chutney', 25, true, true
from vendors v
join categories c on c.vendor_id = v.id and c.name = 'Snacks'
where v.slug = 'my-chai';

insert into menu_items
  (vendor_id, category_id, name, description, price, is_veg)
select v.id, c.id, 'Irani Biscuits',
  'crispy, pairs with chai', 4, true
from vendors v
join categories c on c.vendor_id = v.id and c.name = 'Snacks'
where v.slug = 'my-chai';

-- Service points
insert into service_points (vendor_id, label, service_mode)
select id, 'Counter A', 'counter_pickup'
from vendors where slug = 'my-chai';

insert into service_points (vendor_id, label, service_mode, section)
select id, 'T1', 'dine_in', 'Seating Area'
from vendors where slug = 'my-chai';

insert into service_points (vendor_id, label, service_mode, section)
select id, 'T2', 'dine_in', 'Seating Area'
from vendors where slug = 'my-chai';

-- After inserting, retrieve UUIDs for QR generation:
-- select id, label from service_points
-- where vendor_id = (select id from vendors where slug = 'my-chai');
--
-- QR URL: {APP_URL}/menu?v=my-chai&t={service_point_uuid}
--
-- Create owner user: Supabase Dashboard → Auth → Add User
-- Then: insert into user_roles (user_id, vendor_id, role)
--       values ('{auth_uuid}', '{vendor_uuid}', 'owner');



