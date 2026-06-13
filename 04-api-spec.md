# API Specification

Base URL khuyen nghi production:

```text
https://api.scannow.site
```

Current FE env da duoc chuyen sang:

```text
https://api.scannow.site
```

Response wrapper:

```json
{
  "result": {},
  "code": 0,
  "message": "string"
}
```

Paged result:

```json
{
  "items": [],
  "pageNumber": 1,
  "pageSize": 10,
  "totalItems": 0,
  "totalPages": 0
}
```

## 1. Tenant Header

Tenant FE must send:

```http
X-Tenant-Slug: {restaurantSlug}
```

Examples:

```http
X-Tenant-Slug: pho24
```

Backend also supports host subdomain extraction, but in recommended architecture API host is `api.scannow.site`, so header is required.

Reserved subdomains:

```text
www, api, admin, app, localhost, staging
```

## 2. Authentication

### 2.1 Login

```http
POST /api/auth/login
```

Body:

```json
{
  "identifier": "admin@example.com or username",
  "password": "string"
}
```

Response:

```json
{
  "user": {
    "id": "guid",
    "email": "string",
    "username": "string",
    "fullName": "string",
    "avatarUrl": "string | null",
    "role": "ADMIN | OWNER | BRANCH_MANAGER | STAFF | KITCHEN | CASHIER",
    "isEmailVerified": true,
    "isActive": true,
    "createdAt": "datetime",
    "updatedAt": "datetime | null"
  },
  "accessToken": "jwt",
  "refreshToken": "string | null"
}
```

Notes:

- FE stores access token in localStorage.
- Refresh token flow uses cookie/credentials.

### 2.2 Other auth endpoints

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/auth/register` | Public | Register user |
| POST | `/api/auth/login-google` | Public | Login with Google ID token |
| POST | `/api/auth/refresh-token` | Cookie | Refresh access token |
| POST | `/api/auth/logout` | Cookie/token | Logout |
| POST | `/api/auth/check-email-exist` | Public | Check email existence |
| POST | `/api/auth/check-username-exist` | Public | Check username existence |
| POST | `/api/auth/change-password` | Authenticated | Change password |
| POST | `/api/auth/resend-email-verification` | Public | Resend verification email |
| POST | `/api/auth/verify-email` | Public | Verify email |

## 3. Platform Admin APIs

Role: `ADMIN`

### 3.1 Owner management

| Method | Path | Query/body | Response |
|---|---|---|---|
| GET | `/api/admin/users/owners` | `PageNumber`, `PageSize`, `Search`, `IsActive`, `IsBanned`, `SortBy`, `SortDirection` | `PagedResult<OwnerUserResponse>` |
| GET | `/api/admin/users/owners/available` | paging | available owner list |
| POST | `/api/admin/users/owners` | `CreateOwnerRequest` | `OwnerUserResponse` |
| PUT | `/api/admin/users/owners/{id}` | `UpdateOwnerRequest` | `OwnerUserResponse` |
| PATCH | `/api/admin/users/owners/{id}/ban` | none/body optional not used by controller | null |
| PATCH | `/api/admin/users/owners/{id}/unban` | none | null |

Create owner:

```json
{
  "fullName": "Nguyen Van A",
  "username": "owner_a",
  "email": "owner@example.com",
  "phoneNumber": "0900000000",
  "password": "string"
}
```

### 3.2 Restaurant/Tenant management

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/admin/restaurants` | List restaurants |
| GET | `/api/admin/restaurants/{id}` | Get by GUID |
| GET | `/api/admin/restaurants/by-slug/{slug}` | Get by restaurant slug |
| GET | `/api/admin/restaurants/{id}/branches` | List branches by restaurant GUID |
| GET | `/api/admin/restaurants/by-slug/{slug}/branches` | List branches by restaurant slug |
| GET | `/api/admin/restaurants/{id}/branches/{branchId}` | Get branch by GUIDs |
| GET | `/api/admin/restaurants/by-slug/{slug}/branches/{branchSlug}` | Get branch by slugs |
| POST | `/api/admin/restaurants` | Create restaurant |
| PUT | `/api/admin/restaurants/{id}` | Update restaurant |
| PATCH | `/api/admin/restaurants/{id}/ban` | Inactivate restaurant |
| PATCH | `/api/admin/restaurants/{id}/unban` | Reactivate restaurant |

