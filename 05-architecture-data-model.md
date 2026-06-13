# Architecture & Data Model

## 1. System Overview

ScanNow co 3 codebase chinh:

```text
Backend API              /home/nhat/fpt/ScanNow
Landing/Admin FE         /home/nhat/fpt/scan-now-nextjs
Tenant/Portal FE         /home/nhat/fpt/scan-now-customer
```

Recommended production topology:

```text
scannow.site                  -> scan-now-nextjs
scannow.site/admin            -> scan-now-nextjs
*.scannow.site                -> scan-now-customer
api.scannow.site              -> ScanNow backend
PostgreSQL                    -> application database
Cloudinary                    -> menu image storage
PayOS                         -> online payment link/QR
Resend                        -> landing lead email
SMTP                          -> auth/email verification
SignalR                       -> cart/order realtime
```

## 2. Backend Architecture

Backend stack:

- ASP.NET Core Web API.
- .NET 10 target from build artifacts/project.
- Entity Framework Core.
- PostgreSQL.
- ASP.NET Identity.
- JWT bearer auth.
- Refresh token repository/cookie flow.
- FluentValidation.
- SignalR.
- QRCoder.
- PayOS integration.
- Cloudinary/file storage integration.
- SMTP email.

Layering:

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
  Interfaces
  Mappers

ScanNow.Infrastructure
  EF DbContext
  Configurations
  Repositories
  External services
  Tenancy implementation

ScanNow.Web
  Controllers
  Middleware
  Hubs
  DI
  Program.cs
```

## 3. Frontend Architecture

### 3.1 Landing/Admin FE

Stack:

- Next.js App Router.
- React.
- TypeScript.
- Tailwind.
- next-intl.
- Zustand.
- TanStack Query.
- Axios.
- Resend API route.

Responsibilities:

- Public landing.
- i18n pages.
- Lead capture.
- Platform admin pages.

### 3.2 Tenant/Portal FE

Stack:

- Next.js App Router.
- React.
- TypeScript.
- Tailwind.
- Zustand.
- TanStack Query.
- Axios.

Responsibilities intended:

- Tenant login.
- Owner/manager/staff/kitchen/cashier portal.
- Public QR customer ordering.

Current responsibilities implemented (post `feature/SCRUM-30-customer-qr-session` merge):

- Login, tenant slug extraction, `X-Tenant-Slug` header, role-based redirect/guard.
- Owner/manager back office: restaurant, branches, users, category + menu-item management, table/QR management, branch orders, branch settings (VAT/service charge, PayOS payment-config, paper voucher), reports overview dashboard (Excel export).
- `me/` waiter shell for staff/kitchen/branch-manager: open/close table sessions, branch menu availability, waiter confirm/serve, kitchen grouped queue.
- Cashier: order list/detail, cash/PayOS checkout with voucher, cancel pending payment.
- Public QR/menu/menu-item/checkout/order-tracking/payment-result pages.
- SignalR clients for `/hubs/cart` and `/hubs/orders` (customer cart, customer order tracking, branch order updates).

## 4. Tenant Architecture

Tenant = `Restaurant`.

Resolution:

```text
Request
  -> TenantResolutionMiddleware
    -> X-Tenant-Slug if present
    -> else host subdomain if not reserved
    -> Restaurant lookup by slug and IsActive
    -> TenantContext.RestaurantId/Slug
  -> AuthService prevents login if user is not Owner or BranchStaff of the current Tenant
  -> EF DbContext global query filters apply to ALL tenant entities (Branch, Category, MenuItem, Order, etc.)
