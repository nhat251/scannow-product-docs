# Domain, Tenant & Deployment Specification

## 1. Desired Domain Model

Mong muon cua san pham:

```text
https://scannow.site
https://scannow.site/admin
https://<tenant>.scannow.site
https://api.scannow.site
https://business.scannow.site   # future, not MVP
```

Trong do:

- `scannow.site` la landing page.
- `scannow.site/admin` la platform admin.
- `<tenant>.scannow.site` la tenant site cua mot restaurant.
- `tenant` la `Restaurant.Slug`, vi du `pho24.scannow.site`.
- Branch khong nam o subdomain. Branch duoc xac dinh qua table/session/branch route.
- `api.scannow.site` la backend API production domain.
- `business.scannow.site` la future portal cho business account lon quan ly nhieu restaurant.

Recommended API domain:

```text
https://api.scannow.site
```

Backend co the tiep tuc chay o Render URL trong dev/staging, nhung production nen co API domain rieng de quan ly CORS/cookie/security ro rang.

## 2. Current Support Matrix

| Requirement | Current status | Evidence/notes |
|---|---|---|
| Landing at `scannow.site` | Mostly supported | `scan-now-nextjs` root redirects to locale page and renders `SalesLanding`. |
| Admin at `scannow.site/admin` | Supported | `scan-now-nextjs` has `/admin` route, proxy excludes `/admin` from i18n middleware. |
| Restaurant tenant as subdomain | Supported | Backend resolves tenant from header/host. Customer FE extracts subdomain and sends `X-Tenant-Slug`. |
| Tenant = restaurant | Supported | Backend comment/code states tenant granularity = Restaurant. |
| Restaurant has many branches | Supported | Data model has `Restaurant` -> many `Branch`; branch slug unique within restaurant. |
| Backend tenant isolation | Supported | Strict EF Global Query Filters on all tenant entities + Auth Login Validation. |
| Admin routes by slug | Supported | Backend has `/api/admin/restaurants/by-slug/{slug}` and branch-by-slug endpoints. |
| Wildcard CORS for tenants | Supported | `App:ProductionDomain=scannow.site`; production domain has been configured. |
| Tenant public ordering UI | Supported / needs smoke test | `/tables/[qrCodeToken]`, menu, checkout, payment return/cancel routes exist in tenant FE. |
| QR URL for tenant domain | Supported when env is set | Built dynamically by `ITenantUrlBuilder` using `App:TenantBaseDomain`; if missing, it falls back to platform base URL. |
| Payment return/cancel tenant pages | Supported | PayOS redirects to tenant domain instead of static base URL. |
| `business.scannow.site` | Future / not MVP | Reserved for multi-restaurant business portal, not restaurant daily operations. |

## 3. Backend Tenant Resolution

Backend audited branch `test/deploy` includes `TenantResolutionMiddleware`.

Resolution priority:

1. Read `X-Tenant-Slug`.
2. If no header, extract first subdomain from request host.

Reserved subdomains:

```text
www, api, admin, app, business, localhost, staging
```

Behavior:

- If slug exists and active restaurant found, middleware sets `TenantContext.RestaurantId` and `TenantContext.Slug`.
- If slug missing/invalid/reserved, tenant context is unresolved.
- When tenant context is unresolved, EF filter does not apply. This is intended for platform admin and non-tenant tools.

Important deployment implication:

- If frontend tenant app calls backend at `api.scannow.site`, backend host is `api.scannow.site` and cannot infer tenant from host because `api` is reserved.
- Therefore tenant FE must send `X-Tenant-Slug`.
- `scan-now-customer` already does this in Axios interceptor.

## 4. Tenant Isolation Model

Current isolation point:

```text
TenantContext.RestaurantId
    -> EF Core global query filter on all tenant entities (Branch, Category, MenuItem, Order, etc.)
    -> AuthService prevents login if user is not Owner/BranchStaff of the current Tenant
```

Why this matches product model:

- Tenant is restaurant.
- Branch is the root operational entity under restaurant.
- Menu, tables, orders, payments, vouchers are branch-scoped.