Create restaurant:

```json
{
  "ownerId": "guid",
  "name": "Pho 24",
  "slug": "pho24",
  "logoUrl": "https://...",
  "description": "string"
}
```

Restaurant response:

```json
{
  "restaurantId": "guid",
  "ownerId": "guid",
  "ownerName": "string",
  "ownerEmail": "string",
  "ownerPhone": "string | null",
  "name": "string",
  "slug": "string",
  "logoUrl": "string | null",
  "description": "string | null",
  "totalBranches": 0,
  "isActive": true,
  "createdAt": "datetime",
  "updatedAt": "datetime | null"
}
```

### 3.3 Admin read-only branch/menu/table supervision

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/admin/branches/{branchId}/categories` | List categories |
| GET | `/api/admin/branches/{branchId}/categories/{id}` | Category detail |
| GET | `/api/admin/branches/{branchId}/menu-items` | List menu items |
| GET | `/api/admin/menu-items/{id}` | Menu item detail |
| GET | `/api/admin/menu-items/{id}/price-history` | Price history |
| GET | `/api/admin/branches/{branchId}/tables` | List tables |
| GET | `/api/admin/branches/{branchId}/tables/{id}` | Table detail |
| GET | `/api/admin/branches/{branchId}/sessions` | Active sessions |

## 4. Owner APIs

Role: `OWNER`

### 4.1 Own restaurant

```http
GET /api/owner/restaurant/me
PUT /api/owner/restaurant/me
```

Update body:

```json
{
  "name": "string",
  "slug": "string",
  "logoUrl": "string | null",
  "description": "string | null"
}
```

### 4.2 Branches

| Method | Path | Body/query |
|---|---|---|
| POST | `/api/owner/branches` | `CreateBranchRequest` |
| GET | `/api/owner/branches` | `PageNumber`, `PageSize`, `Search`, `IsActive`, `SortBy`, `SortDirection` |
| GET | `/api/owner/branches/{id}` | none |
| PUT | `/api/owner/branches/{id}` | `UpdateBranchRequest` |
| PATCH | `/api/owner/branches/{id}/inactive` | none |
| PATCH | `/api/owner/branches/{id}/active` | none |

Create/update branch:

```json
{
  "name": "District 1",
  "slug": "district-1",
  "address": "string",
  "phone": "string",
  "email": "string",
  "openTime": "08:00:00",
  "closeTime": "22:00:00",
  "vatPercent": 8,
  "serviceChargePercent": 5,
  "serviceChargeFixed": 0
}
```

### 4.3 Owner user management

```http
GET /api/owner/users
POST /api/owner/users
PUT /api/owner/users/{id}
PATCH /api/owner/users/{id}/ban
PATCH /api/owner/users/{id}/unban
```

Create managed user:

```json
{
  "fullName": "string",
  "username": "string",
  "email": "string",
  "phoneNumber": "string",
  "password": "string",
  "role": "BRANCH_MANAGER | STAFF | KITCHEN | CASHIER",
  "branchIds": ["guid"]
}
```

Validation:

- `branchIds` must contain exactly one non-empty GUID.

## 5. Manager and Me APIs

### 5.1 Manager user management

Role: `BRANCH_MANAGER`

```http
GET /api/manager/users
POST /api/manager/users
PUT /api/manager/users/{id}
PATCH /api/manager/users/{id}/ban
PATCH /api/manager/users/{id}/unban
```

### 5.2 My branches

Roles: `BRANCH_MANAGER`, `STAFF`, `KITCHEN`, `CASHIER`

```http
GET /api/me/branches
GET /api/me/branches/{id}
```

### 5.3 My menu availability

Roles: `STAFF`, `CASHIER`, `KITCHEN`

```http
GET /api/me/branches/{branchId}/menu
GET /api/me/menu-items/{id}
PATCH /api/me/menu-items/{id}/toggle-available
PATCH /api/me/branches/{branchId}/menu-items/bulk-availability
```

## 6. Menu APIs

Roles: `OWNER`, `BRANCH_MANAGER`

### 6.1 Categories

```http
GET /api/owner/branches/{branchId}/categories
GET /api/owner/branches/{branchId}/categories/{id}
POST /api/owner/branches/{branchId}/categories
PUT /api/owner/branches/{branchId}/categories/{id}
PATCH /api/owner/branches/{branchId}/categories/reorder
PATCH /api/owner/branches/{branchId}/categories/{id}/active
PATCH /api/owner/branches/{branchId}/categories/{id}/inactive
```

Create category:

```json
{
  "name": "Noodles",
  "description": "string",
  "imageUrl": "string",
  "displayOrder": 1
}
```

### 6.2 Menu items

```http
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