```

Current EF filters:

```text
Restaurant.Id == TenantContext.RestaurantId
Branch.RestaurantId == TenantContext.RestaurantId
BranchStaff.Branch.RestaurantId == TenantContext.RestaurantId
(And so on for Category, MenuItem, Order, Payment, etc.)
```

When tenant unresolved:

- Filter disabled.
- Admin/system routes can see platform-wide data if role permits.

Reserved host labels:

```text
www, api, admin, app, business, localhost, staging
```

Product implication:

- `pho24.scannow.site` resolves `pho24`.
- `api.scannow.site` is not tenant; FE must send header.
- `scannow.site/admin` should not resolve tenant.

## 5. Data Model

### 5.1 Identity

`ApplicationUser`

Key fields:

- Email
- UserName
- FullName
- AvatarUrl
- IsActive
- IsBanned
- CreatedAt
- UpdatedAt

Relationships:

- Owns restaurants.
- Manages branches.
- Assigned to branches through `BranchStaff`.
- Can confirm/cancel orders.
- Can process/refund payments.

`ApplicationRole`

Roles:

- `ADMIN`
- `OWNER`
- `BRANCH_MANAGER`
- `STAFF`
- `KITCHEN`
- `CASHIER`

### 5.2 Restaurant and Branch

`Restaurant`

- Id
- OwnerId
- Name
- Slug
- LogoUrl
- Description
- IsActive
- CreatedAt
- UpdatedAt

Indexes:

- OwnerId
- Slug unique

`Branch`

- Id
- RestaurantId
- ManagerId
- Name
- Slug
- Address
- Phone
- Email
- OpenTime
- CloseTime
- IsActive
- VatPercent
- ServiceChargePercent
- ServiceChargeFixed
- CreatedAt
- UpdatedAt

Indexes:

- RestaurantId
- ManagerId
- `(RestaurantId, Slug)` unique
- IsActive

Design meaning:

- Restaurant is tenant identity.
- Branch is operational boundary.
- Tax/service/payment/menu/table/order are branch-scoped.

### 5.3 Staff Assignment

`BranchStaff`

- BranchId
- UserId
- AssignedById
- AssignedAt

Indexes:

- `(BranchId, UserId)` unique
- BranchId
- UserId

Current business rule:

- Managed users must belong to exactly one branch.

### 5.4 Menu

`Category`

- BranchId
- Name
- Description
- ImageUrl
- DisplayOrder
- IsActive
- CreatedAt
- UpdatedAt

`MenuItem`

- BranchId
- CategoryId
- Name
- Description
- ImageUrl
- Price
- CostPrice
- PreparationTime
- DisplayOrder
- IsAvailable
- IsFeatured
- IsActive
- CreatedAt
- UpdatedAt

`MenuItemPriceHistory`

- MenuItemId
- OldPrice
- NewPrice
- ChangedById
- ChangedAt
- Note

Design meaning:

- Menu is branch-specific.
- Public customer menu only includes active categories and active/available items.

### 5.5 Tables and QR Sessions

`RestaurantTable`

- BranchId
- TableNumber
- Capacity
- QrCodeToken
- QrCodeUrl
- QrCodeImageUrl
- Status
- IsActive
- CreatedAt
- UpdatedAt

Table status:

- `AVAILABLE`
- `OCCUPIED`
- `RESERVED`
- `DISABLED`

`QrSession`

- TableId
- BranchId
- ActiveOrderId
- SessionToken
- CustomerIdentifier
- IsActive
- ExpiresAt
- CreatedAt
- UpdatedAt

Session behavior:

- Active session marks table occupied.
- Closing session marks table available.
- Session token is used by public customer APIs.
- ActiveOrderId lets customer add more items into same order.

### 5.6 Orders and Items

`Order`

- BranchId
- TableId
- OrderNumber
- CustomerName
- CustomerPhone
- CustomerNote
- SubTotal
- VatPercent
- VatAmount
- ServiceChargePercent
- ServiceChargeAmount
- DiscountAmount
- TotalAmount
- Status
- OrderSource
- StaffNote
- ConfirmedById
- CancelledById
- CancelReason
- CreatedAt
- ConfirmedAt
- PreparingAt
- ReadyAt
- ServedAt
- CompletedAt
- CancelledAt
- UpdatedAt

`OrderItem`

- OrderId
- MenuItemId
- MenuItemName
- UnitPrice
- Quantity
- SubTotal
- Note
- EstimatedCookingMinutes
- Status
- CreatedAt
- ConfirmedAt
- CookingStartedAt
- ReadyAt
- ServedAt
- CancelledAt
- UpdatedAt

Order source:

- `QR`
- `STAFF_MANUAL`

### 5.7 Payments

`Payment`

- OrderId
- Amount
- Method
- Status
- AmountReceived
- ChangeAmount
- TransactionId
- GatewayOrderId
- GatewayResponseCode
- GatewayResponseData
- PaymentUrl
- RefundTransactionId
- RefundReason
- RefundedAt
- RefundedById
- ProcessedById
- CreatedAt
- PaidAt
- UpdatedAt

Payment method enum:

- `CASH`
- `VNPAY`
- `MOMO`
- `BANK_TRANSFER`
- `PAYOS`

Currently supported by validators:

- Public checkout: `PAYOS`, `CASH`.
- Cashier checkout: `PAYOS`, `CASH`.

Payment status:

- `PENDING`
- `SUCCESS`
- `FAILED`
- `REFUNDED`

### 5.8 Branch Payment Config

`BranchPaymentConfig`

Fields inferred from service/config:

- BranchId
- CashEnabled
- PayOsEnabled
- PayOsClientId
- PayOsApiKey
- PayOsChecksumKey
- DefaultMethod
- CreatedAt
- UpdatedAt

Design:

- Payment config is branch-level, not restaurant-level.
- Cash must remain enabled.
- PayOS requires complete credentials.

### 5.9 Paper Voucher

`PaperVoucher`

- BranchId
- Code
- Name
- Description
- DiscountType
- DiscountValue
- MinOrderAmount
- MaxDiscountAmount
- Quantity
- UsedCount
- ValidFrom
- ValidUntil
- IsActive
- CreatedAt
- UpdatedAt

Discount type:

- `PERCENT`
- `FIXED_AMOUNT`

Voucher QR payload:

```text
SCANNOW-VOUCHER:{branchId}:{code}
```

### 5.10 Notifications, Audit, Ratings

Entities exist:

- `Notification`
- `AuditLog`
- `ItemRating`
- `DiscountCode`

Current product exposure:

- Notification/audit/rating entities are present but not fully surfaced in FE docs.
- Paper voucher is active feature.
- DiscountCode appears as entity but main cashier discount flow uses PaperVoucher.

## 6. Order Lifecycle

### 6.1 QR order creation

```text
Active QrSession
  -> customer places order
  -> validate menu items
  -> create Order + OrderItems
  -> set QrSession.ActiveOrderId
  -> publish OrderUpdated + BranchOrderUpdated
