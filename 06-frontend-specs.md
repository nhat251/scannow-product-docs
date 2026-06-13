# Frontend Specification

## 1. Frontend Split

Current repos:

```text
scan-now-nextjs       Landing + platform admin
scan-now-customer     Tenant portal (dung cho tat ca roles: owner, manager, staff, kitchen, cashier, customer)
```

Official domain mapping:

```text
scannow.site              -> scan-now-nextjs
scannow.site/admin        -> scan-now-nextjs
*.scannow.site            -> scan-now-customer
api.scannow.site          -> backend API
business.scannow.site     -> future/parked business portal, not tenant app
```

`business.scannow.site` khong duoc implement trong MVP. No duoc reserved cho tuong lai. Neu wildcard DNS/deployment da lam domain nay resolve duoc, phai parked hoac reserve `business` trong tenant slug parsing de khong bi xem nhu restaurant slug.

### 1.1 Quality Standards (Linting & TypeScript)

**BAT BUOC**: Toan bo code Frontend (FE) phai tuan thu tuyet doi cac quy tac sau:
- Chay linter (`pnpm run lint` hoac `pnpm run lint:fix`) va build (`pnpm run build` / tsc) truoc khi ket thuc bat ky plan nao.
- Code chi duoc coi la hoan thanh ("pass") khi **khong co bat ky loi (error) hay canh bao (warning) nao**.
- Cam su dung `any` sai muc dich, phai cast type hoac bat loi an toan (vi du: dung `unknown` ket hop `axios.isAxiosError(err)`).

## 2. Landing/Admin FE (`scan-now-nextjs`)

### 2.1 Stack

- Next.js App Router.
- React.
- TypeScript.
- Tailwind.
- next-intl.
- Axios.
- Zustand.
- TanStack Query.
- Resend.
- Lucide icons.

### 2.2 Routes

Current app routes:

```text
/                         redirect to /vi
/[locale]                 landing
/admin                    admin dashboard/login
/admin/restaurants        restaurant management
/admin/restaurants/create create restaurant
/admin/restaurants/[id]   restaurant detail; id may be slug or GUID
/admin/restaurants/[id]/edit
/admin/branches/[branchId]
/admin/branches/[branchId]/categories/[categoryId]
/admin/branches/[branchId]/tables/[tableId]
/admin/menu-items/[menuItemId]
/api/lead-capture
/robots.ts
/sitemap.ts
```

Note:

- `src/proxy.ts` uses next-intl middleware and excludes `/admin` from locale handling.
- Root `/` redirects to default locale, so `scannow.site` becomes `scannow.site/vi`.

### 2.3 Landing requirements

- Use `SITE_CONFIG.baseUrl = https://scannow.site`.
- Support `vi` and `en`.
- Keep `/api/lead-capture` server-only env secrets.
- Validate lead payload.
- Do not leak email configuration errors.

### 2.4 Admin requirements

Admin views should support:

- Login.
- Owner list/create/update/ban/unban.
- Restaurant list/create/update/ban/unban.
- Restaurant detail by slug.
- Branch list by restaurant slug.
- Branch detail by branch slug.
- Branch categories tab.
- Branch menu items tab.
- Branch tables tab.
- Active sessions tab.
- Menu item detail and price history.
- Table detail.

### 2.5 Current admin API mapping

Service file: `src/services/admin.ts`.

Backend alignment:

- `getRestaurantBySlug` maps to `/api/admin/restaurants/by-slug/{slug}`.
- `getRestaurantBranchesBySlug` maps to `/api/admin/restaurants/by-slug/{slug}/branches`.
- `getBranchDetailBySlug` maps to `/api/admin/restaurants/by-slug/{restaurantSlug}/branches/{branchSlug}`.

This aligns with the audited backend branch `test/deploy`.

### 2.6 Admin FE gaps

- Admin page checks token/login state but does not enforce `ADMIN` role strongly in route guard.
- TableStatus type is numeric in FE, while backend serializes enum strings.
- `SITE_CONFIG.baseUrl` is configured as `https://scannow.site`.
- `NEXT_PUBLIC_API_URL` is configured as `https://api.scannow.site`; deployment env must stay aligned.
- Some admin views are read-only for menu/table/session.
- Admin does not expose reports dashboard despite backend route.

## 3. Tenant/Portal FE (`scan-now-customer`)

### 3.1 Stack

- Next.js App Router.
- React.
- TypeScript.
- Tailwind.
- Axios.
- Zustand.
- TanStack Query.

### 3.2 Current routes