Create menu item:

```json
{
  "name": "Pho bo",
  "description": "string",
  "imageUrl": "string",
  "price": 65000,
  "costPrice": 30000,
  "preparationTime": 10,
  "displayOrder": 1,
  "isAvailable": true,
  "isFeatured": false
}
```

Update menu item:

```json
{
  "categoryId": "guid",
  "name": "Pho bo dac biet",
  "description": "string",
  "imageUrl": "string",
  "costPrice": 32000,
  "preparationTime": 12,
  "displayOrder": 1,
  "isAvailable": true,
  "isFeatured": true
}
```

Update price:

```json
{
  "price": 70000,
  "note": "Updated menu price"
}
```

## 7. Table & QR APIs

Roles: `OWNER`, `BRANCH_MANAGER`

```http
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

Create/update table:

```json
{
  "tableNumber": "A1",
  "capacity": 4
}
```

Update status:

```json
{
  "status": "AVAILABLE | RESERVED | DISABLED"
}
```

Note:

- `OCCUPIED` is managed by session flow, not manual status update.
- Backend serializes enum as string.

### 7.1 Staff table session APIs

Roles: `STAFF`, `CASHIER`, `KITCHEN`

```http
POST /api/me/branches/{branchId}/tables/{tableId}/open
PATCH /api/me/sessions/{sessionId}/close
GET /api/me/branches/{branchId}/tables
GET /api/me/tables/{id}
GET /api/me/tables/{id}/orders
```

Open session response:

```json
{
  "sessionId": "guid",
  "tableId": "guid",
  "branchId": "guid",
  "sessionCode": "ABC123",
  "isActive": true,
  "expiresAt": "datetime",
  "createdAt": "datetime",
  "updatedAt": "datetime | null"
}
```

## 8. Public Customer APIs

Public, but tenant header should be sent from tenant FE.

### 8.1 Get table by QR token

```http
GET /api/public/tables/{qrCodeToken}
```

Response:

```json
{
  "tableId": "guid",
  "branchId": "guid",
  "branchName": "District 1",
  "tableNumber": "A1",
  "status": "AVAILABLE | OCCUPIED | RESERVED | DISABLED"
}
```

### 8.2 Join session by QR token

```http
POST /api/public/tables/{qrCodeToken}/join
```

Response:

```json
{
  "sessionId": "guid",
  "tableId": "guid",
  "branchId": "guid",
  "sessionCode": "ABC123",
  "tableNumber": "A1",
  "branchName": "District 1",
  "expiresAt": "datetime"
}
```

### 8.3 Join session

```http
POST /api/public/sessions/join
```

Body:

```json
{
  "sessionCode": "ABC123"
}
```

Response:

```json
{
  "sessionId": "guid",
  "tableId": "guid",
  "branchId": "guid",
  "tableNumber": "A1",
  "branchName": "District 1",
  "expiresAt": "datetime"
}
```

### 8.3 Get session menu

```http
GET /api/public/sessions/{sessionCode}/menu
```

Query:

- `PageNumber`
- `PageSize`
- `Search`
- `IsFeatured`
- `CategoryId`
- `SortBy`: `name`, `price`, `createdAt`, default displayOrder
- `SortDirection`: `asc`, `desc`

Response:

```json
{
  "session": {},
  "menu": {
    "items": [
      {
        "categoryId": "guid",
        "categoryName": "string",
        "displayOrder": 1,
        "items": [
          {
            "menuItemId": "guid",
            "branchId": "guid",
            "categoryId": "guid",
            "categoryName": "string",
            "name": "string",
            "description": "string",
            "imageUrl": "string",
            "price": 65000,
            "costPrice": 30000,
            "preparationTime": 10,
            "displayOrder": 1,
            "isAvailable": true,
            "isFeatured": false,
            "isActive": true
          }
        ]
      }
    ],
    "pageNumber": 1,
    "pageSize": 10,
    "totalItems": 1,
    "totalPages": 1
  }
}
```

### 8.4 Place order

```http
POST /api/public/sessions/{sessionCode}/orders
```

Body:

```json
{
  "customerName": "string",
  "customerPhone": "string",
  "customerNote": "Less spicy",
  "items": [
    {
      "menuItemId": "guid",
      "quantity": 2,
      "note": "No onion"
    }
  ]
}
```

Response: `CustomerOrderResponse`.

Important:

- Items must be active, available, and belong to same branch as session.
- If active order exists on session, new items are appended.
- Existing pending payments are marked failed when appending items.

### 8.5 Checkout

```http
POST /api/public/sessions/{sessionCode}/checkout
```

Body:

```json
{
  "paymentMethod": "PAYOS | CASH"
}
```

Response:

```json
{
  "orderId": "guid",
  "paymentId": "guid",
  "paymentMethod": "PAYOS",
  "checkoutUrl": "string | null",
  "qrCode": "string | null",
  "bin": "string | null",
  "accountNumber": "string | null",
  "accountName": "string | null",
  "amount": 100000,
  "description": "SN 123456",
  "paymentExpiresAt": "datetime | null"
}
```

### 8.6 Payment status/cancel

```http
GET /api/public/sessions/{sessionCode}/payment-status
POST /api/public/sessions/{sessionCode}/payment-cancel
```

Response:

```json
{
  "orderId": "guid",
  "paymentStatus": "NO_PAYMENT | PENDING | SUCCESS | FAILED",
  "orderStatus": "PendingConfirmation | Completed | ..."
}
```

### 8.7 Public order detail

```http
GET /api/public/sessions/{sessionCode}/orders/{orderId}
```

Requires order to be active order of that session.

## 9. Waiter APIs

Roles: `STAFF`, `CASHIER`, `BRANCH_MANAGER`

Branch is passed as query parameter in controller methods.

```http
GET /api/waiter/orders/pending-confirmation?branchId={branchId}
POST /api/waiter/orders/{orderId}/confirm?branchId={branchId}
GET /api/waiter/items/ready-to-serve?branchId={branchId}
POST /api/waiter/items/mark-served?branchId={branchId}
POST /api/waiter/orders?branchId={branchId}
```

Create waiter order:

```json
{
  "tableId": "guid",
  "customerName": "string",
  "customerNote": "string",
  "items": [
    {
      "menuItemId": "guid",
      "quantity": 1,
      "note": "string"
    }
  ]
}
```

Mark served:

```json
{
  "orderItemIds": ["guid"]
}
```

## 10. Kitchen APIs

Roles: `KITCHEN`, `BRANCH_MANAGER`

```http
GET /api/kitchen/orders/pending-confirmation?branchId={branchId}
POST /api/kitchen/orders/{orderId}/confirm?branchId={branchId}
POST /api/kitchen/items/confirm?branchId={branchId}
GET /api/kitchen/items/grouped?branchId={branchId}&status={status}
POST /api/kitchen/items/mark-ready?branchId={branchId}
```

Confirm items:

```json
{
  "orderItemIds": ["guid"]
}
```

Mark ready:

```json
{
  "orderItemIds": ["guid"]
}
```

## 11. Cashier APIs

Roles: `CASHIER`, `STAFF`, `BRANCH_MANAGER`, `OWNER` for read; checkout/cancel restricted to `CASHIER`, `BRANCH_MANAGER`, `OWNER`.

```http
GET /api/cashier/branches/{branchId}/orders
GET /api/cashier/branches/{branchId}/orders/{orderId}
POST /api/cashier/branches/{branchId}/orders/{orderId}/checkout
POST /api/cashier/branches/{branchId}/orders/{orderId}/payment-cancel
```

Order query:

- `PageNumber`
- `PageSize`
- `Search`
- `Status`: `active`, `paid`, `all`
- `SortBy`: `createdAt`, `tableNumber`, `totalAmount`, `status`
- `SortDirection`: `asc`, `desc`

Checkout body:

```json
{
  "paymentMethod": "CASH | PAYOS",
  "voucherCode": "VOUCHER10",
  "amountReceived": 200000
}
```

Cashier payment response includes:

- orderId
- paymentId
- paymentMethod
- paymentStatus
- orderStatus
- PayOS checkout fields
- amountReceived
- changeAmount
- order detail

## 12. Branch Settings APIs

Roles: `OWNER`, `BRANCH_MANAGER`

### 12.1 Payment config

```http
GET /api/owner/branches/{branchId}/payment-config
PUT /api/owner/branches/{branchId}/payment-config
GET /api/manager/branches/{branchId}/payment-config
PUT /api/manager/branches/{branchId}/payment-config
```

Upsert body:

```json
{
  "cashEnabled": true,
  "payOsEnabled": true,
  "payOsClientId": "string",
  "payOsApiKey": "string",
  "payOsChecksumKey": "string",
  "defaultMethod": "CASH | PAYOS"
}
```

Rules:

- Cash must remain enabled.
- PayOS enabled requires all credentials.
- Blank credential fields preserve existing credential values.

### 12.2 Paper vouchers

```http
GET /api/owner/branches/{branchId}/paper-vouchers
POST /api/owner/branches/{branchId}/paper-vouchers
PUT /api/owner/branches/{branchId}/paper-vouchers/{voucherId}
GET /api/manager/branches/{branchId}/paper-vouchers
POST /api/manager/branches/{branchId}/paper-vouchers
PUT /api/manager/branches/{branchId}/paper-vouchers/{voucherId}
```

Create/update body:

```json
{
  "code": "WELCOME10",
  "name": "Welcome 10%",
  "description": "string",
  "discountType": "PERCENT | FIXED_AMOUNT",
  "discountValue": 10,
  "minOrderAmount": 100000,
  "maxDiscountAmount": 50000,
  "quantity": 100,
  "validFrom": "datetime | null",
  "validUntil": "datetime | null",
  "isActive": true
}
```

## 13. Reports APIs

| Method | Path | Role |
|---|---|---|
| GET | `/api/owner/reports/overview` | OWNER |
| GET | `/api/manager/reports/overview` | BRANCH_MANAGER |
| GET | `/api/admin/reports/dashboard` | ADMIN |

Query:

- `BranchId`
- `FromDate`
- `ToDate`

## 14. Realtime Hubs

### 14.1 Cart hub

Endpoint:

```text
/hubs/cart
```

Methods:

```text
JoinSession(sessionCode) -> CartDto
UpdateCart(sessionCode, cart)
ClearCart(sessionCode)
LeaveSession(sessionCode)
```

Events:

```text
CartUpdated(cart)
CartCleared()
```

### 14.2 Order hub

Endpoint:

```text
/hubs/orders
```

Methods:

```text
JoinOrder(sessionCode, orderId) -> CustomerOrderResponse
LeaveOrder(orderId)
JoinBranch(branchId)
LeaveBranch(branchId)
```

Events:

```text
OrderUpdated(order)
BranchOrderUpdated(order)
```

## 15. API Gaps / Contract Risks

- Public QR flow now supports QR-token auto join through `POST /api/public/tables/{qrCodeToken}/join`; runtime smoke test is still required with real tenant domain and active table session.
- Public cash checkout remains an accounting/product risk if it can mark an order paid without cashier confirmation.
- QR URL and PayOS return/cancel URLs are tenant-aware through `ITenantUrlBuilder`; old persisted/printed QR records may need regeneration or backfill.
- Backend supports `CASHIER`; tenant FE has auth redirect and placeholder cashier pages, but full cashier order list/detail/checkout UI is still incomplete.
- Backend serializes enums as strings; admin FE table status type currently numeric.
- Tenant isolation should be tested for all direct child entity queries.
- No OpenAPI schema appears configured for production exposure; development maps OpenAPI/Scalar only.
