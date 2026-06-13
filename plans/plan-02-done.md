 # Implementation Plan: Customer QR Ordering MVP + Docs Alignment

  ## 0. Current State And Goal

  Current confirmed state:

  - Backend already has tenant-aware domain foundation:
      - ITenantUrlBuilder
      - TenantUrlBuilder
      - App:TenantBaseDomain=scannow.site
      - QR URL generation now uses tenant subdomain
      - PayOS return/cancel URL generation now uses tenant subdomain
      - CORS allows wildcard subdomains of scannow.site

  - Tenant frontend scan-now-customer already:
      - extracts tenant slug from hostname in src/lib/tenant.ts
      - sends X-Tenant-Slug through src/services/axiosBasic.tsx
      - has partial owner/manager routes
      - has staff/kitchen placeholder routes
      - /tables/[qrCodeToken]
      - /sessions/[sessionCode]/menu
      - /sessions/[sessionCode]/checkout
      - /payment/return
      - /payment/cancel
      - cashier dashboard/orders placeholder routes and CASHIER auth redirect

  - Backend public QR flow currently has:
      - GET /api/public/tables/{qrCodeToken}
      - POST /api/public/tables/{qrCodeToken}/join
      - POST /api/public/sessions/join
      - GET /api/public/sessions/{sessionCode}/menu
      - POST /api/public/sessions/{sessionCode}/orders
      - POST /api/public/sessions/{sessionCode}/checkout
      - GET /api/public/sessions/{sessionCode}/payment-status
      - POST /api/public/sessions/{sessionCode}/payment-cancel
      - GET /api/public/sessions/{sessionCode}/orders/{orderId}

  - Runtime/domain state:
      - Domains have been configured.
      - FE builds were confirmed successful by the project owner.
      - Backend build has NuGet audit warnings in this environment; ignore those per project owner unless compile errors appear.
      - Still smoke test deployed tenant QR, PayOS return/cancel, CORS preflight, and `business.scannow.site` reserved/parked behavior.

  Goal of this phase:

  A customer can scan a static table QR code on:
  https://{restaurantSlug}.scannow.site/tables/{qrCodeToken}

  Then the app auto joins the active table session, shows menu, allows cart/order/checkout, and handles payment return/cancel on the same tenant domain.

  Out of scope for this phase:

  - Do not implement business.scannow.site product surface.
  - Do not route business.scannow.site into tenant app unless it is parked or reserved in slug parsing.
  - Do not solve the P1 risk where an old customer with a valid sessionCode can revisit an active session.
  - Do not build full kitchen/staff realtime dashboards.
  - Do not build full business multi-restaurant portal.
  - Do not replace localStorage cart with SignalR shared cart yet.

  ## 1. Domain And Product Decisions To Encode

  Use this final domain model everywhere:

  scannow.site                    -> public landing
  scannow.site/admin              -> ScanNow platform admin
  api.scannow.site                -> backend API
  <restaurantSlug>.scannow.site   -> tenant app for one restaurant
  business.scannow.site           -> future portal for large customers managing multiple restaurants

  Tenant domain contains all daily restaurant operations:

  <tenant>.scannow.site/login
  <tenant>.scannow.site/owner/...
  <tenant>.scannow.site/manager/...
  <tenant>.scannow.site/staff/...
  <tenant>.scannow.site/kitchen/...
  <tenant>.scannow.site/cashier/...
  <tenant>.scannow.site/tables/{qrCodeToken}
  <tenant>.scannow.site/sessions/{sessionCode}/menu
  <tenant>.scannow.site/sessions/{sessionCode}/checkout
  <tenant>.scannow.site/payment/return
  <tenant>.scannow.site/payment/cancel

  business.scannow.site future meaning:

  Business portal for large customers who own/manage multiple restaurants.

  business.scannow.site does not replace tenant domain. Daily restaurant operations still happen inside each restaurant subdomain.

  business.scannow.site should not be used as a central app portal in docs or code. It should be parked/future-only or included in reserved-subdomain lists to prevent tenant collision.

  ## 2. Backend Implementation

  ### 2.1 Add Join By Static QR Token API

  Add a new endpoint to PublicTableQrController:

  POST /api/public/tables/{qrCodeToken}/join

  Response envelope remains the existing API shape:

  ApiResponse<JoinSessionResponse>

  Expected response body:

  {
    "result": {
      "sessionId": "guid",
      "tableId": "guid",
      "branchId": "guid",
      "sessionCode": "ABC123",
      "tableNumber": "A1",
      "branchName": "Nguyen Trai Branch",
      "expiresAt": "2026-06-12T12:00:00Z"
    },
    "message": "Join session successfully"
  }

  Add to ITableQrService:

  Task<JoinSessionResponse> JoinSessionByQrTokenAsync(string qrCodeToken);

  Implement in TableQrService.

  Required backend logic:

  1. Normalize qrCodeToken: trim, reject empty.
  2. Find table by qrCodeToken using existing repository method if available.
  3. If table not found, throw NotFoundException("Table not found").
  4. If table.IsActive == false, throw NotFoundException("Table not found").
  5. If table.Branch.IsActive == false, throw NotFoundException("Table not found").
  6. If table.Branch.Restaurant.IsActive == false, throw NotFoundException("Table not found").
  7. Enforce tenant isolation:
     - The existing tenant middleware/filter should already scope by tenant.
     - Still verify manually if table.Branch.Restaurant.Slug is available and TenantContext.Slug is available.
     - If mismatch, throw NotFoundException("Table not found").
  8. Require table.Status == OCCUPIED.
  9. Find active session for this table.
  10. Require session.IsActive == true.
  11. Require session.ExpiresAt > DateTime.UtcNow.
  12. If no valid session, throw BusinessRuleException("Table is not ready for ordering. Please contact staff.").
  13. Return MapJoinSession(session).

  Do not create a session from this endpoint. Public QR scan must never auto-open a table.

  If repository already has GetActiveSessionByTableIdAsync(table.Id), use it. If not, add:

  Task<QrSession?> GetActiveSessionByTableIdAsync(Guid tableId);

  Repository query must include:

  Session
  Session.Table
  Session.Table.Branch
  Session.Table.Branch.Restaurant

  so tenant/active checks and response mapping are reliable.

  ### 2.2 Keep Existing Join By Session Code

  Do not remove:

  POST /api/public/sessions/join

  It remains backward compatible and can be used for manual session-code fallback later.

  ### 2.3 Session Security Rules For MVP

  Enforce these rules in join-by-QR:

  QR token is static.
  Session is dynamic.
  A customer can join only when staff/cashier has opened the table.
  Expired sessions cannot be joined.
  Closed sessions cannot be joined.
  Public endpoint cannot create sessions.

  Known accepted MVP risk:

  If someone already has a valid sessionCode while the session is still active, they can revisit it.

  Do not solve that in this phase. Mark it as P1 hardening in docs.

  ### 2.4 Backend Tests

  Add/extend tests for TableQrService.JoinSessionByQrTokenAsync.

  If the backend currently has no test project, add a minimal test project only if practical. Otherwise create service-level tests in the existing test structure if present.

  Test cases:

  1. Active table + active session:
     Input qrCodeToken.
     Expected JoinSessionResponse with sessionCode.

  2. Table exists but status != OCCUPIED:
     Expected BusinessRuleException with generic not-ready message.

  3. Table occupied but no active session:
     Expected BusinessRuleException with generic not-ready message.

  4. Active session expired:
     Expected BusinessRuleException with generic not-ready message.

  5. Table inactive:
     Expected NotFoundException.

  6. Branch inactive:
     Expected NotFoundException.

  7. Restaurant inactive:
     Expected NotFoundException.

  8. Tenant mismatch:
     Request tenant pho24 with QR from highlands.
     Expected NotFoundException.

  9. Existing POST /api/public/sessions/join still works.

  Run:

  dotnet build ScanNow/ScanNow.Web/ScanNow.Web.csproj --no-restore
  dotnet test

  If build fails only due NuGet audit warnings, report that separately.

  ## 3. Tenant Frontend Implementation

  Repo:

  scan-now-customer

  ### 3.1 Add Route Constants

  Update src/constants/path.ts.

  Add customer paths:

  customer = {
    get table() {
      return (qrCodeToken: string) => `/tables/${qrCodeToken}` as const;
    },
    get menu() {
      return (sessionCode: string) => `/sessions/${sessionCode}/menu` as const;
    },
    get checkout() {
      return (sessionCode: string, orderId?: string) =>
        orderId
          ? `/sessions/${sessionCode}/checkout?orderId=${orderId}` as const
          : `/sessions/${sessionCode}/checkout` as const;
    },
  }

  Add payment paths:

  payment = {
    return: "/payment/return" as const,
    cancel: "/payment/cancel" as const,
  }

  Add cashier paths:

  cashier = {
    dashboard: "/cashier/dashboard" as const,
    orders: "/cashier/orders" as const,
  }

  ### 3.2 Update Role Redirect

  Current implementation:

  CASHIER: PATH.cashier.dashboard exists in src/lib/auth.ts.

  Keep:

  OWNER -> /owner/users
  MANAGER -> /manager/users
  BRANCH_MANAGER -> /manager/users
  STAFF -> /staff/dashboard
  KITCHEN -> /kitchen/dashboard

  Do not route tenant ADMIN users into tenant app unless already required. Platform admin remains scannow.site/admin.

  ### 3.3 Add Minimal Cashier Placeholder

  Already added:

  src/app/cashier/dashboard/page.tsx
  src/app/cashier/orders/page.tsx

  For this phase:

  - Must be protected for CASHIER, BRANCH_MANAGER, OWNER if local patterns support allowed roles.
  - Must not be a blank page.
  - Must show a simple operational placeholder such as:
      - title
      - branch/order operations coming soon
      - logout/navigation support through existing shell if available

  This prevents cashier login from dead-ending.

  ### 3.4 Hide Public Chrome On Customer Flow

  Update src/components/auth/app-chrome.tsx.

  Add prefixes to hide header/footer:

  "/tables"
  "/sessions"
  "/payment"
  "/cashier"
  "/staff"
  "/kitchen"

  Minimum required for customer flow:

  "/tables"
  "/sessions"
  "/payment"

  If cashier/staff/kitchen use their own protected shell, hide public marketing chrome there too.

  ### 3.5 Add Public Ordering Types

  Create:

  src/types/public-ordering.ts

  Types:

  export type PublicTableResponse = {
    tableId: string;
    branchId: string;
    branchName: string;
    tableNumber: string;
    status: "AVAILABLE" | "OCCUPIED" | "DISABLED" | string;
  };

  export type JoinSessionResponse = {
    sessionId: string;
    tableId: string;
    branchId: string;
    sessionCode: string;
    tableNumber: string;
    branchName: string;
    expiresAt: string;
  };

  export type MenuItemResponse = {
    menuItemId: string;
    branchId: string;
    branchName?: string | null;
    categoryId: string;
    categoryName?: string | null;
    name: string;
    description?: string | null;
    imageUrl?: string | null;
    price: number;
    costPrice?: number;
    preparationTime: number;
    displayOrder: number;
    isAvailable: boolean;
    isFeatured: boolean;
    isActive: boolean;
    createdAt?: string;
    updatedAt?: string | null;
  };

  export type MenuCategoryResponse = {
    categoryId: string;
    categoryName: string;
    displayOrder: number;
    items: MenuItemResponse[];
  };

  export type SessionMenuResponse = {
    session: JoinSessionResponse;
    menu: {
      items: MenuCategoryResponse[];
      pageNumber: number;
      pageSize: number;
      totalItems: number;
      totalPages: number;
    };
  };

  export type CartItem = {
    menuItemId: string;
    name: string;
    price: number;
    imageUrl?: string | null;
    quantity: number;
    note?: string;
  };

  export type PlaceOrderRequest = {
    customerName?: string;
    customerPhone?: string;
    customerNote?: string;
    items: Array<{
      menuItemId: string;
      quantity: number;
      note?: string;
    }>;
  };

  export type CustomerOrderResponse = {
    orderId: string;
    orderNumber: string;
    branchId: string;
    tableId?: string | null;
    customerName?: string | null;
    customerPhone?: string | null;
    customerNote?: string | null;
    subTotal: number;
    vatPercent: number;
    vatAmount: number;
    serviceChargePercent: number;
    serviceChargeAmount: number;
    totalAmount: number;
    status: string;
    orderSource: string;
    items: Array<{
      orderItemId: string;
      menuItemId: string;
      menuItemName: string;
      unitPrice: number;
      quantity: number;
      subTotal: number;
      note?: string | null;
      status: string;
      estimatedCookingMinutes: number;
    }>;
    createdAt: string;
    updatedAt?: string | null;
  };

  export type CheckoutResponse = {
    orderId: string;
    paymentId: string;
    paymentMethod: "PAYOS" | "CASH" | string;
    checkoutUrl?: string | null;
    qrCode?: string | null;
    bin?: string | null;
    accountNumber?: string | null;
    accountName?: string | null;
    amount?: number | null;
    description?: string | null;
    paymentExpiresAt?: string | null;
  };

  export type PaymentStatusResponse = {
    orderId: string;
    paymentStatus: "NO_PAYMENT" | "PENDING" | "SUCCESS" | "FAILED" | string;
    orderStatus: string;
  };

  ### 3.6 Add Public Ordering Service

  Create:

  src/services/public-ordering.ts

  Functions:

  getPublicTable(qrCodeToken: string)
  joinTableByQr(qrCodeToken: string)
  getSessionMenu(sessionCode: string, params?: { search?: string; categoryId?: string; pageNumber?: number; pageSize?: number })
  placeOrder(sessionCode: string, payload: PlaceOrderRequest)
  getOrderDetail(sessionCode: string, orderId: string)
  checkoutSession(sessionCode: string, payload: { paymentMethod: "PAYOS" | "CASH" })
  getPaymentStatus(sessionCode: string)
  cancelPendingPayment(sessionCode: string)

  Use axiosBasic.

  Do not manually add tenant header here because axiosBasic already handles it.

  ### 3.7 Add Cart Utility

  Create:

  src/lib/customer-cart.ts

  Functions:

  getCartStorageKey(tenantSlug: string | null, sessionCode: string): string
  readCart(tenantSlug: string | null, sessionCode: string): CartItem[]
  writeCart(tenantSlug: string | null, sessionCode: string, items: CartItem[]): void
  clearCart(tenantSlug: string | null, sessionCode: string): void
  calculateCartTotal(items: CartItem[]): number
  toPlaceOrderItems(items: CartItem[]): PlaceOrderRequest["items"]

  Storage key:

  cart:{tenantSlug || "unknown"}:{sessionCode}

  Rules:

  - If localStorage unavailable, return empty cart.
  - Invalid JSON returns empty cart and clears corrupted key.
  - Quantity minimum is 1.
  - Remove item if quantity becomes 0.

  ### 3.8 Add Customer Page: /tables/[qrCodeToken]

  Create:

  src/app/tables/[qrCodeToken]/page.tsx

  This page should be a client component or render a client child component.

  Behavior:

  1. Read qrCodeToken from params.
  2. Call getPublicTable(qrCodeToken).
  3. Then call joinTableByQr(qrCodeToken).
  4. On success, router.replace(`/sessions/${sessionCode}/menu`).
  5. While loading, show clear loading state.
  6. If table lookup fails, show invalid QR state.
  7. If join fails because table not ready, show not-ready state.

  Displayed states:

  Loading:

  Đang kiểm tra bàn...

  Invalid QR:

  Mã QR không hợp lệ hoặc bàn không còn hoạt động.

  Table not open:

  Bàn chưa sẵn sàng để gọi món. Vui lòng gọi nhân viên.

  If table info is available, show:

  Branch name
  Table number

  CTA:

  Thử lại

  No customer auth required.

  ### 3.9 Add Customer Page: /sessions/[sessionCode]/menu

  Create:

  src/app/sessions/[sessionCode]/menu/page.tsx

  Behavior:

  1. Read sessionCode.
  2. Fetch menu by sessionCode.
  3. Render branch/table context from response.session.
  4. Render categories.
  5. Render menu items.
  6. Allow add/increase/decrease quantity.
  7. Persist cart in localStorage.
  8. Submit order.
  9. On order success, clear cart and redirect to checkout with orderId.

  Minimum UI sections:

  Header:
    Branch name
    Table number

  Category filter:
    All
    Category list

  Menu list:
    Item image if available
    Name
    Description
    Price
    Quantity stepper

  Sticky cart summary:
    Item count
    Total
    Button "Đặt món"

  Submit order:

  POST /api/public/sessions/{sessionCode}/orders

  Payload:

  {
    "items": [
      {
        "menuItemId": "guid",
        "quantity": 2,
        "note": "less spicy"
      }
    ]
  }

  For MVP, customer name/phone can be optional. If a simple customer form is added, it should collect:

  customerName optional
  customerPhone optional
  customerNote optional

  Validation:

  - Cannot submit empty cart.
  - Cannot submit item quantity <= 0.
  - Disable submit while pending.
  - Show API error message if order fails.

  After success:

  router.push(`/sessions/${sessionCode}/checkout?orderId=${order.orderId}`)

  ### 3.10 Add Customer Page: /sessions/[sessionCode]/checkout

  Create:

  src/app/sessions/[sessionCode]/checkout/page.tsx

  Behavior:

  1. Read sessionCode from params.
  2. Read orderId from search params.
  3. If orderId exists, fetch order detail.
  4. Show order summary.
  5. Let customer choose payment method.
  6. Default payment method = PAYOS.
  7. On confirm, call checkout endpoint.

  Payment methods:

  PAYOS -> "Thanh toán online"
  CASH  -> "Thanh toán tại quầy"

  PAYOS behavior:

  1. POST /api/public/sessions/{sessionCode}/checkout with { paymentMethod: "PAYOS" }
  2. If response.checkoutUrl exists, window.location.href = checkoutUrl.
  3. If no checkoutUrl but has qrCode/account info, display fallback payment info.

  CASH behavior:

  1. POST /api/public/sessions/{sessionCode}/checkout with { paymentMethod: "CASH" }
  2. Show "Vui lòng thanh toán tại quầy".
  3. Provide CTA back to menu.

  Important:

  - Do not mark payment success on frontend.
  - Backend/payment status remains source of truth.

  ### 3.11 Add Payment Return Page

  Create:

  src/app/payment/return/page.tsx

  Behavior:

  1. Read sessionCode, orderId, source from query params.
  2. If sessionCode exists, call getPaymentStatus(sessionCode).
  3. Render status from backend.
  4. If sessionCode missing and source=cashier, render generic cashier payment return state.

  Status copy:

  SUCCESS:
    Thanh toán thành công.

  PENDING:
    Thanh toán đang được xác nhận.

  FAILED:
    Thanh toán thất bại.

  NO_PAYMENT:
    Chưa tìm thấy giao dịch thanh toán.

  CTA:

  Back to menu -> /sessions/{sessionCode}/menu
  Retry checkout -> /sessions/{sessionCode}/checkout?orderId={orderId}

  Do not call cancel automatically.

  ### 3.12 Add Payment Cancel Page

  Create:

  src/app/payment/cancel/page.tsx

  Behavior:

  1. Read sessionCode, orderId, source from query params.
  2. Show payment cancelled/not completed state.
  3. If sessionCode exists, optionally fetch payment status.
  4. Provide CTA back to checkout/menu.

  Do not cancel backend pending payment automatically unless user explicitly clicks a button.

  Optional button:

  Hủy giao dịch đang chờ

  If implemented, it calls:

  POST /api/public/sessions/{sessionCode}/payment-cancel

  ### 3.13 UI Constraints

  Use existing UI primitives in src/components/ui.

  Use lucide-react icons where useful.

  Mobile-first layout:

  - Avoid marketing header/footer.
  - Sticky bottom cart summary on mobile.
  - Buttons must not overflow.
  - Text must fit containers.
  - Keep operational UI dense and clear.

  No landing-page hero for customer flow.

  ## 4. Docs Update

  Update docs after backend/FE code is implemented.

  ### 4.1 Clarify business.scannow.site

  In all docs:

  business.scannow.site

  should not be described as the restaurant operations app.

  Replace product concept with:

  business.scannow.site

  Meaning:

  Future business portal for large customers managing multiple restaurants.

  Required remaining mention:

  business is a future/parked subdomain label to prevent tenant collision until the business portal exists.

  Do not describe business.scannow.site as a current product surface.

  ### 4.2 Update QR Auto Join Explanation

  Add to PRD/spec:

  QR is static and identifies a table.
  Session is dynamic and created by staff/cashier when opening the table.
  Customer scans static QR.
  Backend finds the current active session for that table.
  If no active session exists, customer cannot order.

  Add join rules:

  table active
  branch active
  restaurant active
  table status OCCUPIED
  session active
  session not expired
  tenant matches restaurant
  public QR does not create session

  ### 4.3 Add API Spec

  In 04-api-spec.md, add:

  POST /api/public/tables/{qrCodeToken}/join

  Response:

  {
    "sessionId": "guid",
    "tableId": "guid",
    "branchId": "guid",
    "sessionCode": "ABC123",
    "tableNumber": "A1",
    "branchName": "Main Branch",
    "expiresAt": "datetime"
  }

  Error cases:

  404 Table not found
  400/409 Table is not ready for ordering. Please contact staff.

  ### 4.4 Update Roadmap Status

  In 08-gaps-roadmap.md, after implementation:

  - Mark tenant-aware QR URL as done.
  - Mark tenant-aware PayOS redirect as done.
  - Mark customer QR ordering UI as partial/done depending actual implementation.
  - Add P1 hardening item:

  Prevent old customers with sessionCode from rejoining an active session after leaving.

  Possible future approaches:

  short-lived customer participant token
  device-bound participant id
  staff approval for new participant after payment
  session rotation after payment

  Do not implement this now.

  ### 4.5 Add Business Portal Future Section

  Add P2/Future section:

  business.scannow.site

  Purpose:

  multi-restaurant business account
  tenant switcher
  business-level reports
  business-level user management
  billing/subscription

  Future data model, not implemented:

  BusinessOrganization
  BusinessUserMembership
  BusinessRestaurantMembership

  ## 5. Verification Checklist

  Backend commands:

  dotnet build ScanNow/ScanNow.Web/ScanNow.Web.csproj --no-restore
  dotnet test

  Tenant FE commands:

  pnpm lint
  pnpm build

  Manual local test:

  NEXT_PUBLIC_API_URL=http://localhost:{backend-port}
  NEXT_PUBLIC_DEV_TENANT_SLUG=pho24
  pnpm dev

  Manual scenarios:

  1. Open /tables/{invalidToken}
     Expected invalid QR state.

  2. Open /tables/{validToken} before staff opens table.
     Expected not-ready state.

  3. Staff opens table.
     Expected backend creates active QrSession.

  4. Open /tables/{validToken}.
     Expected auto redirect to /sessions/{sessionCode}/menu.

  5. Add menu items to cart.
     Refresh page.
     Expected cart persists.

  6. Place order.
     Expected order created and redirect to checkout.

  7. Checkout PAYOS.
     Expected browser redirects to PayOS checkoutUrl.

  8. Simulate return:
     /payment/return?sessionCode={sessionCode}&orderId={orderId}

  9. Simulate cancel:
     /payment/cancel?sessionCode={sessionCode}&orderId={orderId}
     Expected cancel UI with retry/back CTA.

  10. Login CASHIER.
      Expected redirect to /cashier/dashboard, not /.

  11. Tenant mismatch:
      Use QR from tenant A under tenant B.
      Expected not found/not ready, no data leak.

  ## 6. Acceptance Criteria

  Implementation is complete when:

  - Backend has POST /api/public/tables/{qrCodeToken}/join.
  - Static QR can auto join an active table session.
  - Public QR cannot create session.
  - Expired/closed/not-open sessions cannot be joined by QR.
  - scan-now-customer has all customer routes:
      - /tables/[qrCodeToken]
      - /sessions/[sessionCode]/menu
      - /sessions/[sessionCode]/checkout
      - /payment/return
      - /payment/cancel

  - Customer can:
      - scan QR
      - auto join
      - view menu
      - add cart
      - place order
      - checkout
      - return/cancel payment

  - CASHIER role redirects to /cashier/dashboard.
  - Docs explain:
      - static QR + dynamic session
      - join rules
      - accepted old-sessionCode risk
      - business.scannow.site future role
      - no business.scannow.site product surface
