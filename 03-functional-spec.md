# Functional Specification

## 1. Role Matrix

| Role | Domain | Primary routes | Main responsibilities |
|---|---|---|---|
| Public visitor | `scannow.site` | `/`, `/vi`, `/en` | Read landing, submit lead form |
| Platform admin | `scannow.site/admin` | `/admin`, `/admin/restaurants`, `/admin/branches/...` | Manage owners/restaurants, supervise branches/menu/tables |
| Owner | `<tenant>.scannow.site` | `/owner/...` | Manage restaurant, branches, users, settings, reports |
| Branch manager | `<tenant>.scannow.site` | `/manager/...` | Manage assigned branch users/operations |
| Staff/Waiter | `<tenant>.scannow.site` | `/staff/...` | Open/close tables, confirm orders, serve items |
| Kitchen | `<tenant>.scannow.site` | `/kitchen/...` | Confirm items, group kitchen queue, mark ready |
| Cashier | `<tenant>.scannow.site` | `/cashier/...` | List orders, apply voucher, checkout cash/PayOS |
| Customer | `<tenant>.scannow.site` | `/tables/{token}`, `/sessions/...` | QR order, shared cart, checkout, track order |

Backend roles:

```text
ADMIN, OWNER, BRANCH_MANAGER, STAFF, KITCHEN, CASHIER
```

Current frontend status:

- Tenant FE has `CASHIER` auth redirect and placeholder `/cashier/dashboard`, `/cashier/orders` pages.
- Tenant FE has customer QR/menu/checkout/payment result routes.
- Customer QR flow is implemented at basic route/UI level, but SignalR cart/order client and production smoke tests remain pending.
- Remaining gaps are full cashier order workflow, staff/kitchen dashboards, realtime tracking hardening, and owner role-picker alignment for cashier creation.

## 2. Landing Module

### 2.1 Goals

- Communicate product value.
- Convert interested restaurants into leads.
- Support Vietnamese and English locales.

### 2.2 Current implementation

Repo: `scan-now-nextjs`

Routes:

- `/` redirects to default locale.
- `/vi` renders landing.
- `/en` renders landing.
- `/api/lead-capture` sends lead email via Resend.

Lead payload:

```json
{
  "phone": "string",
  "email": "string",
  "location": "string",
  "locale": "vi | en",
  "source": "hero-primary | feature-payment | final-primary"
}
```

### 2.3 Requirements

- Landing must run on `https://scannow.site`.
- SEO metadata and sitemap must use `https://scannow.site`.
- Lead form must validate phone/email/location.
- Missing email config must return 500 without exposing secrets.
- Admin route `/admin` must not be localized accidentally.

## 3. Platform Admin Module

### 3.1 Authentication

Admin uses common `/api/auth/login`.

Current FE:

- Admin login card stores JWT in localStorage.
- Zustand stores user state.
- Admin page checks token + login state client-side.

Requirement:

- Only `ADMIN` should access admin pages.
- Current FE should add explicit role guard, not just token presence.

### 3.2 Owner Management

Admin can:

- List owners with pagination/search/status filters.
- Create owner.
- Update owner.
- Ban/unban owner.
- List available owners not yet attached to restaurants.

Backend routes:

- `GET /api/admin/users/owners`
- `GET /api/admin/users/owners/available`
- `POST /api/admin/users/owners`
- `PUT /api/admin/users/owners/{id}`
- `PATCH /api/admin/users/owners/{id}/ban`
- `PATCH /api/admin/users/owners/{id}/unban`

Owner fields:

- fullName
- username
- email
- phoneNumber
- password for create
- active/banned status
- restaurantId/restaurantName if assigned

### 3.3 Restaurant/Tenant Management

Admin can:

- List restaurants.
- Create restaurant with owner and slug.
- View restaurant by GUID or slug.
- Update restaurant.
- Ban/unban restaurant.
- View branches by restaurant GUID or slug.
- View branch by GUID or branch slug.

Tenant rule:

- `Restaurant.Slug` is global unique.
- This slug becomes subdomain: `{slug}.scannow.site`.

Backend routes:

- `GET /api/admin/restaurants`
- `GET /api/admin/restaurants/{id}`
- `GET /api/admin/restaurants/by-slug/{slug}`
- `GET /api/admin/restaurants/{id}/branches`
- `GET /api/admin/restaurants/by-slug/{slug}/branches`
- `GET /api/admin/restaurants/{id}/branches/{branchId}`
- `GET /api/admin/restaurants/by-slug/{slug}/branches/{branchSlug}`
- `POST /api/admin/restaurants`
- `PUT /api/admin/restaurants/{id}`
- `PATCH /api/admin/restaurants/{id}/ban`
- `PATCH /api/admin/restaurants/{id}/unban`