```text
/                         home
/login                    login
/admin/dashboard          placeholder/protected

# Owner portal
/owner/dashboard          reports overview dashboard
/owner/restaurant         owner restaurant page
/owner/users              owner user management
/owner/settings           branch settings (VAT/service charge, PayOS payment-config, paper voucher)
/owner/branches           owner branches list
/owner/branches/create    create branch
/owner/branches/[id]      branch detail/edit
/owner/branches/[id]/categories[/create | /[categoryId]]   category management
/owner/branches/[id]/menu-items[/create]                   branch menu-item list/create
/owner/branches/[id]/tables[/create | /[tableId]]          table/QR management
/owner/branches/[id]/orders                                branch orders
/owner/menu-items/[menuItemId][/price-history]             menu-item detail + price history

# Manager portal (same set, scoped to managed branch)
/manager/dashboard        reports overview dashboard
/manager/users            manager user management
/manager/settings         branch settings
/manager/orders           branch orders
/manager/branches/[branchId]/categories[/create | /[categoryId]]
/manager/branches/[branchId]/menu-items[/create]
/manager/branches/[branchId]/tables[/create | /[tableId]]
/manager/branches/[branchId]/orders
/manager/menu-items/[menuItemId][/price-history]

# Staff / Kitchen / Branch Manager operations (me/ waiter shell)
/me/branches                              my branches
/me/branches/[branchId]                   branch detail
/me/branches/[branchId]/menu              branch menu + toggle availability
/me/branches/[branchId]/tables            open/close table sessions
/me/branches/[branchId]/orders            branch orders (waiter)
/me/branches/[branchId]/kitchen           kitchen grouped queue
/me/tables/[tableId]                      table detail
/me/menu-items/[menuItemId]               menu-item detail
/staff/dashboard                          legacy placeholder (kept; not the login target)
/kitchen/dashboard                        legacy placeholder (kept; not the login target)

# Cashier portal
/cashier/dashboard        order list/detail + checkout cash/PayOS + voucher + cancel payment
/cashier/orders           cashier orders

# Public customer QR
/tables/[qrCodeToken]                         public QR entry and auto-join
/sessions/[sessionCode]/menu                  session menu + shared cart
/sessions/[sessionCode]/menu-items/[menuItemId]   menu-item detail
/sessions/[sessionCode]/checkout              place order + checkout
/sessions/[sessionCode]/orders/[orderId]      order tracking (realtime)
/payment/return
/payment/cancel

/robots.ts
/sitemap.ts
```

### 3.3 Tenant slug extraction

File: `src/lib/tenant.ts`.

Behavior:

- Reads `window.location.hostname`.
- If hostname has at least 3 parts, uses first part as subdomain.
- Reserved subdomains return null.
- Localhost uses `NEXT_PUBLIC_DEV_TENANT_SLUG`.

Reserved:

```text
www, app, api, admin, business, staging, localhost
```

Works for:

```text
pho24.scannow.site -> pho24
```

Tenant comments/config now use `scannow.site`.

### 3.4 Axios/auth behavior

File: `src/services/axiosBasic.tsx`.

Behavior:

- `baseURL = NEXT_PUBLIC_API_URL`.
- `withCredentials = true`.
- Adds `Authorization: Bearer {jwt}` if local token exists.
- Adds `X-Tenant-Slug` when tenant slug resolves.
- On 401, calls `/api/auth/refresh-token`, updates auth store, retries once.

### 3.5 Auth state

Files:

- `src/lib/auth.ts`
- `src/stores/user`
- `src/components/auth/protected-route.tsx`

Current redirect map (`src/lib/auth.ts`):

```text
ADMIN          -> /admin/dashboard
OWNER          -> /owner/users
MANAGER        -> /manager/users
BRANCH_MANAGER -> /me/branches
STAFF          -> /me/branches
KITCHEN        -> /me/branches
CASHIER        -> /cashier/dashboard
```

Notes:

- `BRANCH_MANAGER`, `STAFF`, `KITCHEN` now land on the `me/` waiter shell (`/me/branches`), aligned with the backend `/api/me/...` operational model. `/staff/dashboard` and `/kitchen/dashboard` routes still exist but are not the login target.
- `MANAGER` is a legacy FE-only key kept for compatibility; backend role is `BRANCH_MANAGER`.
- Owner/manager user role picker now exposes `BRANCH_MANAGER/STAFF/KITCHEN/CASHIER`.
- Cashier pages are now a full order list/detail/checkout/cancel UI (no longer placeholders).

### 3.6 Owner portal implemented

Owner pages currently integrate:

- `GET /api/owner/restaurant/me`
- `PUT /api/owner/restaurant/me`
- `GET /api/owner/branches`
- `GET /api/owner/branches/{id}`
- `POST /api/owner/branches`
- `PUT /api/owner/branches/{id}`
- `PATCH /api/owner/branches/{id}/active`
- `PATCH /api/owner/branches/{id}/inactive`
- `GET /api/owner/users`
- `POST /api/owner/users`
- `PUT /api/owner/users/{id}`
- `PATCH /api/owner/users/{id}/ban`
- `PATCH /api/owner/users/{id}/unban`

Owner UI supports:

- Restaurant edit.
- Branch CRUD/status.
- User CRUD/status.
- Branch filters/sorting/paging.
- User filters/sorting/paging.

Now also implemented (post SCRUM-30 merge):

- Owner user role picker exposes `BRANCH_MANAGER/STAFF/KITCHEN/CASHIER`.
- Category + menu-item management (CRUD, reorder, active/inactive, toggle availability, bulk availability, toggle featured, image upload, price update + price history).
- Table/QR management (CRUD, status, activate/deactivate, regenerate-qr, qr-image) and branch orders view.
- Branch settings: VAT/service charge, PayOS `payment-config`, paper voucher management.
- Owner dashboard wired to `GET /api/owner/reports/overview` (with Excel export).

Remaining gaps:

- Reports UI is overview-level; advanced analytics/export breakdowns are still future work.

### 3.7 Manager portal implemented

Manager pages currently integrate:

- `GET /api/manager/users`
- `POST /api/manager/users`
- `PUT /api/manager/users/{id}`
- `PATCH /api/manager/users/{id}/ban`
- `PATCH /api/manager/users/{id}/unban`
- `GET /api/me/branches`

Manager UI supports:

- User list.
- User form.
- Branch multi-select/filter UI.
- Role/status filters.

Now also implemented (post SCRUM-30 merge):

- Manager role picker exposes `STAFF/KITCHEN/CASHIER`.
- Manager menu/category management, table/QR management, branch orders, branch settings (payment-config, paper voucher), and reports overview dashboard — same feature set as owner, scoped to the managed branch (`/manager/...`).

Remaining gaps:

- Backend requires exactly one branch for managed user; ensure the UI only submits one branch even if the multi-select control suggests otherwise.

### 3.8 Staff/Kitchen operations (me/ waiter shell)

Staff, Kitchen, and Branch Manager now operate through the `me/` waiter shell (`/me/branches/...`), the login target for these roles. Legacy `/staff/dashboard` and `/kitchen/dashboard` routes still exist but are not used as the entry point.

Implemented and wired to backend:

- List my branches, branch detail (`GET /api/me/branches`, `/api/me/branches/{id}`).
- Branch menu with toggle availability + bulk availability (`/api/me/branches/{id}/menu`, `/api/me/menu-items/{id}/toggle-available`, `/api/me/branches/{id}/menu-items/bulk-availability`).
- Table list + open/close session (`/api/me/branches/{id}/tables`, `/api/me/branches/{branchId}/tables/{tableId}/open`, `/api/me/sessions/{sessionId}/close`, `/api/me/tables/{id}`, `/api/me/tables/{id}/orders`).
- Waiter order lifecycle: pending confirmation, confirm, ready-to-serve, mark served, manual order (`/api/waiter/...`).
- Kitchen queue: grouped items, confirm, mark ready (`/api/kitchen/...`).
- SignalR branch order subscription via `useBranchOrderUpdates` (`/hubs/orders`, `JoinBranch`).

Remaining:

- Runtime smoke test under a live branch with real order traffic.

### 3.9 Cashier UI partial

Backend supports:

- `CASHIER` role.
- `/api/cashier/...`.
- Cash/PayOS checkout.
- Voucher application.

Tenant FE now implements (post SCRUM-30 merge):

- `/cashier/dashboard` with order list + detail (`GET /api/cashier/branches/{branchId}/orders`, `/orders/{orderId}`).
- Checkout form for cash/PayOS with voucher code input (`POST /api/cashier/branches/{branchId}/orders/{orderId}/checkout`).
- Cancel pending payment (`POST /api/cashier/branches/{branchId}/orders/{orderId}/payment-cancel`).
- Cashier role redirect to `/cashier/dashboard`.

Remaining:

- Runtime smoke test of cash amount-received/change and PayOS QR/link on a deployed tenant.

Minimum cashier screens (now covered):

1. Branch selector or auto branch from user assignment.
2. Active orders list with filters.
3. Order detail with items, totals, payment status.
4. Voucher code input.
5. Payment method selector: Cash/PayOS.
6. Cash amount received/change preview.
7. PayOS QR/link display.
8. Cancel pending payment.

### 3.10 Public customer QR app

Implemented routes:

```text
/tables/[qrCodeToken]                          auto-join active session
/sessions/[sessionCode]/menu                   menu + shared cart
/sessions/[sessionCode]/menu-items/[menuItemId]   menu-item detail
/sessions/[sessionCode]/checkout               place order + checkout
/sessions/[sessionCode]/orders/[orderId]       realtime order tracking
/payment/return
/payment/cancel
```

Implemented components/features:

- Table/session join state.
- Menu category list.
- Menu item cards.
- Local cart state via `src/lib/customer-cart.ts`.
- Customer info form.
- Place order confirmation.
- PayOS checkout page.
- Cash-at-counter state.
- Error/expired session state.

Implemented API calls:

- `GET /api/public/tables/{qrCodeToken}`
- `POST /api/public/tables/{qrCodeToken}/join`
- `GET /api/public/sessions/{sessionCode}/menu`
- `POST /api/public/sessions/{sessionCode}/orders`
- `POST /api/public/sessions/{sessionCode}/checkout`
- `GET /api/public/sessions/{sessionCode}/payment-status`
- `POST /api/public/sessions/{sessionCode}/payment-cancel`
- `GET /api/public/sessions/{sessionCode}/orders/{orderId}`

Now also implemented (post SCRUM-30 merge):

- FE SignalR clients: shared cart (`useSharedCart` -> `/hubs/cart`) and order tracking (`useOrderUpdates` -> `/hubs/orders`).
- Standalone order tracking page (`/sessions/[sessionCode]/orders/[orderId]`) with realtime status.
- Menu-item detail page (`GET /api/public/branches/{branchId}/menu-items/{menuItemId}`).

Current gaps:

- Needs deployed-domain smoke test with active table session, real menu data, tenant header, PayOS return/cancel.
- Public cash checkout is labelled "pay at cashier" but backend still marks the order `Completed` with a `PENDING` payment until cashier confirmation.

### 3.11 Tenant FE domain requirements

Deployment must support wildcard hostnames.

The same build should serve:

```text
pho24.scannow.site
highlands.scannow.site
some-restaurant.scannow.site
```

Runtime behavior:

- Extract slug per request/browser host.
- Send slug to backend header.
- Never hardcode tenant slug in build.

Potential metadata issue:

- Static `SITE_CONFIG.baseUrl` does not work for every tenant.
- For tenant customer pages, metadata should derive from request host or be generic/noindex depending product decision.

## 4. App Domain Decision

Quyet dinh chinh thuc: Khong trien khai `business.scannow.site` trong MVP.
- Tat ca users (owner, manager, staff, kitchen, cashier, customer) deu truy cap tren tenant subdomain (`*.scannow.site`) thong qua repository `scan-now-customer`.
- Platform admin doc lap tai `scannow.site/admin` (qua `scan-now-nextjs`).
- `business.scannow.site` chi la future/parked domain cho business portal; khong duoc load tenant operations UI.

## 5. Required FE Work Before Production

P0:

- Set production `NEXT_PUBLIC_API_URL`.
- Align enum types with backend string enums.
- Smoke test tenant public QR ordering, operations (waiter/kitchen/cashier), realtime hubs, and PayOS return/cancel on deployed domains.
- Keep tenant comments/config aligned to `scannow.site`.

Done (was P0/P1):

- SignalR cart/order clients — implemented (`useSharedCart`, `useOrderUpdates`, `useBranchOrderUpdates`).
- Cashier order list/detail/checkout/cancel UI — implemented.
- Owner/manager menu/table/payment-config/voucher UI — implemented.
- Staff/kitchen operations — implemented via the `me/` waiter shell.
- Owner/manager reports overview dashboards — implemented.

P1:

- Admin role guard hardening (in `scan-now-nextjs`).
- Better route-level loading/error states.

P2:

- PWA/offline support.
- Tenant-specific branding/theme/logo.
- Tenant SEO strategy.
- Customer feedback/rating UI.

## 6. FE Acceptance Criteria

- `scannow.site` renders landing and correct metadata.
- `scannow.site/admin` renders admin login and does not redirect to locale route.
- Admin can open restaurant detail by slug.
- `pho24.scannow.site/login` sends `X-Tenant-Slug: pho24`.
- Owner can edit restaurant and branch data under tenant.
- Branch manager can manage users under branch.
- Cashier user logs in and lands on cashier dashboard.
- Customer QR URL loads table page under tenant domain.
- Customer can place order; realtime status updates are delivered via the FE SignalR clients (`/hubs/cart`, `/hubs/orders`).
- Branch manager / staff / kitchen land on the `me/` shell and can run table/menu/order operations.
- Cashier can list orders and checkout cash/PayOS with voucher.
- Payment return/cancel pages handle PayOS redirects.
- No route uses legacy production metadata or links.
