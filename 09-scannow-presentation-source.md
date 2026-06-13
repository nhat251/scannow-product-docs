# ScanNow Presentation Source

Cap nhat: 2026-06-13
Muc dich: Tai lieu nguon chi tiet de dua vao NotebookLM hoac dung lam outline thuyet trinh ve ScanNow.

## 1. Executive Summary

ScanNow la mot SaaS cho nha hang/F&B, tap trung vao QR ordering tai ban, quan tri nhieu chi nhanh, van hanh order theo role va thanh toan cash/PayOS.

Kien truc san pham chinh:

```text
scannow.site                  Landing page va platform admin
scannow.site/admin            Platform admin
api.scannow.site              Backend API
<restaurantSlug>.scannow.site Tenant app cho nha hang
business.scannow.site         Future portal, khong nam trong MVP
```

Ket luan quan trong:

- Tenant = `Restaurant`.
- Mot restaurant co nhieu `Branch`.
- Branch la don vi van hanh: menu, category, table, QR session, order, payment config, voucher.
- Customer QR, owner, manager, staff, kitchen, cashier deu nam trong tenant domain `<tenant>.scannow.site`.
- Platform admin nam rieng o `scannow.site/admin`.
- Backend da co tenant resolution, global query filters, role auth, public QR APIs, cashier APIs, kitchen/waiter APIs, reports APIs, PayOS, QR generation va SignalR hubs.
- Tenant FE (sau merge `feature/SCRUM-30-customer-qr-session`) da hien thuc gan nhu toan bo product: owner/manager back office (menu, table, settings, payment-config, voucher, reports), `me/` waiter shell cho staff/kitchen/branch-manager, full cashier, customer QR/menu/checkout/order-tracking/payment, va SignalR cart/order realtime.
- Remaining production gaps lon nhat: production smoke test tren domain that, cross-tenant integration tests, SignalR backplane cho multi-instance, va dependency warning remediation.

## 2. Audit Sources

Audit nay dua tren 4 repo:

```text
Backend:
  Path: /home/nhat/fpt/ScanNow
  Audited branch: test/deploy
  Commit: c41c3c4
  Note: requested branch test/delpy was not present locally/remotely in listed branches.

Landing/Admin FE:
  Path: /home/nhat/fpt/scan-now-nextjs
  Audited branch: main
  Commit: 7d59f7c
  Note: repo checkout was feat/multitenant, so main was audited through a temporary worktree.

Tenant/Customer FE:
  Path: /home/nhat/fpt/scan-now-customer
  Audited branch: main
  Commit: 1c44a03

Docs:
  Path: /home/nhat/fpt/scannow-product-docs
  Branch: main
  Base commit before this update: 9a0d189
```

Verification commands run:

```text
Backend:
  dotnet build ScanNow.slnx --no-restore
  Result: pass, 0 errors, 14 warnings

Landing/Admin FE main:
  pnpm install --frozen-lockfile
  pnpm lint
  Result: pass

Tenant/Customer FE:
  pnpm lint
  Result: pass
```

Backend warnings to mention honestly:

- AutoMapper 12.0.1 high severity advisory.
- MailKit/MimeKit moderate severity advisories.
- Nullable warnings in domain/repository code.
- Unread logger parameter warning.

## 3. Product Problem

Nha hang vua va nho thuong gap cac van de:

- Khach phai doi nhan vien den ban moi duoc goi mon.
- Menu giay kho cap nhat khi mon het, gia doi, combo thay doi.
- Nhan vien, bep va thu ngan khong nhin cung mot trang thai order.
- Nha hang co nhieu chi nhanh can quan ly rieng menu, ban, thue, phi dich vu, thanh toan.
- Chu nha hang can tao tai khoan cho staff/kitchen/cashier theo chi nhanh.
- Thanh toan can linh hoat: cash tai quay hoac PayOS QR.
- Platform can onboard nha hang moi nhanh, moi nha hang co tenant domain rieng.

ScanNow giai quyet bang:

- QR table ordering.
- Tenant subdomain cho tung nha hang.
- Role-based back office.
- Multi-branch data model.
- Order lifecycle co waiter/kitchen/cashier.
- Payment config/voucher theo branch.
- Backend tenant isolation de tranh ro ri du lieu giua nha hang.

## 4. Product Scope

### 4.1 In Scope For Current MVP

Platform:

- Landing page `scannow.site`.
- Lead capture form.
- Admin login.
- Admin manage owner accounts.
- Admin manage restaurant/tenant.
- Admin view restaurant by slug/GUID.
- Admin supervise branch/menu/table/session read-only.

Tenant/back office:

- Tenant login tai `<tenant>.scannow.site/login`.
- Owner restaurant profile.
- Owner branch CRUD/status.
- Owner user CRUD/status.
- Manager user management.
- Owner/manager back office day du: menu/category, table/QR, branch settings (payment-config, voucher), reports overview dashboard.
- `me/` waiter shell cho staff/kitchen/branch-manager: open/close session, branch menu availability, waiter confirm/serve, kitchen queue.
- Cashier portal day du: order list/detail, cash/PayOS checkout, voucher, cancel pending payment.