### 3.4 Admin Supervision

Admin can read:

- Branch categories.
- Branch menu items.
- Menu item detail and price history.
- Branch tables.
- Branch table detail.
- Active sessions.

Current FE supports admin read views/tabs. It does not expose create/update for menu/table from admin, based on service mapping.

## 4. Tenant Back Office Module

### 4.1 Tenant Entry

Moi user thuoc nha hang (Owner, Branch Manager, Staff/Waiter, Kitchen, Cashier) deu truy cap thong qua tenant domain cua nha hang ho va khong dung `business.scannow.site`:

```text
https://{restaurantSlug}.scannow.site/login
```

Sau khi dang nhap, he thong se kiem tra role va redirect ve route phu hop tren cung tenant domain nay.

Frontend:

- Extracts `{restaurantSlug}` from hostname.
- Sends `X-Tenant-Slug` on API requests.

Backend:

- Resolves tenant to active restaurant.
- Applies branch filter for tenant-scoped queries.

### 4.2 Auth

Common auth routes:

- `POST /api/auth/login`
- `POST /api/auth/refresh-token`
- `POST /api/auth/logout`
- `POST /api/auth/register`
- `POST /api/auth/login-google`
- `POST /api/auth/check-email-exist`
- `POST /api/auth/check-username-exist`
- `POST /api/auth/change-password`
- `POST /api/auth/resend-email-verification`
- `POST /api/auth/verify-email`

Access token:

- Sent as Bearer token.
- FE stores token in localStorage.

Refresh token:

- Backend uses cookie flow.
- FE refresh request uses `withCredentials`.

### 4.3 Owner Restaurant Management

Owner can:

- View own restaurant.
- Update restaurant name, slug, logo, description.

Routes:

- `GET /api/owner/restaurant/me`
- `PUT /api/owner/restaurant/me`

Functional constraints:

- Owner should only see restaurant they own.
- Updating slug changes tenant URL; product must define redirect/notification behavior.
- Slug update should be carefully controlled in production because it changes subdomain.

### 4.4 Branch Management

Owner can:

- Create branch.
- List branches.
- View branch detail.
- Update branch.
- Set active/inactive.

Branch fields:

- name
- slug
- address
- phone
- email
- openTime
- closeTime
- vatPercent
- serviceChargePercent
- serviceChargeFixed
- isActive

Routes:

- `POST /api/owner/branches`
- `GET /api/owner/branches`
- `GET /api/owner/branches/{id}`
- `PUT /api/owner/branches/{id}`
- `PATCH /api/owner/branches/{id}/active`
- `PATCH /api/owner/branches/{id}/inactive`

Branch rules:

- Branch slug is unique within restaurant.
- Branch belongs to tenant restaurant.
- Branch can have manager.
- Branch is the operational root for menu, table, order, payment config.

### 4.5 User Management

Owner can manage:

- `BRANCH_MANAGER`
- `STAFF`
- `KITCHEN`
- `CASHIER`

Branch manager can manage:

- Subordinate users scoped to managed branch.
- Backend currently permits manager users API for branch manager.

Routes:

- `GET /api/owner/users`
- `POST /api/owner/users`
- `PUT /api/owner/users/{id}`
- `PATCH /api/owner/users/{id}/ban`
- `PATCH /api/owner/users/{id}/unban`
- `GET /api/manager/users`
- `POST /api/manager/users`
- `PUT /api/manager/users/{id}`
- `PATCH /api/manager/users/{id}/ban`
- `PATCH /api/manager/users/{id}/unban`

Validation:

- Managed user must belong to exactly one branch.
- Role must be one of backend managed roles.

FE gap:

- Tenant FE user-management types include `CASHIER`, but owner role picker currently exposes only `BRANCH_MANAGER`, `STAFF`, `KITCHEN`.
- Manager UI appears to allow only `KITCHEN` and `STAFF`.

## 5. Menu Management Module

### 5.1 Category Management

Owner/branch manager can:

- List categories.
- Get category detail.
- Create category.
- Update category.
- Reorder categories.
- Activate/deactivate category.

Routes:

- `GET /api/owner/branches/{branchId}/categories`
- `GET /api/owner/branches/{branchId}/categories/{id}`
- `POST /api/owner/branches/{branchId}/categories`
- `PUT /api/owner/branches/{branchId}/categories/{id}`
- `PATCH /api/owner/branches/{branchId}/categories/reorder`
- `PATCH /api/owner/branches/{branchId}/categories/{id}/active`
- `PATCH /api/owner/branches/{branchId}/categories/{id}/inactive`

Category fields:

- name
- description
- imageUrl
- displayOrder
- isActive

### 5.2 Menu Item Management

Owner/branch manager can:

- List menu items.
- View item detail.
- Upload item image.
- Create item under category.
- Update item.
- Activate/deactivate item.
- Toggle availability.
- Toggle featured.
- Bulk availability.
- Reorder items.
- Update price with price history.
- View price history.

Routes:

- `GET /api/owner/branches/{branchId}/menu-items`
- `GET /api/owner/menu-items/{id}`
- `POST /api/owner/menu-items/images`
- `POST /api/owner/branches/{branchId}/categories/{categoryId}/menu-items`
- `PUT /api/owner/menu-items/{id}`
- `PATCH /api/owner/menu-items/{id}/active`
- `PATCH /api/owner/menu-items/{id}/inactive`
- `PATCH /api/owner/branches/{branchId}/menu-items/reorder`
- `PATCH /api/owner/menu-items/{id}/toggle-available`
- `PATCH /api/owner/branches/{branchId}/menu-items/bulk-availability`
- `PATCH /api/owner/menu-items/{id}/toggle-featured`
- `PATCH /api/owner/menu-items/{id}/price`
- `GET /api/owner/menu-items/{id}/price-history`

Staff/kitchen/cashier can read/toggle availability via `/api/me/...` routes.

## 6. Table & QR Session Module

### 6.1 Table Management

Owner/branch manager can:

- List tables by branch.
- Create table.
- Update table.
- Activate/deactivate table.
- Update table status, except occupied status is controlled by session flow.
- Regenerate QR.
- Download QR image.

Routes:

- `GET /api/owner/branches/{branchId}/tables`
- `GET /api/owner/branches/{branchId}/tables/{id}`
- `GET /api/owner/branches/{branchId}/tables/{id}/orders`
- `GET /api/owner/branches/{branchId}/orders`
- `POST /api/owner/branches/{branchId}/tables`
- `PUT /api/owner/tables/{id}`
- `PATCH /api/owner/tables/{id}/status`
- `PATCH /api/owner/tables/{id}/activate`
- `PATCH /api/owner/tables/{id}/deactivate`
- `POST /api/owner/tables/{id}/regenerate-qr`
- `GET /api/owner/tables/{id}/qr-image`

Table statuses:

```text
AVAILABLE, OCCUPIED, RESERVED, DISABLED
```

FE gap:

- Admin FE defines `TableStatus` as numeric `0 | 1 | 2 | 3`, while backend now serializes enums as strings.

### 6.2 Opening/Closing Table Sessions

Staff/cashier/kitchen can:

- Open a table session.
- Close a session.
- List my branch tables.
- View table detail.
- View table active orders.

Routes:

- `POST /api/me/branches/{branchId}/tables/{tableId}/open`
- `PATCH /api/me/sessions/{sessionId}/close`
- `GET /api/me/branches/{branchId}/tables`
- `GET /api/me/tables/{id}`
- `GET /api/me/tables/{id}/orders`

Session behavior:

- Open table creates `QrSession`.
- Session token is 6 characters, excluding ambiguous characters.
- Session expires after 6 hours.
- Opening sets table status `OCCUPIED`.
- Closing sets session inactive and table status `AVAILABLE`.

## 7. Public Customer QR Ordering

### 7.1 Intended Flow

1. Staff opens table session.
2. Customer scans QR.
3. Customer lands on tenant URL:

```text
https://{restaurantSlug}.scannow.site/tables/{qrCodeToken}
```

4. FE gets table info.
5. Customer joins active session.
6. FE loads session menu.
7. Customer adds items to cart.
8. Customer places order.
9. Customer tracks order status; realtime tracking is intended but FE SignalR client is not wired yet.
10. Customer checks out via PayOS or cash-at-counter.

### 7.2 Backend APIs