```

If session already has active order:

```text
Append new items
  -> fail pending payments
  -> increase subtotal/tax/service/total
  -> recalculate status
  -> publish update
```

### 6.2 Confirmation and kitchen flow

```text
Pending item
  -> Confirmed
  -> Ready
  -> Served
```

Order status recalculated from item statuses:

- All served -> `Served`.
- Some served -> `PartiallyServed`.
- All ready -> `ReadyToServe`.
- Some ready -> `PartiallyReady`.
- Confirmed/cooking -> `Confirmed`/`Preparing` depending logic.

### 6.3 Checkout flow

Public PayOS:

```text
Order
  -> create pending PayOS Payment
  -> return payment link/QR
  -> poll payment-status
  -> if gateway paid: payment SUCCESS, order Completed
```

Public cash:

```text
Order
  -> create CASH Payment PENDING
  -> order Completed
```

Cashier cash:

```text
Order
  -> apply voucher optional
  -> validate amountReceived >= total
  -> create CASH Payment SUCCESS
  -> set paidAt, changeAmount, processedBy
  -> order Completed
```

Cashier PayOS:

```text
Order
  -> apply voucher optional
  -> create pending PayOS Payment
  -> return payment link/QR
```

## 7. Realtime Architecture

### 7.1 Cart hub

Hub:

```text
/hubs/cart
```

Groups:

- Session code group.

Events:

- `CartUpdated`
- `CartCleared`

Storage:

- Memory cache through CartService.

Risks:

- Not durable.
- Not shared across multiple backend instances.
- No explicit hub auth/session verification in hub methods.

### 7.2 Order hub

Hub:

```text
/hubs/orders
```

Groups:

- `order:{orderId}`
- `branch:{branchId}`

Events:

- `OrderUpdated`
- `BranchOrderUpdated`

Publisher:

- `SignalROrderUpdatePublisher`.

Published after:

- Place order.
- Public checkout.
- Cashier checkout.
- Kitchen confirm/ready.
- Waiter confirm/served.
- Cancel order.

## 8. Tenant Data Isolation Analysis

Strengths:

- Backend has tenant context.
- Tenant context is populated before authentication/authorization.
- Authentication (`AuthService.LoginAsync`) checks if the user is an Owner or BranchStaff for the current Tenant, preventing cross-tenant login.
- Global Query Filters in `ApplicationDbContext` apply to **all** tenant-related entities (Branch, Order, MenuItem, etc.), ensuring strict application-level isolation without needing to manually include `BranchId` filters everywhere.
- Service methods often check branch ownership/branchId explicitly (belt and suspenders approach).

Risks:

- Global filters traverse navigation properties for some entities (`OrderItem`, `Payment`, `MenuItemPriceHistory`), so query correctness should still be covered by integration tests.
- Public QR token lookup queries `RestaurantTable` by token and includes Branch. This should be tested because QR token is globally unique and not tenant-prefixed.
- SignalR `JoinBranch(branchId)` does not authenticate/authorize branch membership in hub method.

Recommended tests:

1. Tenant A header + Branch B ID on every branch route returns not found/forbidden.
2. Tenant A header + QR token from Tenant B returns not found.
3. Tenant A header + sessionCode from Tenant B returns not found.
4. Tenant A header + orderId from Tenant B returns not found.
5. Unauthenticated SignalR cannot join arbitrary branch groups or data is harmless.

## 9. Deployment Architecture

### 9.1 Production domains

```text
scannow.site
www.scannow.site
*.scannow.site
api.scannow.site
```

### 9.2 Recommended services

Landing/Admin:

- Vercel/Node standalone host.
- Domain: `scannow.site`.

Tenant FE:

- Vercel/Node standalone host with wildcard domain.
- Domain: `*.scannow.site`.

Backend:

- Render/Fly/Azure/App Service/container host.
- Domain: `api.scannow.site`.

Database:

- Managed PostgreSQL.

### 9.3 CORS

Backend code supports:

- explicit origins from `App:FrontendBaseUrl`, `App:ClientUrl`, `App:AllowedOrigins`.
- wildcard subdomain via `App:ProductionDomain`.

Required:

```text
App:ProductionDomain=scannow.site
App:TenantBaseDomain=scannow.site
```

`App:ProductionDomain` is used by CORS wildcard checks. `App:TenantBaseDomain` is used by `ITenantUrlBuilder` to generate tenant QR/payment URLs. If `TenantBaseDomain` is missing, generated URLs fall back to the platform base URL.

### 9.4 Cookies

Refresh-token cookie must be reviewed for:

- Domain.
- SameSite.
- Secure.
- HttpOnly.

If frontend and API are on sibling domains (`pho24.scannow.site` and `api.scannow.site`), cookie behavior must be explicitly tested.

## 10. Known Architecture Gaps

- QR URL generation is tenant-aware through `ITenantUrlBuilder`; old QR records created before the change may need regeneration/backfill.
- PayOS return/cancel redirect is tenant-aware through `ITenantUrlBuilder`; production smoke test with real PayOS config is still required.
- Tenant FE has the full customer QR/menu/checkout/order-tracking/payment flow plus realtime cart/order via SignalR (`useSharedCart`, `useOrderUpdates`, `useBranchOrderUpdates`).
- Tenant FE cashier is a full order list/detail/checkout/cancel workflow (no longer placeholders); the `me/` shell covers staff/kitchen/branch-manager operations; owner/manager cover menu/table/settings/voucher/reports.
- FE auth/user-management role enums include `CASHIER` and the owner/manager role pickers expose it.
- Known build issue: `pnpm build` fails prerendering the built-in `/_global-error` page (`Cannot read properties of null (reading 'useContext')`) — pre-existing on `main`, Next 16 + React 19 + `reactCompiler`; must be fixed before a production build.
- `ScanNow.Web/appsettings.json` currently includes `App:ProductionDomain` but not `App:TenantBaseDomain`; deployment env must set it.
- Keep production secrets out of tracked config; rotate any values that may have been exposed through local files, logs, or deployment providers.
- `SITE_CONFIG.baseUrl` has been moved to `https://scannow.site`; tenant-specific metadata is still a future improvement.
- SignalR backend hubs use memory cache with no backplane; multi-instance deploy needs Redis/distributed cache.
- Backend supports enum string serialization; some FE types expect numeric table status.
