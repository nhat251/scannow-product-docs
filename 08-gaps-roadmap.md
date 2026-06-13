# Gaps, Risks & Roadmap

## 1. Executive Summary

Code hien tai da tien toi kien truc tenant mong muon:

- Tenant = restaurant.
- Restaurant slug co the dung lam subdomain.
- Backend resolve tenant tu header/subdomain.
- Admin slug APIs da co.
- CORS wildcard subdomain da co co che cau hinh.
- QR URL va PayOS redirect da tenant-aware thong qua `ITenantUrlBuilder`.
- Tenant FE da co public QR/order/payment routes co ban.
- Domain production da duoc setup; can smoke test runtime voi data/domain that.

Vi con mot so thieu sot trong frontend va backend duoc tong hop qua bang trang thai duoi day.

### Bang Trang Thai Code Hien Tai

| Module / Route | Vi tri / Repository | Trang thai | Chi tiet |
|---|---|---|---|
| Landing (`scannow.site`) | `scan-now-nextjs` | Supported | Renders `SalesLanding` voi da ngon ngu vi/en. |
| Platform Admin (`scannow.site/admin`) | `scan-now-nextjs` | Supported | Login + CRUD restaurant, owner, supervision branches. |
| Owner Portal (`/owner/*`) | `scan-now-customer` | Partial | Co mot phan UI (Restaurant detail, branch/user list & CRUD). Thieu menu management, tables, payment config, vouchers, reports. |
| Manager Portal (`/manager/*`) | `scan-now-customer` | Partial | Co mot phan UI (User management, branch access). Thieu live operations dashboard, reports. |
| Staff Dashboard (`/staff/dashboard`) | `scan-now-customer` | Partial | Chi co placeholder, thieu API integration cho table open/close va waiter live view. |
| Kitchen Dashboard (`/kitchen/dashboard`) | `scan-now-customer` | Partial | Chi co placeholder, thieu API integration cho kitchen grouped queue. |
| Cashier Portal (`/cashier/*`) | `scan-now-customer` | Partial | Dashboard & Orders placeholders and auth routing added. |
| Customer QR Ordering | `scan-now-customer` | Supported | Table QR redirection, menu page, cart storage, checkout/payment result pages implemented. |
| Tenant-aware QR URL | Backend `ScanNow` | Supported | `ITenantUrlBuilder` sinh `https://{restaurantSlug}.scannow.site/tables/{qrCodeToken}`. |
| Tenant-aware Redirect | Backend `ScanNow` | Supported | PayOS return/cancel sinh ve dung tenant domain. |

### Gaps Hien Tai Theo Role & Module

1. **Owner/Manager**: Da co mot phan UI ve mat thong tin co ban va user management. Thieu toan bo phan menu/table management va reports.
2. **Staff/Kitchen**: Dang o dang placeholder tinh, chua ket noi thuc te voi backend va SignalR.
3. **Cashier**: Da co dashboard/orders placeholder va role redirect, nhung chua co order list/detail/checkout UI day du.
4. **Customer QR Ordering**: Da co route co ban cho QR/menu/cart/order/checkout/payment result; can smoke test domain + PayOS + data that.

---

## 2. P0 Blockers Before Production

### P0.1 Public QR ordering runtime smoke test pending

Current:

- Backend public APIs exist.
- Tenant FE has `/tables/[qrCodeToken]`, `/sessions/[sessionCode]/menu`, `/sessions/[sessionCode]/checkout`, `/payment/return`, `/payment/cancel`.
- Backend has `POST /api/public/tables/{qrCodeToken}/join`.

Impact:

- Code path exists, but must still be verified with deployed domain, real tenant data, active table session, and PayOS config.

Required:

- Smoke test `https://{tenant}.scannow.site/tables/{qrCodeToken}` end-to-end.
- Verify `X-Tenant-Slug` arrives at `api.scannow.site`.
- Verify PayOS return/cancel returns to tenant domain.
- Verify invalid/not-open/expired sessions show correct customer state.

### P0.2 QR URL tenant-aware implemented

Current:

- Backend uses `ITenantUrlBuilder` + `App:TenantBaseDomain=scannow.site`.
- It generates `https://{restaurantSlug}.scannow.site/tables/{token}`.

Impact:

- Existing printed QR codes created before this change may still point to old/static URLs.

Required:

- Regenerate/backfill existing QR URLs before printing/using old tables.

### P0.3 Payment redirect tenant-aware implemented

Current:

- CheckoutService and CashierService build PayOS return/cancel URL from restaurant slug via `ITenantUrlBuilder`.