- `GET /api/public/tables/{qrCodeToken}`
- `POST /api/public/tables/{qrCodeToken}/join`
- `POST /api/public/sessions/join`
- `GET /api/public/sessions/{sessionCode}/menu`
- `POST /api/public/sessions/{sessionCode}/orders`
- `POST /api/public/sessions/{sessionCode}/checkout`
- `GET /api/public/sessions/{sessionCode}/payment-status`
- `POST /api/public/sessions/{sessionCode}/payment-cancel`
- `GET /api/public/sessions/{sessionCode}/orders/{orderId}`

### 7.3 Current FE Implementation and Gaps

Implemented in `scan-now-customer`:

- `/tables/[qrCodeToken]` auto-checks table, auto-joins active table session, then redirects to menu.
- `/sessions/[sessionCode]/menu` loads categories/items, supports search/category filter, stores cart in local storage, and places order.
- `/sessions/[sessionCode]/checkout` loads order detail and starts PayOS or cash-at-counter checkout.
- `/payment/return` checks payment status.
- `/payment/cancel` cancels pending public payment.

Current gaps:

- No FE SignalR client integration for `/hubs/cart` or `/hubs/orders`.
- No rich standalone order tracking timeline/page after checkout.
- Runtime smoke test with deployed tenant domain, active table session, real branch/menu data, and PayOS config is still required.
- Public cash checkout should be presented as "pay at cashier" until cashier confirms payment.

## 8. Shared Cart Module

SignalR hub:

```text
/hubs/cart
```

Methods:

- `JoinSession(sessionCode)` -> returns current `CartDto`.
- `UpdateCart(sessionCode, cart)` -> broadcasts `CartUpdated`.
- `ClearCart(sessionCode)` -> broadcasts `CartCleared`.
- `LeaveSession(sessionCode)`.

Cart DTO:

- items
- totalAmount computed from items

Cart item:

- menuItemId
- menuItemName
- price
- quantity
- specialRequest
- imageUrl

Limitations:

- Cart service uses memory cache, not durable storage.
- No explicit session authorization in hub methods.
- Multiple app instances would not share memory cache without distributed cache/backplane.
- Current tenant FE customer cart uses `src/lib/customer-cart.ts` local storage; SignalR shared cart hub is not wired on the frontend yet.

## 9. Order Lifecycle

### 9.1 Statuses

Order:

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

Order item:

```text
Pending
Confirmed
Cooking
Ready
Served
Cancelled
```

### 9.2 Customer QR Order

`POST /api/public/sessions/{sessionCode}/orders`

Behavior:

- Validates session active/not expired.
- Validates menu items active/available and in same branch.
- Creates new order if session has no active order.
- Appends new items if session has active order.
- Marks pending payments failed when adding more items to existing order.
- Calculates subtotal, VAT, service charge percent.
- Publishes realtime order update.

Known calculation gap:

- Branch has `ServiceChargeFixed`, but order calculation currently uses only `ServiceChargePercent`.

### 9.3 Waiter Manual Order

`POST /api/waiter/orders?branchId={branchId}`

Behavior:

- Staff/cashier/branch manager can create order for table.
- Uses `OrderSource.STAFF_MANUAL`.
- May append to active order for same table/session.
- Publishes realtime update.

### 9.4 Confirmation

Waiter routes:

- `GET /api/waiter/orders/pending-confirmation?branchId=...`
- `POST /api/waiter/orders/{orderId}/confirm?branchId=...`

Kitchen routes:

- `GET /api/kitchen/orders/pending-confirmation?branchId=...`
- `POST /api/kitchen/orders/{orderId}/confirm?branchId=...`
- `POST /api/kitchen/items/confirm?branchId=...`

Kitchen can confirm order or individual items.

### 9.5 Kitchen Preparation

Routes:

- `GET /api/kitchen/items/grouped?branchId=...&status=...`
- `POST /api/kitchen/items/mark-ready?branchId=...`

Grouped kitchen item includes:

- menuItemId
- menuItemName
- status
- note
- totalQuantity
- averageCookingMinutes
- priorityScore
- suggestedPriorityLevel
- oldestConfirmedAt
- waitingMinutes
- child order items

Priority score formula in code:

```text
waitingMinutes * 1.5 + avgCookingMinutes * 1.0 + totalQuantity * 0.5
```

### 9.6 Serving

Routes:

- `GET /api/waiter/items/ready-to-serve?branchId=...`
- `POST /api/waiter/items/mark-served?branchId=...`

Behavior:

- Items must be `Ready` before served.
- Order status recalculated after item updates.

## 10. Checkout & Payment

### 10.1 Public Checkout