Customer QR:

- QR URL ve `<tenant>.scannow.site/tables/{qrCodeToken}`.
- Auto join active table session.
- Session menu.
- Cart local storage.
- Place order.
- Checkout page.
- PayOS return/cancel pages.
- Payment status HTTP check.

Backend operations:

- Auth/register/login/google/refresh/logout/email verification/change password.
- Role auth: ADMIN, OWNER, BRANCH_MANAGER, STAFF, KITCHEN, CASHIER.
- Tenant resolution by `X-Tenant-Slug` or host subdomain.
- Strict application-level tenant isolation using EF Core global query filters.
- Owner/manager menu APIs.
- Table/QR/session APIs.
- Public QR APIs.
- Waiter APIs.
- Kitchen APIs.
- Cashier APIs.
- Branch payment config.
- Paper voucher.
- Reports.
- SignalR hubs for cart/order.

### 4.2 Out Of Scope For MVP

- `business.scannow.site` business portal.
- Multi-restaurant business account.
- Tenant switcher.
- Full POS integration.
- Loyalty/membership.
- Inventory management.
- Reservation/booking.
- Customer account/login.
- Tenant-specific SEO strategy.
- Full PWA/offline mode.
- Multi-instance SignalR scaling.

### 4.3 Future Scope

- Business portal for large customers.
- Multi-restaurant owner/business organization.
- Tenant branding: logo, colors, custom menu theme.
- Advanced analytics/export.
- PayOS webhook and reconciliation.
- Refund flow.
- Customer ratings/reviews.
- Push/SMS notifications.
- Redis/distributed cache and SignalR backplane.
- Staff performance and kitchen prep-time analytics.

## 5. Users And Personas

### 5.1 Platform Admin

Runs ScanNow platform.

Main needs:

- Create owner accounts.
- Create restaurants/tenants.
- Assign owner to restaurant.
- Ban/unban owner or restaurant.
- Inspect branches, menu items, tables, sessions.
- View platform dashboard/report.

Current FE:

- Supported in `scan-now-nextjs`.
- Admin route guard still client-side and does not strongly enforce `ADMIN` role in FE.

### 5.2 Restaurant Owner

Owns one restaurant tenant.

Main needs:

- Login through tenant domain.
- Edit restaurant profile.
- Manage branches.
- Manage branch users.
- Eventually manage menu, tables, payment config, vouchers, reports.

Current FE:

- Restaurant, branch, user management implemented.
- Menu/table/payment/voucher/report UI not yet implemented.
- Owner role picker currently does not expose `CASHIER`, although backend and shared types support cashier.

### 5.3 Branch Manager

Manages assigned branch.

Main needs:

- Manage staff/kitchen/cashier users in branch.
- See branch operations.
- Manage menu/table/payment depending permission model.
- View branch report.

Current FE:

- Manager users page implemented; role picker exposes STAFF/KITCHEN/CASHIER.
- Manager menu/table management, branch orders, branch settings (payment-config, voucher), and reports overview dashboard implemented (scoped to managed branch).
- Branch operations available via the `me/` waiter shell.

### 5.4 Staff/Waiter

Runs table service.

Main needs:

- Open table session.
- Close session.
- Confirm customer orders.
- Create manual order.
- See ready-to-serve items.
- Mark items served.

Current FE:

- Implemented via the `me/` waiter shell (`/me/branches/...`): open/close table session, branch menu availability, waiter confirm, ready-to-serve, mark served. Wired to `/api/me` + `/api/waiter` and SignalR `/hubs/orders`.

### 5.5 Kitchen

Runs preparation workflow.

Main needs:

- See pending confirmation.
- Confirm order or individual items.
- See grouped kitchen queue by item.
- Mark items ready.

Current FE:

- Implemented via the `me/` kitchen page (`/me/branches/[branchId]/kitchen`): pending confirmation, confirm order/items, grouped queue, mark ready. Wired to `/api/kitchen` and SignalR `/hubs/orders`.

### 5.6 Cashier

Runs payment workflow.

Main needs:

- List active/paid orders.
- Search by order number, table, customer, phone, session code.
- Apply paper voucher.
- Checkout cash with received/change.
- Create PayOS payment link/QR.
- Cancel pending payment.

Current FE:

- Full cashier UI implemented: order list/detail, cash/PayOS checkout with received/change, voucher code, PayOS QR/link, cancel pending payment (`/api/cashier/branches/{branchId}/orders/...`).

### 5.7 Public Customer

Eats at a table.

Main needs:

- Scan QR.
- Join active table session.
- Browse menu.
- Add items to cart.
- Place order.
- Checkout PayOS or pay at cashier.
- See payment status.

Current FE:

- Full QR/menu/menu-item/cart/order/checkout/order-tracking/payment pages implemented.
- Shared cart + order tracking via SignalR (`useSharedCart` -> `/hubs/cart`, `useOrderUpdates` -> `/hubs/orders`).

## 6. Repository Architecture

### 6.1 Backend Repo

Path:

```text
/home/nhat/fpt/ScanNow
```

Solution/project structure:

```text
ScanNow.Domain
  Entities
  Enums
  Exceptions
  Abstractions

ScanNow.Application
  Features
  DTOs
  Validators
  Services
  Mappers
  Abstractions

ScanNow.Infrastructure
  EF Core DbContext
  Entity configurations
  Repositories
  External services
  Tenancy implementation
  Migrations

ScanNow.Web
  Controllers
  Middleware
  SignalR hubs
  Program.cs
  Exception handlers
```

Design style:

- Clean-ish layered architecture.
- Domain contains core entities/enums.
- Application contains use-case services and validators.
- Infrastructure implements persistence/external providers.
- Web exposes controllers, middleware, hubs.

### 6.2 Landing/Admin FE Repo

Path:

```text
/home/nhat/fpt/scan-now-nextjs
```

Responsibilities:

- Landing page.
- i18n landing `vi/en`.
- Lead capture API route.
- Platform admin.
- Owner/restaurant/admin supervision screens.

Current routes:

```text
/
/[locale]
/admin
/admin/restaurants
/admin/restaurants/create
/admin/restaurants/[id]
/admin/restaurants/[id]/edit
/admin/branches/[branchId]
/admin/branches/[branchId]/categories/[categoryId]
/admin/branches/[branchId]/tables/[tableId]
/admin/menu-items/[menuItemId]
/api/lead-capture
```

### 6.3 Tenant/Customer FE Repo

Path:

```text
/home/nhat/fpt/scan-now-customer
```

Responsibilities:

- Tenant login/auth bootstrap.
- Owner portal.
- Manager portal.
- Staff/kitchen/cashier protected routes.
- Customer QR ordering.
- Payment result pages.

Current routes:

```text
/
/login
/admin/dashboard
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

## 7. Technology Stack

### 7.1 Backend Technology

Runtime/framework:

- .NET 10 target framework.
- ASP.NET Core Web API.
- ASP.NET Core Identity.
- JWT Bearer authentication.
- EF Core 10.
- PostgreSQL via Npgsql.
- SignalR.
- Scalar/OpenAPI in development.

Application libraries:

- FluentValidation for request validation.
- AutoMapper for mapping.
- Google.Apis.Auth for Google login.
- QRCoder for QR image generation.
- DotNetEnv for local env loading.

External integrations:

- PayOS for payment link/QR.
- Cloudinary for menu image upload.
- SMTP/MailKit for email verification.

Operational behavior:

- Loads `.env` from solution/web directories.
- Adds environment variables to configuration.
- Auto-migrates database on startup.
- Seeds roles and initial data.
- CORS allows configured origins and wildcard tenant subdomain through `App:ProductionDomain`.

Important backend config keys:

```text
ConnectionStrings__ScanNowDB
Jwt__Issuer
Jwt__Audience
Jwt__SecretKey
Jwt__AccessTokenExpirationMinutes
Jwt__RefreshTokenExpirationDays
App__ClientUrl
App__FrontendBaseUrl
App__AllowedOrigins
App__ProductionDomain
App__TenantBaseDomain
App__QrTablePath
Email__Host
Email__Port
Email__SenderName
Email__SenderEmail
Email__Username
Email__Password
Cloudinary__CloudName
Cloudinary__ApiKey
Cloudinary__ApiSecret
PayOS__ClientId
PayOS__ApiKey
PayOS__ChecksumKey
```

Difference between two domain config keys:

- `App:ProductionDomain`: used by CORS wildcard subdomain logic.
- `App:TenantBaseDomain`: used by `ITenantUrlBuilder` for QR/payment URLs.

This distinction is critical. If `TenantBaseDomain` is missing, QR/payment URLs can fall back to platform base URL.

### 7.2 Frontend Technology

Both FE repos:

- Next.js 16 App Router.
- React 19.
- TypeScript.
- Tailwind CSS v4.
- Radix/shadcn-style UI components.
- Axios.
- TanStack Query.
- Zustand.
- React Hook Form.
- Zod.
- Lucide icons.
- pnpm 10.
- Node >= 22.

Landing/Admin FE adds:

- next-intl for landing locale routing.
- Resend for lead email.
- HubSpot/Slack deps present in package, but core audited flow is landing/admin.

Tenant/Customer FE adds:

- Tenant slug extraction.
- Auth bootstrap and refresh flow.
- Protected route by role.
- Public ordering service.
- Local customer cart helper.

## 8. Domain And Tenant Architecture

Official domain model:

```text
scannow.site
scannow.site/admin
api.scannow.site
<tenant>.scannow.site
business.scannow.site future only
```

Tenant resolution:

1. Frontend extracts restaurant slug from browser hostname.
2. Tenant FE sends `X-Tenant-Slug`.
3. Backend middleware reads `X-Tenant-Slug`.
4. If header absent, backend attempts host subdomain.
5. Backend finds active `Restaurant` by slug.
6. Backend writes `TenantContext.RestaurantId` and `TenantContext.Slug`.
7. EF Core global filters scope tenant data.

Reserved subdomain labels:

```text
www, api, admin, app, business, localhost, staging
```

Why `business.scannow.site` is not MVP:

- Current product model is restaurant tenant, not business organization.
- Owner/manager/staff/kitchen/cashier operate in restaurant tenant domain.
- Customer QR also belongs in restaurant tenant domain.
- Business portal would need extra concepts: business account, multiple restaurants per account, tenant switcher, business-level billing and reports.

## 9. Data Model

### 9.1 Identity

Core entities:

- `ApplicationUser`
- `ApplicationRole`
- `RefreshToken`
- `PasswordResetToken`

Roles:

```text
ADMIN
OWNER
BRANCH_MANAGER
STAFF
KITCHEN
CASHIER
```

User can:

- Own restaurants.
- Manage branches.
- Be assigned to branches through `BranchStaff`.
- Confirm/cancel orders.
- Process/refund payments.

### 9.2 Tenant And Branch

`Restaurant`:

- Tenant identity.
- Has global unique slug.
- Owned by owner user.
- Has many branches.

`Branch`:

- Belongs to restaurant.
- Has optional manager.
- Has address/contact/open/close time.
- Has VAT percent.
- Has service charge percent and fixed field.
- Has menu, tables, orders, payment config, vouchers.

Important product rule:

- Tenant is restaurant.
- Branch is not a subdomain.
- Branch is selected by route/session/table context.

### 9.3 Menu

Entities:

- `Category`
- `MenuItem`
- `MenuItemPriceHistory`

Menu is branch-specific.

Public menu includes:

- Active categories.
- Active items.
- Available items.

Menu item supports:

- Image.
- Price/cost price.
- Preparation time.
- Display order.
- Availability.
- Featured flag.
- Active flag.
- Price history.

### 9.4 Table And QR Session

Entities:

- `RestaurantTable`
- `QrSession`

Table fields:

- Table number.
- Capacity.
- QR code token.
- QR code URL.
- QR image URL.
- Status.
- Active flag.

Table statuses:

```text
AVAILABLE
OCCUPIED
RESERVED
DISABLED
```

QR/session behavior:

- Table QR token is static.
- Staff/cashier/kitchen can open a table session.
- Opening a session marks table occupied.
- Session token is 6 characters.
- Session expires after 6 hours.
- Customer scan uses table token, then backend joins active session.
- Public customer cannot create session.

### 9.5 Orders And Items

Entities:

- `Order`
- `OrderItem`

Order statuses:

```text
PendingConfirmation
Confirmed
Preparing
PartiallyReady
ReadyToServe
PartiallyServed
Served
Completed
Cancelled
```

Order item statuses:

```text
Pending
Confirmed
Cooking
Ready
Served
Cancelled
```

Order sources:

```text
QR
STAFF_MANUAL
```

Order totals:

- Subtotal.
- VAT percent/amount.
- Service charge percent/amount.
- Discount amount.
- Total amount.

Known calculation gap:

- `Branch.ServiceChargeFixed` exists, but code currently applies percent service charge only.

### 9.6 Payment And Vouchers

Entities:

- `Payment`
- `BranchPaymentConfig`
- `PaperVoucher`
- `DiscountCode`

Payment methods:

```text
CASH
VNPAY
MOMO
BANK_TRANSFER
PAYOS
```

Currently supported checkout methods in validator:

```text
CASH
PAYOS
```

Payment statuses:

```text
PENDING
SUCCESS
FAILED
REFUNDED
```

Branch payment config:

- Cash enabled.
- PayOS enabled.
- PayOS client ID/API key/checksum key.
- Default method.

Paper voucher:

- Code.
- Name.
- Percent or fixed amount.
- Min order amount.
- Max discount.
- Quantity.
- Used count.
- Valid date range.

## 10. Backend Modules

### 10.1 Auth

Endpoints:

```text
POST /api/auth/register
POST /api/auth/login
POST /api/auth/login-google
POST /api/auth/refresh-token
POST /api/auth/logout
POST /api/auth/check-email-exist
POST /api/auth/check-username-exist
POST /api/auth/change-password
POST /api/auth/resend-email-verification
POST /api/auth/verify-email
```

Key behavior:

- Email/username login.
- Google login.
- Email confirmation required for local login.
- JWT access token.
- Refresh token repository/cookie flow.
- Tenant login guard: non-admin user must be owner or branch staff of current tenant.

### 10.2 Platform Admin

Admin owners:

```text
GET /api/admin/users/owners
GET /api/admin/users/owners/available
POST /api/admin/users/owners
PUT /api/admin/users/owners/{id}
PATCH /api/admin/users/owners/{id}/ban
PATCH /api/admin/users/owners/{id}/unban
```

Admin restaurants:

```text
GET /api/admin/restaurants
GET /api/admin/restaurants/{id}
GET /api/admin/restaurants/by-slug/{slug}
GET /api/admin/restaurants/{id}/branches
GET /api/admin/restaurants/by-slug/{slug}/branches
GET /api/admin/restaurants/{id}/branches/{branchId}
GET /api/admin/restaurants/by-slug/{slug}/branches/{branchSlug}
POST /api/admin/restaurants
PUT /api/admin/restaurants/{id}
PATCH /api/admin/restaurants/{id}/ban
PATCH /api/admin/restaurants/{id}/unban
```

Admin supervision:

- Branch categories.
- Branch menu items.
- Menu item detail.
- Price history.
- Branch tables.
- Table detail.
- Active sessions.

### 10.3 Owner And Manager

Owner restaurant:

```text
GET /api/owner/restaurant/me
PUT /api/owner/restaurant/me
```

Owner branches:

```text
POST /api/owner/branches
GET /api/owner/branches
GET /api/owner/branches/{id}
PUT /api/owner/branches/{id}
PATCH /api/owner/branches/{id}/active
PATCH /api/owner/branches/{id}/inactive
```

Owner users:

```text
GET /api/owner/users
POST /api/owner/users
PUT /api/owner/users/{id}
PATCH /api/owner/users/{id}/ban
PATCH /api/owner/users/{id}/unban
```

Manager users:

```text
GET /api/manager/users
POST /api/manager/users
PUT /api/manager/users/{id}
PATCH /api/manager/users/{id}/ban
PATCH /api/manager/users/{id}/unban
```

### 10.4 Menu Management

Owner/manager category routes:

```text
GET /api/owner/branches/{branchId}/categories
GET /api/owner/branches/{branchId}/categories/{id}
POST /api/owner/branches/{branchId}/categories
PUT /api/owner/branches/{branchId}/categories/{id}
PATCH /api/owner/branches/{branchId}/categories/reorder
PATCH /api/owner/branches/{branchId}/categories/{id}/active
PATCH /api/owner/branches/{branchId}/categories/{id}/inactive
```

Menu item routes:

```text
GET /api/owner/branches/{branchId}/menu-items
GET /api/owner/menu-items/{id}
POST /api/owner/menu-items/images
POST /api/owner/branches/{branchId}/categories/{categoryId}/menu-items
PUT /api/owner/menu-items/{id}
PATCH /api/owner/menu-items/{id}/active
PATCH /api/owner/menu-items/{id}/inactive
PATCH /api/owner/branches/{branchId}/menu-items/reorder
PATCH /api/owner/menu-items/{id}/toggle-available
PATCH /api/owner/branches/{branchId}/menu-items/bulk-availability
PATCH /api/owner/menu-items/{id}/toggle-featured
PATCH /api/owner/menu-items/{id}/price
GET /api/owner/menu-items/{id}/price-history
```

### 10.5 Table And QR

Owner/manager table routes:

```text
GET /api/owner/branches/{branchId}/tables
GET /api/owner/branches/{branchId}/tables/{id}
GET /api/owner/branches/{branchId}/tables/{id}/orders
GET /api/owner/branches/{branchId}/orders
POST /api/owner/branches/{branchId}/tables
PUT /api/owner/tables/{id}
PATCH /api/owner/tables/{id}/status
PATCH /api/owner/tables/{id}/activate
PATCH /api/owner/tables/{id}/deactivate
POST /api/owner/tables/{id}/regenerate-qr
GET /api/owner/tables/{id}/qr-image
```

Staff/cashier/kitchen "me" routes:

```text
GET /api/me/branches
GET /api/me/branches/{id}
GET /api/me/branches/{branchId}/tables
GET /api/me/tables/{id}
GET /api/me/tables/{id}/orders
POST /api/me/branches/{branchId}/tables/{tableId}/open
PATCH /api/me/sessions/{sessionId}/close
```

### 10.6 Public Customer QR

Public routes:

```text
GET /api/public/tables/{qrCodeToken}
POST /api/public/tables/{qrCodeToken}/join
POST /api/public/sessions/join
GET /api/public/sessions/{sessionCode}/menu
POST /api/public/sessions/{sessionCode}/orders
GET /api/public/sessions/{sessionCode}/orders/{orderId}
POST /api/public/sessions/{sessionCode}/checkout
GET /api/public/sessions/{sessionCode}/payment-status
POST /api/public/sessions/{sessionCode}/payment-cancel
```

Important behavior:

- QR token identifies table.
- Table must be active.
- Branch and restaurant must be active.
- Table must already be occupied/opened.
- Active unexpired session must exist.
- Tenant mismatch returns not found.

### 10.7 Waiter

Routes:

```text
GET /api/waiter/orders/pending-confirmation?branchId=...
POST /api/waiter/orders/{orderId}/confirm?branchId=...
GET /api/waiter/items/ready-to-serve?branchId=...
POST /api/waiter/items/mark-served?branchId=...
POST /api/waiter/orders?branchId=...
```

### 10.8 Kitchen

Routes:

```text
GET /api/kitchen/orders/pending-confirmation?branchId=...
POST /api/kitchen/orders/{orderId}/confirm?branchId=...
POST /api/kitchen/items/confirm?branchId=...
GET /api/kitchen/items/grouped?branchId=...
POST /api/kitchen/items/mark-ready?branchId=...
```

Kitchen grouped items include:

- Menu item.
- Status.
- Note.
- Total quantity.
- Average cooking minutes.
- Priority score.
- Suggested priority level.
- Child order items.

Priority score:

```text
waitingMinutes * 1.5 + avgCookingMinutes * 1.0 + totalQuantity * 0.5
```

### 10.9 Cashier

Routes:

```text
GET /api/cashier/branches/{branchId}/orders
GET /api/cashier/branches/{branchId}/orders/{orderId}
POST /api/cashier/branches/{branchId}/orders/{orderId}/checkout
POST /api/cashier/branches/{branchId}/orders/{orderId}/payment-cancel
```

Cashier can:

- Read active/paid/all orders.
- Search by order number, table, customer, phone, session token.
- Apply voucher.
- Checkout cash.
- Checkout PayOS.
- Cancel pending payment.

Cash checkout behavior:

- Requires amount received.
- Creates `SUCCESS` cash payment.
- Computes change.
- Marks order completed.
- Publishes order update.

PayOS checkout behavior:

- Requires branch PayOS config.
- Creates pending PayOS payment.
- Returns checkout URL/QR/bank details.

### 10.10 Branch Settings

Payment config:

```text
GET /api/owner/branches/{branchId}/payment-config
PUT /api/owner/branches/{branchId}/payment-config
GET /api/manager/branches/{branchId}/payment-config
PUT /api/manager/branches/{branchId}/payment-config
```

Paper vouchers:

```text
GET /api/owner/branches/{branchId}/paper-vouchers
POST /api/owner/branches/{branchId}/paper-vouchers
PUT /api/owner/branches/{branchId}/paper-vouchers/{voucherId}
GET /api/manager/branches/{branchId}/paper-vouchers
POST /api/manager/branches/{branchId}/paper-vouchers
PUT /api/manager/branches/{branchId}/paper-vouchers/{voucherId}
```

### 10.11 Reports

Routes:

```text
GET /api/owner/reports/overview
GET /api/manager/reports/overview
GET /api/admin/reports/dashboard
```

Owner/manager report includes:

- Total revenue.
- Paid revenue.
- Pending revenue.
- Orders.
- Completed orders.
- Average order value.
- Revenue by day.
- Peak hours.
- Top items.
- Branch breakdown.

Admin report includes:

- Total users.
- Total restaurants.
- Total branches.
- Total orders.
- Total revenue.
- Platform growth.
- Revenue by month.

## 11. Realtime Architecture

Backend hubs:

```text
/hubs/cart
/hubs/orders
```

Cart hub:

- `JoinSession(sessionCode)`.
- `UpdateCart(sessionCode, cart)`.
- `ClearCart(sessionCode)`.
- `LeaveSession(sessionCode)`.
- Events: `CartUpdated`, `CartCleared`.
- Storage: memory cache.

Order hub:

- `JoinOrder(sessionCode, orderId)`.
- `LeaveOrder(orderId)`.
- `JoinBranch(branchId)`.
- `LeaveBranch(branchId)`.
- Events: `OrderUpdated`, `BranchOrderUpdated`.

Publisher:

- `SignalROrderUpdatePublisher`.

Published after:

- Place order.
- Public checkout.
- Cashier checkout.
- Kitchen confirm/ready.
- Waiter confirm/served.
- Cancel order.

Current state:

- Tenant FE integrates SignalR clients: `useSharedCart` (`/hubs/cart`), `useOrderUpdates` and `useBranchOrderUpdates` (`/hubs/orders`). Landing/admin FE has no SignalR (not needed).
- Initial loads + payment/order status still use HTTP calls; SignalR layers on realtime updates.
- Multi-instance backend still needs Redis/distributed cache and a SignalR backplane (hub uses memory cache).
- Hub `JoinBranch` should be authorized/hardened before production scale.

## 12. Key User Flows

### 12.1 Admin Onboards Restaurant

```text
Admin opens scannow.site/admin
Admin logs in
Admin creates owner
Admin creates restaurant with slug
Admin/owner creates branch
Owner configures menu/tables/users
Wildcard DNS lets <slug>.scannow.site serve tenant app
```

### 12.2 Owner Sets Up Branch

```text
Owner opens <tenant>.scannow.site/login
Tenant FE sends X-Tenant-Slug
Backend validates owner belongs to tenant
Owner creates branch
Owner creates users
Owner configures menu/tables/payment/vouchers through APIs
```

Current FE:

- Branch/user basics implemented.
- Menu/table/payment/voucher UI missing.

### 12.3 Staff Opens Table

```text
Staff logs in to tenant domain
Staff selects branch/table
Staff opens table
Backend creates QrSession
Table status becomes OCCUPIED
Customer can now scan QR
```

Current FE:

- Implemented in the `me/` waiter shell (open/close session, branch tables).

### 12.4 Customer Orders

```text
Customer scans QR
FE calls GET /api/public/tables/{qrCodeToken}
FE calls POST /api/public/tables/{qrCodeToken}/join
Backend returns session code
FE loads /sessions/{sessionCode}/menu
Customer adds items to local cart
Customer places order
Backend creates or appends order
Kitchen/waiter can confirm
Customer checks out PayOS or pay at cashier
```

Current FE:

- Fully implemented incl. menu-item detail and a realtime order tracking page (`/sessions/{sessionCode}/orders/{orderId}`).
- SignalR cart/order tracking wired (`/hubs/cart`, `/hubs/orders`).
- Needs smoke test in production domain.

### 12.5 Kitchen Prepares

```text
Kitchen sees pending confirmation/grouped queue
Kitchen confirms items/order
Kitchen marks items ready
Backend recalculates order status
Backend publishes order update
Waiter can serve ready items
```

Current FE:

- Implemented in the `me/` kitchen page (grouped queue, confirm, mark ready) with SignalR branch updates.

### 12.6 Cashier Pays

```text
Cashier sees branch orders
Cashier opens order detail
Cashier optionally applies voucher
Cashier chooses CASH or PAYOS
CASH -> SUCCESS payment + change amount
PAYOS -> pending payment link/QR
Cashier can cancel pending payment
```

Current FE:

- Fully implemented (`cashier-dashboard-page`): branch orders, detail, voucher, CASH/PAYOS checkout, cancel pending payment.

## 13. Security And Isolation

Current controls:

- JWT bearer auth.
- Role-based controller authorization.
- Tenant resolution before authentication.
- `AuthService` checks tenant access on login.
- EF Core global query filters cover tenant-related entities.
- Service-level branch ownership checks in many flows.
- Reserved subdomains prevent `api`, `admin`, `business`, etc. from being tenant slugs.

Risks:

- Need integration tests for cross-tenant access.
- Access token is stored in localStorage, so XSS risk exists.
- Admin FE guard is client-side and does not strongly verify `ADMIN` role.
- SignalR `JoinBranch` should authorize branch membership.
- Refresh cookie behavior must be tested across sibling domains.
- Public QR token is globally unique and should be tested for tenant mismatch.

Recommended P0 tests:

- Tenant A header + Tenant B branch ID returns not found/forbidden.
- Tenant A header + Tenant B QR token returns not found.
- Tenant A header + Tenant B session code returns not found.
- Tenant A header + Tenant B order ID returns not found.
- Staff cannot access branch outside assignment.
- Cashier cannot checkout order from another branch.
- Public order cannot include menu item from another branch.

## 14. Operations And Deployment

Production topology:

```text
Landing/Admin FE: scannow.site
Tenant FE: *.scannow.site
Backend API: api.scannow.site
Database: managed PostgreSQL
Storage: Cloudinary
Payment: PayOS
Email: SMTP + Resend for landing lead capture
```

DNS:

```text
A/CNAME scannow.site
CNAME www.scannow.site
CNAME *.scannow.site
CNAME api.scannow.site
```

TLS:

- Apex certificate for `scannow.site`.
- Wildcard certificate for `*.scannow.site`.
- API certificate for `api.scannow.site`.

Backend required env:

```text
ASPNETCORE_ENVIRONMENT=Production
ConnectionStrings__ScanNowDB=...
Jwt__Issuer=...
Jwt__Audience=...
Jwt__SecretKey=...
App__ClientUrl=https://scannow.site
App__FrontendBaseUrl=https://scannow.site
App__AllowedOrigins=https://scannow.site;https://www.scannow.site
App__ProductionDomain=scannow.site
App__TenantBaseDomain=scannow.site
App__QrTablePath=/tables
Cloudinary__CloudName=...
Cloudinary__ApiKey=...
Cloudinary__ApiSecret=...
Email__Host=...
Email__Port=...
Email__Username=...
Email__Password=...
PayOS__ClientId=...
PayOS__ApiKey=...
PayOS__ChecksumKey=...
```

Landing/Admin FE env:

```text
NEXT_PUBLIC_API_URL=https://api.scannow.site
RESEND_API_KEY=...
TO_EMAIL_ADDRESS=...
SENDER_EMAIL_ADDRESS=...
```

Tenant FE env:

```text
NEXT_PUBLIC_API_URL=https://api.scannow.site
NEXT_PUBLIC_DEV_TENANT_SLUG= # local only
```

Production launch checklist:

- DNS live.
- TLS valid.
- `App__ProductionDomain` set.
- `App__TenantBaseDomain` set.
- FE API URLs point to `api.scannow.site`.
- Wildcard tenant FE serves `<tenant>.scannow.site`.
- `business.scannow.site` parked or reserved.
- CORS preflight passes with `Authorization` and `X-Tenant-Slug`.
- QR generation returns tenant URL.
- PayOS return/cancel returns tenant URL.
- Tenant login sends header.
- Cross-tenant tests pass.
- Payment smoke tests pass.
- Backup/restore verified.

## 15. Current Status Matrix

| Area | Status | Notes |
|---|---|---|
| Landing | Supported | i18n landing, lead capture. |
| Platform admin | Supported/partial | Owner/restaurant CRUD and supervision; admin role guard hardening needed. |
| Tenant auth | Supported | Login/refresh/protected routes, tenant header. |
| Owner portal | Supported | Restaurant/branches/users + menu/table/settings/payment-config/voucher/reports. |
| Manager portal | Supported | Same set scoped to managed branch. |
| Staff/Kitchen/Branch-manager ops (`/me/*`) | Supported | Open/close session, availability, waiter confirm/serve, kitchen queue; SignalR branch updates. |
| Cashier portal | Supported | Order list/detail/checkout (cash/PayOS)/voucher/cancel. |
| Public QR ordering | Supported | + menu-item detail and realtime order tracking; needs production smoke test. |
| Payments | Backend supported | PayOS/cash flows; public cash accounting risk. |
| Reports | Supported | Owner/manager overview dashboards wired (Excel export). |
| Realtime | Supported | FE SignalR cart/order clients wired; scale backplane still missing. |
| Tenant isolation | Strong in code | Needs integration tests. |
| Build/lint | Lint pass; build blocked | FE lint+typecheck pass; `pnpm build` fails on built-in `/_global-error` (pre-existing) due to environment dif error (can skip). Backend build pass with warnings. |

## 16. Major Risks And Gaps

P0 before production:

- Verify deployed tenant QR ordering + operations (waiter/kitchen/cashier) + realtime end-to-end.
- Verify PayOS return/cancel on tenant domain.
- Set `App__TenantBaseDomain=scannow.site`.
- Keep secrets out of git/provider logs.
- Add cross-tenant tests.

P1:

- Fix backend dependency vulnerability warnings.
- Align admin FE `TableStatus` string enum (in `scan-now-nextjs`).
- Drop the legacy `MANAGER` FE alias once nothing depends on it.
- SignalR hub `JoinBranch` authorization + backplane for multi-instance.
- Advanced reports/analytics and export breakdowns.

Done (was P1):

- Staff/kitchen operations (me/ shell), owner/manager menu/table/payment-config/voucher UI, FE SignalR clients, owner/manager report dashboards, cashier UI.

P2:

- Business portal.
- Tenant branding.
- Advanced analytics.
- PWA/offline.
- More payment gateways.
- Webhooks/reconciliation/refunds.

## 17. Suggested Slide Deck Structure

Suggested 16-slide story:

1. Title: ScanNow, QR ordering and restaurant operations SaaS.
2. Problem: customer wait time, paper menu, disconnected staff/kitchen/cashier.
3. Product answer: tenant subdomain, QR session, role-based operations.
4. Domain model: scannow.site, api.scannow.site, tenant.scannow.site.
5. User roles: admin, owner, manager, staff, kitchen, cashier, customer.
6. High-level architecture: 3 repos + backend + PostgreSQL + PayOS + Cloudinary + SignalR.
7. Backend architecture: Domain/Application/Infrastructure/Web.
8. Frontend split: landing/admin vs tenant/customer.
9. Data model: Restaurant -> Branch -> Menu/Table/Order/Payment.
10. Tenant isolation: header/subdomain, TenantContext, EF filters, login guard.
11. Customer QR flow: scan -> join session -> menu -> order -> checkout.
12. Operations flow: waiter confirm, kitchen prepare, waiter serve, cashier pay.
13. Payment flow: branch PayOS config, cash, vouchers, pending/success states.
14. Current implementation status: supported vs partial vs placeholder.
15. Production readiness: env, CORS, DNS, smoke tests, warnings.
16. Roadmap: complete shift workflow, realtime, reports, business portal.

## 18. Short Speaking Script

Use this as a concise narrative:

```text
ScanNow is a SaaS platform for restaurants that turns each restaurant into a tenant on its own subdomain. A customer scans a QR code at a table, joins an active session, browses the branch menu, places an order, and checks out through PayOS or pays at the cashier. Behind that customer experience, the restaurant team works by role: waiter confirms and serves, kitchen prepares, cashier takes payment, owner and manager control branches and users, and platform admin onboards restaurants.

Technically, the backend is an ASP.NET Core/.NET 10 API with EF Core/PostgreSQL, Identity/JWT, SignalR, PayOS, Cloudinary, SMTP and QR generation. The frontend is split into two Next.js 16 apps: one for landing and platform admin, one for the tenant app used by owners, staff and customers. The core architecture decision is tenant equals Restaurant, while Branch is the operational unit. Tenant isolation is handled by resolving X-Tenant-Slug or subdomain into TenantContext, then applying EF Core global query filters and login-time tenant access checks.

The system has the backend foundations plus a tenant FE that now implements the full product: owner/manager back office, the me/ waiter shell for staff/kitchen operations, the cashier workflow, the customer QR flow with realtime SignalR cart and order tracking. The main remaining work is production hardening: cross-tenant tests, a SignalR backplane for scale, and end-to-end smoke tests on a live tenant domain.
```

## 19. One-Page Summary For Slide Notes

ScanNow current state:

- Backend foundation: strong.
- Domain model: clear.
- Tenant isolation: implemented, needs tests.
- Customer QR flow: fully implemented incl. realtime tracking.
- Admin: usable.
- Owner/manager: full back office (menu/table/settings/voucher/reports).
- Staff/kitchen/cashier: implemented (me/ shell + cashier UI).
- Realtime: backend ready, FE SignalR clients wired.
- Production: needs env verification, smoke tests, and dependency warning cleanup.

Most important decisions:

- Tenant = Restaurant.
- Branch is operational unit.
- `business.scannow.site` is future-only.
- `App:ProductionDomain` is for CORS.
- `App:TenantBaseDomain` is for tenant URL generation.
- Public cash checkout should mean pay at cashier until cashier confirms.