Impact:

- Must still verify branch PayOS credentials and gateway callback/status behavior in deployment.

Required:

- Smoke test PayOS return/cancel on tenant domain.
- Confirm deployment env does not override source config with old domain values.

### P0.4 Cashier UI missing

Current:

- Backend supports `CASHIER` role and APIs.
- Tenant FE has cashier dashboard/orders placeholders.
- Role redirect includes `CASHIER`.

Impact:

- Cash payment/voucher operations cannot be used in UI.
- Public cash checkout has weaker accounting than cashier checkout.

Required:

- Add order list/detail/checkout/cancel payment UI.
- Add branch access and filters.

### P0.5 Production domain deployment verification pending

Current:

- Source config has been moved to `scannow.site` / `api.scannow.site`.
- Backend config includes `App:ProductionDomain=scannow.site`.
- FE `SITE_CONFIG.baseUrl` and API env have been moved to the target production domains.
- Runtime domains have been setup; verify provider env vars still match source config after deploy.
- `business.scannow.site` is future-only. If wildcard DNS routes it to the tenant stack, it must be parked or added to reserved subdomain parsing before production use.

Impact:

- If deployment env still points to old domains, CORS wildcard tenant may not work.
- If deployment env still points to old domains, SEO/sitemap can be wrong.
- Existing old QR records may still need regeneration/backfill.
- If `business.scannow.site` reaches tenant FE/API without being reserved, `business` can be treated as a restaurant slug.

Required:

- Set and verify backend env for production.
- Set and verify FE base URL/API URL in deployment provider.
- Verify CORS preflight with `X-Tenant-Slug`.
- Verify `business.scannow.site` returns a parked/future page or is ignored as reserved subdomain.

### P0.6 Secrets are present in repo config

Current:

- Sensitive-looking database/API/email values are present in local repo config files.

Impact:

- Treat as compromised.
- Production security risk.

Required:

- Rotate exposed secrets.
- Remove from tracked files.
- Use secret manager/env vars.
- Commit only examples/placeholders.

### P0.7 Cross-tenant tests missing

Current:

- Tenant filter exists on `Branch`.
- Direct queries on child entities may need explicit tests.

Impact:

- Risk of Tenant A accessing Tenant B data via GUID/token/sessionCode.

Required:

- Integration tests for branch/menu/table/session/order/payment/voucher APIs.
- SignalR branch join authorization review.

## 3. P1 High Priority

### P1.1 Backend service charge fixed not applied

Current:

- `Branch.ServiceChargeFixed` exists.
- Order calculations use `ServiceChargePercent` only.

Impact:

- Branch settings UI may promise a fixed service fee that totals ignore.

Options:

- Apply fixed charge in all order calculations.
- Or remove/hide fixed charge until implemented.

### P1.2 Public cash checkout behavior

Current:

- Public cash checkout marks order completed but creates payment `PENDING`.
- Cashier cash checkout correctly creates payment `SUCCESS`.

Impact:

- Revenue/reporting may be inconsistent.
- Completed order can have unpaid payment.

Recommended:

- Rename public `CASH` option to "Pay at cashier".
- Do not mark order completed until cashier confirms cash.
- Or add cash confirmation endpoint.

### P1.3 FE role mismatch

Current:

- Backend role: `BRANCH_MANAGER`.
- FE also has `MANAGER` in auth redirect map.
- Backend role: `CASHIER`.
- Tenant FE user-management type/options do not fully include `CASHIER`.

Impact:

- Login redirects and user creation can fail/omit roles.

Required:

- Normalize frontend roles to backend enum.
- Add compatibility mapping only where needed.

### P1.4 Enum serialization mismatch

Current:

- Backend adds `JsonStringEnumConverter`.
- Admin FE `TableStatus` type is numeric `0 | 1 | 2 | 3`.

Impact:

- Table status display/filter may break.

Required:

- Update FE enum types to string:

```text
AVAILABLE | OCCUPIED | RESERVED | DISABLED
```

### P1.5 Owner/manager operational UI incomplete

Current:

- Owner FE covers restaurant/branches/users.
- Backend supports menu/table/payment/voucher/reports.

Required:

- Add menu/category management.
- Add table/QR management.
- Add branch payment config.
- Add paper voucher management.
- Add reports.

### P1.6 Kitchen/staff dashboards incomplete

Current:

- Backend has kitchen/waiter APIs.
- FE dashboards are not fully wired.

Required:

- Staff table open/close UI.
- Waiter pending/ready-to-serve UI.
- Kitchen grouped items UI.
- SignalR branch subscription.

### P1.7 SignalR scaling

Current:

- Cart hub uses memory cache.
- No backplane.

Impact:

- Multi-instance deploy breaks shared cart/group state.

Required for scale:

- Redis/distributed cache.
- SignalR backplane or managed SignalR.

## 4. P2 Roadmap

### P2.1 Business portal

Only add `business.scannow.site` if needed for:

- Business account managing multiple restaurants.
- Tenant switcher.
- Business-level users/permissions.
- Business-level reports and billing.

Do not add for MVP if tenant subdomain covers restaurant users and daily operations.

### P2.2 Tenant branding

Add:

- Restaurant logo/theme.
- Branch selection UX.
- Custom menu layout.
- Tenant-specific metadata.

### P2.3 Customer improvements

Add:

- Customer phone OTP optional.
- Order history by session.
- Rating/review.
- Favorite items.
- Multi-language menu.

### P2.4 Reports and analytics

Add:

- Export CSV/PDF.
- Revenue by payment method.
- Peak day/hour.
- Kitchen prep time.
- Staff performance.
- Voucher effectiveness.

### P2.5 Payments

Add:

- PayOS webhook.
- Refund flow.
- More payment gateways only when validators/services/UI support them.
- Payment reconciliation report.

## 5. Recommended Implementation Plan

### Phase 1: Domain readiness

1. Configure DNS and deployment targets. **Done**
2. Update FE base URLs and API URLs. **Done**
3. Set backend `App:ProductionDomain=scannow.site`. **Done**
4. Add dynamic tenant QR URL generation. **Done**
5. Add dynamic tenant payment redirect. **Done**
6. Rotate/remove secrets. **Pending/ops**

Deliverable:

- Admin and tenant portal can run on target domains.
- QR URL and payment redirect point to tenant domain.

### Phase 2: Tenant customer MVP

1. Build `/tables/[qrCodeToken]`. **Done**
2. Build QR-token auto join/menu/cart. **Done**
3. Build place order. **Done**
4. Build checkout. **Done**
5. Build payment return/cancel. **Done**
6. Integrate `/hubs/cart` and `/hubs/orders`. **Future hardening**
7. Add richer order tracking page. **Future hardening**

Deliverable:

- Customer can scan QR and place order end-to-end.

### Phase 3: Restaurant operations MVP

1. Add cashier routes and UI.
2. Add kitchen grouped queue.
3. Add waiter ready-to-serve and table open/close.
4. Add branch payment config UI.
5. Add voucher UI.

Deliverable:

- Staff/kitchen/cashier can run a service shift.

### Phase 4: Security and scale

1. Add cross-tenant integration tests.
2. Add hub authorization/hardening.
3. Add Redis/distributed cache if multi-instance.
4. Add audit logging.
5. Add health checks and monitoring.

Deliverable:

- Production readiness baseline.

### Phase 5: Reports and polish

1. Owner/manager/admin reports UI.
2. Better error states.
3. Tenant branding.
4. SEO/privacy/legal pages.
5. Customer UX polish.

Deliverable:

- Commercially presentable SaaS.

## 6. Domain Decision Record

Decision:

```text
Use scannow.site for landing/admin.
Use *.scannow.site for restaurant tenants.
Reserve business.scannow.site for future business portal.
```

Rationale:

- Backend tenant model already maps subdomain to restaurant.
- Owner/manager/staff/kitchen/cashier can all be scoped to tenant.
- Customer QR flow must be tenant-scoped.
- Adding `business.scannow.site` now would add business-account data/auth complexity without solving the current single-restaurant tenant workflow.

Revisit decision when:

- One user can belong to multiple restaurants.
- A business user manages multiple restaurant tenants.
- A centralized portal becomes a product requirement.

## 7. Final Support Answer

Question: Da ho tro kien truc nay chua?

```text
1. scannow.site + scannow.site/admin
2. tenant.scannow.site
3. business.scannow.site reserved for future business portal
```

Answer:

- `scannow.site` + `/admin`: **Supported**, can still harden admin role guard and smoke test production.
- `tenant.scannow.site`: **Supported for domain/customer QR MVP**, with remaining gaps in full cashier/staff/kitchen operations and production smoke testing.
- `business.scannow.site`: **Future**, chi can khi business account quan ly nhieu restaurant.
- Tenant = 1 nha hang, nha hang co nhieu chi nhanh: **Dung voi data model va backend hien tai**.
