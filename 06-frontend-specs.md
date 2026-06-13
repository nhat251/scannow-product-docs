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
/owner/dashboard          placeholder/protected
/owner/restaurant         owner restaurant page
/owner/branches           owner branches list
/owner/branches/create    create branch
/owner/branches/[id]      branch detail/edit
/owner/users              owner user management
/manager/dashboard        placeholder/protected
/manager/users            manager user management
/staff/dashboard          placeholder/protected
/kitchen/dashboard        placeholder/protected
/cashier/dashboard        cashier placeholder/protected
/cashier/orders           cashier orders placeholder/protected
/tables/[qrCodeToken]     public QR entry and auto-join
/sessions/[sessionCode]/menu
/sessions/[sessionCode]/checkout
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

Current redirect map:

```text
ADMIN          -> /admin/dashboard
OWNER          -> /owner/users
MANAGER        -> /manager/users
BRANCH_MANAGER -> /manager/users
STAFF          -> /staff/dashboard
KITCHEN        -> /kitchen/dashboard
CASHIER        -> /cashier/dashboard
```

Gap:

- Type includes `MANAGER`, but backend role is `BRANCH_MANAGER`.
- Auth and user-management types include `CASHIER`, but owner role picker still needs cashier option alignment.
- Cashier pages currently exist as placeholders, not full order list/detail/checkout UI.

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

Gaps:

- Owner user role options do not expose `CASHIER` yet, although shared user-management types include it.
- Owner does not manage menu/tables/payment config/vouchers from current FE.
- Owner dashboard/reports not wired.

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

Gaps:

- Backend requires exactly one branch for managed user.
- UI may suggest multi-select behavior; ensure it only submits one branch.
- Manager role options are currently kitchen/staff only.
- Manager dashboard/reports/menu/table operations not wired.

### 3.8 Staff/Kitchen dashboards

Routes exist:

```text
/staff/dashboard
/kitchen/dashboard
```

Backend has APIs for:

- Table open/close.
- Pending confirmation.
- Ready to serve.
- Kitchen grouped items.
- Mark ready/served.

Current FE gap:

- Dashboards are not fully integrated with these APIs.
- No SignalR branch order subscription.

### 3.9 Cashier UI partial

Backend supports:

- `CASHIER` role.
- `/api/cashier/...`.
- Cash/PayOS checkout.
- Voucher application.

Tenant FE currently has:

- `/cashier/dashboard`.
- `/cashier/orders`.
- Cashier role redirect.

Tenant FE still misses:

- Cashier order list/detail.
- Cashier checkout form.
- Payment cancel action.

Minimum cashier screens:

1. Branch selector or auto branch from user assignment.
2. Active orders list with filters.
3. Order detail with items, totals, payment status.
4. Voucher code input.
5. Payment method selector: Cash/PayOS.
6. Cash amount received/change preview.
7. PayOS QR/link display.
8. Cancel pending payment.

### 3.10 Public customer QR app implemented basic

Implemented routes:

```text
/tables/[qrCodeToken]
/sessions/[sessionCode]/menu
/sessions/[sessionCode]/checkout
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

Current gaps:

- No FE SignalR client integration for `/hubs/cart` or `/hubs/orders`.
- No rich standalone order tracking timeline/page.
- Needs deployed-domain smoke test with active table session, real menu data, tenant header, PayOS return/cancel.
- Public cash checkout should be treated as "pay at cashier" until cashier confirms payment.

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
- Add SignalR clients for cart/order.
- Complete cashier order list/detail/checkout/cancel UI.
- Smoke test tenant public QR ordering and PayOS return/cancel on deployed domains.
- Keep tenant comments/config aligned to `scannow.site`.

P1:

- Owner/manager menu/table/payment/voucher UI.
- Staff/kitchen operational dashboards.
- Reports dashboards.
- Admin role guard hardening.
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
- Customer can place order; realtime status updates require FE SignalR integration.
- Payment return/cancel pages handle PayOS redirects.
- No route uses legacy production metadata or links.