Expected access boundary:

- Tenant request for `pho24` should only resolve branches where `Branch.RestaurantId = pho24.RestaurantId`.
- If a route passes a branchId from another restaurant, branch lookup should return not found or forbidden.

Current implementation detail:

- `ApplicationDbContext` defines global filters for `Restaurant`, `Branch`, `BranchStaff`, `Category`, `MenuItem`, `RestaurantTable`, `Order`, `QrSession`, `ItemRating`, `DiscountCode`, `BranchPaymentConfig`, `PaperVoucher`, `Notification`, `AuditLog`, `OrderItem`, `Payment`, and `MenuItemPriceHistory`.
- Some filters traverse navigation paths such as `Order.Branch.RestaurantId` or `Payment.Order.Branch.RestaurantId`.
- This is stronger than branch-only filtering, but it still needs regression tests because tenant safety is business-critical.

Recommended hardening:

- Add integration tests for every public and tenant route using tenant A header + tenant B IDs.
- Keep repository/service-level branch assertions even with global filters.
- Avoid relying only on `Include(Branch)` side effects.

## 5. Frontend Domain Routing

### 5.1 Landing/Admin FE

Repo: `scan-now-nextjs`

Expected deployment:

```text
Host: scannow.site
Routes:
  /              -> redirects to /vi
  /vi, /en       -> landing
  /admin         -> admin login/dashboard
  /admin/...     -> admin management pages
```

Current caveats:

- `SITE_CONFIG.baseUrl` is configured as `https://scannow.site`.
- `NEXT_PUBLIC_API_URL` is configured as `https://api.scannow.site`.
- Admin route uses local storage JWT and Zustand; there is no server-side auth guard.

### 5.2 Tenant/Portal FE

Repo: `scan-now-customer`

Expected deployment:

```text
Host: *.scannow.site
Routes Map:
  /login
  /owner/dashboard
  /owner/restaurant
  /owner/branches
  /owner/branches/create
  /owner/branches/[id]
  /owner/users
  /manager/dashboard
  /manager/users
  /staff/dashboard
  /kitchen/dashboard
  /cashier/dashboard
  /cashier/orders
  /tables/[qrCodeToken]
  /sessions/[sessionCode]/menu
  /sessions/[sessionCode]/checkout
  /payment/return
  /payment/cancel
```

Current tenant slug extraction:

- Browser hostname `pho24.scannow.site` -> `pho24`.
- Reserved subdomains return null.
- Local dev can use `NEXT_PUBLIC_DEV_TENANT_SLUG`.
- Axios sends `X-Tenant-Slug` to backend when slug exists.

Current notes:

- Tenant comments/config use `scannow.site`.
- Public QR ordering pages exist; full production readiness still requires smoke test with real domain/data.
- Cashier routes exist as placeholders and auth routing supports `CASHIER`; full cashier operations UI is still future work.
- `UserRole` FE types include backend role `CASHIER`.
- Tenant FE `SITE_CONFIG.baseUrl` is configured as `https://scannow.site`; tenant-specific metadata can be made host-aware later.

## 6. What Is `business.scannow.site`?

Short answer: **future business portal, not MVP**.

All restaurant roles (owner, manager, staff, kitchen, cashier) and customers will operate entirely under `<tenant>.scannow.site` using the `scan-now-customer` app.

`business.scannow.site` is reserved for future use cases such as:
- A business account that owns/manages multiple restaurant tenants.
- A tenant switcher for business users who belong to multiple restaurants.
- Business-level reports, billing/subscription, and organization-level user management.
- Optional support/admin workflows for business customers.

It must not contain customer QR ordering or replace `<tenant>.scannow.site` for daily restaurant operations.

## 7. DNS and Hosting Requirements

Minimum DNS:

```text
A/CNAME scannow.site          -> landing/admin deployment
CNAME www.scannow.site        -> scannow.site
CNAME *.scannow.site          -> tenant FE deployment
CNAME api.scannow.site        -> backend/API deployment
```