Route:

```text
POST /api/public/sessions/{sessionCode}/checkout
```

Supported methods:

```text
PAYOS, CASH
```

Behavior:

- Requires active session and active order.
- If successful payment exists: conflict.
- If pending PayOS payment exists: return existing payment info.
- CASH path:
  - creates `Payment` with `PENDING`.
  - marks order `Completed`.
  - publishes realtime update.
- PAYOS path:
  - requires branch PayOS config.
  - creates payment link.
  - creates `Payment` with `PENDING`.
  - returns checkout URL/QR/payment account info.

Important gap:

- Public CASH leaves payment `PENDING` while order is `Completed`. Cashier checkout has better cash accounting.

### 10.2 Cashier Checkout

Routes:

- `GET /api/cashier/branches/{branchId}/orders`
- `GET /api/cashier/branches/{branchId}/orders/{orderId}`
- `POST /api/cashier/branches/{branchId}/orders/{orderId}/checkout`
- `POST /api/cashier/branches/{branchId}/orders/{orderId}/payment-cancel`

Cashier checkout:

- Supports `CASH` and `PAYOS`.
- Can apply paper voucher.
- CASH requires `amountReceived`.
- CASH creates payment `SUCCESS`, sets paidAt, amountReceived, changeAmount, processedBy.
- PAYOS creates pending payment link.
- Pending payment must be cancelled before switching to cash.

### 10.3 Branch Payment Config

Routes:

- `GET /api/owner/branches/{branchId}/payment-config`
- `PUT /api/owner/branches/{branchId}/payment-config`
- `GET /api/manager/branches/{branchId}/payment-config`
- `PUT /api/manager/branches/{branchId}/payment-config`

Rules:

- Cash must remain enabled.
- PayOS can be enabled only when credentials complete.
- Default method must be CASH or PAYOS.
- Credentials are masked in response.

### 10.4 Paper Voucher

Routes:

- `GET /api/owner/branches/{branchId}/paper-vouchers`
- `POST /api/owner/branches/{branchId}/paper-vouchers`
- `PUT /api/owner/branches/{branchId}/paper-vouchers/{voucherId}`
- Manager equivalents under `/api/manager/...`

Voucher:

- code
- name
- description
- discountType: `PERCENT` or `FIXED_AMOUNT`
- discountValue
- minOrderAmount
- maxDiscountAmount
- quantity
- usedCount
- validFrom/validUntil
- isActive
- qrPayload: `SCANNOW-VOUCHER:{branchId}:{code}`

## 11. Reports

Routes:

- `GET /api/owner/reports/overview`
- `GET /api/manager/reports/overview`
- `GET /api/admin/reports/dashboard`

Owner/manager report includes:

- totalRevenue
- paidRevenue
- pendingRevenue
- totalOrders
- completedOrders
- averageOrderValue
- revenueByDay
- peakHours
- topItems
- branches

Admin dashboard includes:

- totalUsers
- totalRestaurants
- totalBranches
- totalOrders
- totalRevenue
- platformGrowth
- revenueByMonth

FE gap:

- Current tenant FE dashboards are not fully integrated with reports.

## 12. Realtime Order Module

SignalR hub:

```text
/hubs/orders
```

Methods:

- `JoinOrder(sessionCode, orderId)` -> validates public order detail and joins order group.
- `LeaveOrder(orderId)`.
- `JoinBranch(branchId)`.
- `LeaveBranch(branchId)`.

Events:

- `OrderUpdated` to order group.
- `BranchOrderUpdated` to branch group.

Publishers trigger updates after:

- Place order.
- Checkout.
- Kitchen confirm/mark ready.
- Waiter confirm/mark served.
- Cashier checkout.
- Cancel order.

FE gap:

- No current FE SignalR integration found for cart/orders.

## 13. Functional Acceptance Criteria

- Admin can create tenant and view it by slug.
- Tenant domain sends correct `X-Tenant-Slug`.
- Owner can manage branches under own restaurant only.
- Managed users are limited to exactly one branch.
- Staff can open table session and QR session expires after configured 6 hours.
- Public customer can place order only with menu items from same branch.
- Kitchen can mark only confirmed/cooking items ready.
- Waiter can serve only ready items.
- Cashier cash payment records amount received/change and completes order.
- PayOS cannot be used if branch config incomplete.
- Voucher cannot exceed quantity, date range, min order, or max discount rules.
- Realtime order updates reach both customer order group and branch group.
