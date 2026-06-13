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
| Owner Portal (`/owner/*`) | `scan-now-customer` | Supported | Restaurant, branch/user CRUD, category + menu-item management, table/QR management, branch orders, branch settings (VAT/service charge, PayOS payment-config, paper voucher), reports overview dashboard. |
| Manager Portal (`/manager/*`) | `scan-now-customer` | Supported | Same feature set as owner, scoped to the managed branch (users, menu, table, orders, settings, reports). |
| Staff / Kitchen / Branch-Manager ops (`/me/*`) | `scan-now-customer` | Supported | `me/` waiter shell: open/close sessions, branch menu availability, waiter confirm/serve, kitchen grouped queue. Wired to `/api/me`, `/api/waiter`, `/api/kitchen` + `/hubs/orders`. |
| Cashier Portal (`/cashier/*`) | `scan-now-customer` | Supported | Order list/detail, cash/PayOS checkout, voucher, cancel pending payment. |
| Customer QR Ordering | `scan-now-customer` | Supported | Auto-join, menu + shared cart, menu-item detail, checkout, realtime order tracking, payment result; SignalR cart/order wired. |
| Tenant-aware QR URL | Backend `ScanNow` | Supported | `ITenantUrlBuilder` sinh `https://{restaurantSlug}.scannow.site/tables/{qrCodeToken}`. |
| Tenant-aware Redirect | Backend `ScanNow` | Supported | PayOS return/cancel sinh ve dung tenant domain. |

### Gaps Hien Tai Theo Role & Module

> Sau khi merge `feature/SCRUM-30-customer-qr-session`, tenant FE da hien thuc gan nhu toan bo product. Cac gap "thieu UI" o ban truoc da duoc dong.

1. **Owner/Manager**: Day du UI cho restaurant/branch/user, menu/category, table/QR, branch settings (payment-config, voucher), reports overview. Con lai: advanced analytics/export breakdowns.
2. **Staff/Kitchen/Branch Manager**: Hien thuc qua `me/` waiter shell, ket noi `/api/me`, `/api/waiter`, `/api/kitchen` va SignalR `/hubs/orders`. Con lai: smoke test voi order traffic that.
3. **Cashier**: Day du order list/detail/checkout (cash/PayOS)/voucher/cancel.
4. **Customer QR Ordering**: Day du QR/menu/menu-item/cart/order/checkout/order-tracking/payment-result + SignalR; con lai: smoke test domain + PayOS + data that.

### Audit verification 2026-06-13

| Check | Result | Notes |
|---|---|---|
| Backend `dotnet build ScanNow.slnx --no-restore` | Pass | 0 errors, 14 warnings. Warnings include AutoMapper high severity advisory, MailKit/MimeKit moderate advisories, nullable warnings, and one unread parameter. |
| Landing/Admin FE `pnpm lint` on audited `main` | Pass | Temporary worktree needed `pnpm install --frozen-lockfile` first. |
| Tenant/Portal FE `pnpm lint` on `main` (post-merge) | Pass | ESLint and `tsc --noEmit` pass after the SCRUM-30 merge resolution. |
| Tenant/Portal FE `pnpm build` on `main` | Fail | Compiles + type-checks + generates all real pages, but prerender of the built-in `/_global-error` throws `Cannot read properties of null (reading 'useContext')`. It is environment diff error, can skip |

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
- Source `appsettings.json` currently includes `App:ProductionDomain` but not `App:TenantBaseDomain`; deploy env must set `App__TenantBaseDomain=scannow.site`.

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

### P0.4 Cashier UI — implemented

Current:

- Backend supports `CASHIER` role and APIs.
- Tenant FE has a full cashier UI: order list/detail, cash/PayOS checkout with voucher, cancel pending payment (`/api/cashier/branches/{branchId}/orders/...`), branch access + filters.
- Role redirect: `CASHIER -> /cashier/dashboard`.

Remaining:

- Smoke test cash amount-received/change, PayOS QR/link, and voucher application on a deployed tenant.
- Public cash checkout still has weaker accounting than cashier checkout (see P1.2).

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

### P0.6 Secrets management must stay clean

Current:

- Tracked backend `appsettings.json` uses placeholders in the audited branch.
- Secrets may still exist in local `.env`, deployment provider variables, team notes, or logs.

Impact:

- Any secret ever committed/shared/logged should be treated as compromised.
- Production security risk if real values enter git history or runtime logs.

Required:

- Rotate exposed secrets if any real values were previously committed or shared.
- Keep tracked files as placeholders only.
- Use secret manager/env vars.
- Commit only examples/placeholders.

### P0.7 Cross-tenant tests missing

Current:

- Global query filters exist for tenant-related entities in `ApplicationDbContext`.
- Cross-tenant integration tests are still missing.

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

- Backend role: `BRANCH_MANAGER`; FE keeps a legacy `MANAGER` key in the auth redirect map (compatibility).
- Owner/manager user-management role pickers now include `CASHIER` (`BRANCH_MANAGER/STAFF/KITCHEN/CASHIER` for owner; `STAFF/KITCHEN/CASHIER` for manager).

Impact:

- Mostly resolved. The only residue is the legacy `MANAGER` alias kept for compatibility.

Required:

- Optionally drop the legacy `MANAGER` alias once nothing depends on it.

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

### P1.5 Owner/manager operational UI — implemented

Current:

- Owner/manager FE now covers restaurant/branches/users, menu/category management, table/QR management, branch payment config, paper voucher management, and reports overview dashboard.

Remaining:

- Advanced reports/analytics and export breakdowns (P2.4).

### P1.6 Kitchen/staff dashboards — implemented (me/ shell)

Current:

- The `me/` waiter shell wires staff table open/close, waiter pending/ready-to-serve, kitchen grouped items, and SignalR branch subscription (`useBranchOrderUpdates` -> `/hubs/orders`, `JoinBranch`).

Remaining:

- Smoke test under live order traffic; SignalR hub-side branch authorization review (see P0.7).

### P1.7 SignalR scaling

Current:

- Cart hub uses memory cache.
- No backplane.

Impact:

- Multi-instance deploy breaks shared cart/group state.

Required for scale:

- Redis/distributed cache.
- SignalR backplane or managed SignalR.

### P1.8 Dependency and compiler warnings

Current:

- Backend build passes but emits NuGet vulnerability warnings for AutoMapper, MailKit, and MimeKit.
- Build also emits nullable dereference warnings and an unread handler parameter warning.

Required:

- Upgrade or mitigate affected packages.
- Review nullable warnings in repository/base entity code.
- Keep CI configured to surface warnings before release.

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
3. Set backend `App:ProductionDomain=scannow.site` and `App:TenantBaseDomain=scannow.site`. **Pending deploy verification**
4. Add dynamic tenant QR URL generation. **Done**
5. Add dynamic tenant payment redirect. **Done**
6. Keep tracked config placeholder-only and rotate any historically exposed secrets. **Pending/ops verification**

Deliverable:

- Admin and tenant portal can run on target domains.
- QR URL and payment redirect point to tenant domain.

### Phase 2: Tenant customer MVP

1. Build `/tables/[qrCodeToken]`. **Done**
2. Build QR-token auto join/menu/cart. **Done**
3. Build place order. **Done**
4. Build checkout. **Done**
5. Build payment return/cancel. **Done**
6. Integrate `/hubs/cart` and `/hubs/orders`. **Done**
7. Add richer order tracking page. **Done** (`/sessions/[sessionCode]/orders/[orderId]`)

Deliverable:

- Customer can scan QR and place order end-to-end.

### Phase 3: Restaurant operations MVP

1. Add cashier routes and UI. **Done**
2. Add kitchen grouped queue. **Done** (me/ kitchen page)
3. Add waiter ready-to-serve and table open/close. **Done** (me/ shell)
4. Add branch payment config UI. **Done** (owner/manager settings)
5. Add voucher UI. **Done** (paper voucher in settings + cashier voucher)

Deliverable:

- Staff/kitchen/cashier can run a service shift. **UI complete; pending production smoke test.**

### Phase 4: Security and scale

1. Add cross-tenant integration tests.
2. Add hub authorization/hardening.
3. Add Redis/distributed cache if multi-instance.
4. Add audit logging.
5. Add health checks and monitoring.

Deliverable:

- Production readiness baseline.

### Phase 5: Reports and polish

1. Owner/manager reports overview UI **Done**; admin reports UI (in `scan-now-nextjs`) and advanced analytics/export still pending.
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
- `tenant.scannow.site`: **Supported** — owner/manager back office, staff/kitchen/cashier operations, and the customer QR flow are all implemented with realtime SignalR.
- `business.scannow.site`: **Future**, chi can khi business account quan ly nhieu restaurant.
- Tenant = 1 nha hang, nha hang co nhieu chi nhanh: **Dung voi data model va backend hien tai**.