Hosting requirements:

- Landing/admin deployment must serve `/admin` and locale routes.
- Tenant deployment must accept wildcard hostnames.
- `business.scannow.site` must not be routed to the tenant app until the business portal exists. If wildcard DNS already resolves it, either point it to a parked/future portal response or add `business` to frontend/backend reserved subdomain lists before routing it through the tenant stack.
- Backend must accept CORS from:
  - `https://scannow.site`
  - `https://www.scannow.site`
  - `https://*.scannow.site`
- TLS certificates must cover apex and wildcard:
  - `scannow.site`
  - `*.scannow.site`
  - `api.scannow.site`

## 8. Required Environment Configuration

### 8.1 Backend production env

Required:

```text
App__ClientUrl=https://scannow.site
App__FrontendBaseUrl=https://scannow.site
App__AllowedOrigins=https://scannow.site;https://www.scannow.site
App__ProductionDomain=scannow.site
App__TenantBaseDomain=scannow.site
App__QrTablePath=/tables
Jwt__Issuer=...
Jwt__Audience=...
Jwt__SecretKey=...
ConnectionStrings__ScanNowDB=...
PayOS__ReturnUrl=...
PayOS__CancelUrl=...
```

Tenant QR URLs are not built from `App__FrontendBaseUrl` alone. Backend uses `ITenantUrlBuilder` and `App__TenantBaseDomain` to generate:

```text
https://{restaurantSlug}.scannow.site/tables/{qrCodeToken}
```

Important: source `ScanNow.Web/appsettings.json` currently contains `App:ProductionDomain=scannow.site` but not `App:TenantBaseDomain`. Production/staging env must explicitly set `App__TenantBaseDomain=scannow.site`; otherwise QR/payment tenant links fall back to `App__ClientUrl`/`App__FrontendBaseUrl`.

### 8.2 Landing/admin FE env

```text
NEXT_PUBLIC_API_URL=https://api.scannow.site
RESEND_API_KEY=...
TO_EMAIL_ADDRESS=...
SENDER_EMAIL_ADDRESS=...
```

Also update source/config:

```text
SITE_CONFIG.baseUrl=https://scannow.site
```

### 8.3 Tenant FE env

```text
NEXT_PUBLIC_API_URL=https://api.scannow.site
NEXT_PUBLIC_DEV_TENANT_SLUG=        # only local dev
```

Also update source/config:

```text
SITE_CONFIG.baseUrl=https://scannow.site or tenant-aware metadata strategy
```

For tenant FE, static `SITE_CONFIG.baseUrl` is less useful because host changes by tenant. Metadata should derive from request host if SEO for tenant pages matters.

## 9. Implemented Tenant Customer Flow

### 9.1 Tenant FE public route

Implemented:

```text
scan-now-customer/src/app/tables/[qrCodeToken]/page.tsx
```

Flow:

1. Read `qrCodeToken`.
2. `GET /api/public/tables/{qrCodeToken}`.
3. `POST /api/public/tables/{qrCodeToken}/join`.
4. Backend finds active session for the table.
5. Redirect to `/sessions/{sessionCode}/menu`.

QR remains static and only identifies the table. Session remains dynamic and must be opened by staff/cashier before customer can join. Public QR never creates a session.

### 9.2 Tenant FE menu/order route

Implemented pages/components:

```text
/sessions/{sessionCode}/menu
/sessions/{sessionCode}/checkout
/payment/return
/payment/cancel
```

Use APIs:

- `GET /api/public/sessions/{sessionCode}/menu`
- `POST /api/public/sessions/{sessionCode}/orders`
- `POST /api/public/sessions/{sessionCode}/checkout`
- `GET /api/public/sessions/{sessionCode}/payment-status`
- `POST /api/public/sessions/{sessionCode}/payment-cancel`
- `GET /api/public/sessions/{sessionCode}/orders/{orderId}`

Backend hubs available:

- `/hubs/cart`
- `/hubs/orders`

Current FE status: no SignalR client integration was found in `scan-now-customer`; customer menu uses local cart storage and payment/status pages use HTTP APIs. Realtime cart/order tracking remains future hardening.

### 9.3 Dynamic QR URL generation

Implemented in backend:

- `ITenantUrlBuilder.BuildTenantTableUrl(slug, qrCodeToken)`.
- `TableQrService` passes branch restaurant slug when creating/regenerating QR.
- Generated URL:

```text
https://{restaurantSlug}.scannow.site/tables/{qrCodeToken}
```

### 9.4 Dynamic payment redirect

Implemented in backend:

- `CheckoutService` builds customer PayOS redirect with restaurant slug.
- `CashierService` builds cashier PayOS redirect with restaurant slug.
- Redirect shape:

```text
https://{restaurantSlug}.scannow.site/payment/{return|cancel}?sessionCode=...&orderId=...
```

Fallback remains platform base URL when slug/domain config is missing.

### 9.5 Role and route sync

Backend role list includes:

```text
ADMIN, OWNER, BRANCH_MANAGER, STAFF, KITCHEN, CASHIER
```

Tenant FE includes:

- `CASHIER` role type.
- `CASHIER -> /cashier/dashboard` redirect.
- `/cashier/dashboard` and `/cashier/orders` placeholders.
- User-management types include `CASHIER`, but owner role picker still exposes only `BRANCH_MANAGER`, `STAFF`, `KITCHEN`; manager user form exposes `STAFF`, `KITCHEN`.

## 10. Request Flow Examples

### 10.1 Admin creates tenant

```text
Admin browser -> https://scannow.site/admin
Admin FE -> POST https://api.scannow.site/api/admin/users/owners
Admin FE -> POST https://api.scannow.site/api/admin/restaurants { slug: "pho24" }
Admin FE -> optional create branches via owner/admin workflows
DNS wildcard already maps pho24.scannow.site to tenant FE
```

### 10.2 Owner manages restaurant

```text
Owner browser -> https://pho24.scannow.site/login
Tenant FE extracts "pho24"
Tenant FE -> POST /api/auth/login with X-Tenant-Slug: pho24
Tenant FE -> GET /api/owner/restaurant/me with X-Tenant-Slug: pho24
Backend resolves tenant and scopes branch queries to RestaurantId
```

### 10.3 Customer order

```text
Customer scans QR
QR URL -> https://pho24.scannow.site/tables/{qrCodeToken}
Tenant FE extracts "pho24"
Tenant FE -> GET /api/public/tables/{qrCodeToken} with X-Tenant-Slug: pho24
Tenant FE -> POST /api/public/tables/{qrCodeToken}/join
Tenant FE -> GET /api/public/sessions/{sessionCode}/menu
Tenant FE -> POST /api/public/sessions/{sessionCode}/orders
Tenant FE -> POST /api/public/sessions/{sessionCode}/checkout
Future hardening -> SignalR /hubs/cart and /hubs/orders for shared cart/order tracking
```

### 10.4 Cashier checkout

```text
Cashier browser -> https://pho24.scannow.site/login
Cashier role redirects to /cashier...
Cashier FE -> GET /api/cashier/branches/{branchId}/orders
Cashier FE -> POST /api/cashier/branches/{branchId}/orders/{orderId}/checkout
```

## 11. Acceptance Checklist For Domain Architecture

- `scannow.site` loads landing.
- `/vi` and `/en` work.
- `/admin` does not go through locale middleware.
- `/admin` can login admin.
- Admin restaurant detail by slug works.
- `pho24.scannow.site` loads tenant FE.
- Tenant FE sends `X-Tenant-Slug: pho24`.
- Backend CORS accepts `https://pho24.scannow.site`.
- Tenant request cannot access another restaurant branch.
- QR generated for branch under `pho24` points to `https://pho24.scannow.site/tables/...`.
- Payment return/cancel returns to `https://pho24.scannow.site/payment/...`.
- `api.scannow.site` is the backend API base domain.
- `business.scannow.site` is reserved for future business portal and not used in MVP.
