# ScanNow Product Documentation

Cap nhat: 2026-06-13
Nguon doc:

- Backend: `/home/nhat/fpt/ScanNow`, audited branch `test/deploy`, commit `c41c3c4`.
- Landing/Admin FE: `/home/nhat/fpt/scan-now-nextjs`, audited branch `main`, commit `7d59f7c`.
- Tenant/Portal FE: `/home/nhat/fpt/scan-now-customer`, audited branch `main`, commit `1c44a03`.
- Product docs: `/home/nhat/fpt/scannow-product-docs`, branch `main`, base commit `9a0d189`.

## Ket luan nhanh ve kien truc domain

Kien truc chinh thuc cua san pham:

```text
scannow.site                    -> landing page
scannow.site/admin              -> platform admin cua ScanNow
api.scannow.site                -> backend API
<tenant>.scannow.site           -> toan bo app cua mot nha hang (owner, manager, staff, kitchen, cashier, customer)
business.scannow.site           -> future business portal cho khach hang lon quan ly nhieu nha hang
```

Trong tenant domain se chua tat ca role/app cua nha hang:
- `<tenant>.scannow.site/login`
- `<tenant>.scannow.site/owner/...`
- `<tenant>.scannow.site/manager/...`
- `<tenant>.scannow.site/staff/...`
- `<tenant>.scannow.site/kitchen/...`
- `<tenant>.scannow.site/cashier/...`
- `<tenant>.scannow.site/tables/{qrCodeToken}`
- `<tenant>.scannow.site/sessions/{sessionCode}/menu`
- `<tenant>.scannow.site/payment/return`
- `<tenant>.scannow.site/payment/cancel`

`business.scannow.site` khong nam trong MVP hien tai. Day la future portal cho khach hang business lon can quan ly nhieu nha hang, nhieu quan ly, tenant switcher, business-level reports/billing. Van hanh hang ngay cua tung nha hang van nam trong `<tenant>.scannow.site`.

Neu wildcard DNS/deployment da lam `business.scannow.site` resolve duoc, domain nay phai duoc parked/future-only hoac `business` phai duoc them vao reserved subdomain list truoc khi route qua tenant stack.

Kien truc nay la huong dung va backend branch `test/deploy` da ho tro phan cot loi cua tenant:

- Tenant = `Restaurant`.
- Mot tenant/restaurant co nhieu `Branch` (chi nhanh ben trong tenant), khong co branch subdomain.
- Backend resolve tenant tu `X-Tenant-Slug` hoac subdomain host.
- EF Core áp dụng Global Query Filter trên **tất cả** thực thể (Branch, Order, MenuItem...) theo `RestaurantId` để đảm bảo Data Isolation tuyệt đối. Auth Service cũng chặn login trái tenant.
- Admin APIs da co route theo slug: `/api/admin/restaurants/by-slug/{slug}`.
- CORS allow wildcard subdomain cua `scannow.site` khi `App:ProductionDomain=scannow.site`.
- QR URL va PayOS redirect da sinh dong theo tenant domain thong qua `ITenantUrlBuilder`; production phai set `App:TenantBaseDomain=scannow.site`.
- Tenant FE (sau merge `feature/SCRUM-30-customer-qr-session`) da hien thuc gan nhu toan bo product: owner/manager back office (menu, table, settings, payment-config, voucher, reports), `me/` waiter shell cho staff/kitchen/branch-manager, full cashier, customer QR/menu/checkout/order-tracking/payment, va SignalR cart/order realtime.
- Domain production da duoc setup; van can smoke test runtime sau moi lan deploy.

Trang thai con lai:

- Tenant FE (scan-now-customer) la app chung cho ca owner/manager/staff/kitchen/cashier/customer.
- Landing/admin FE (scan-now-nextjs) chi giu landing va platform admin.
- FE SignalR client cho cart/order realtime da duoc wire (`/hubs/cart`, `/hubs/orders`); backend hub van dung memory cache (chua backplane).
- Can production smoke test toan bo flow (QR, operations, PayOS, realtime) tren domain that.
- Public CASH checkout hien van la known accounting risk vi public flow mark order `Completed` nhung payment van `PENDING`, nen nen xem la "pay at cashier" cho den khi cashier confirm.

## Audit verification 2026-06-13

- Backend `dotnet build ScanNow.slnx --no-restore`: pass, 0 errors, 14 warnings.
- Landing/Admin FE `pnpm lint` on audited `main` temp worktree: pass after temp `pnpm install --frozen-lockfile`.
- Tenant/Portal FE `pnpm lint` (post SCRUM-30 merge): pass (ESLint + `tsc --noEmit`).
- Tenant/Portal FE `pnpm build`: fail — compiles/type-checks/generates all real pages, but prerender of built-in `/_global-error` throws `useContext` null (reproduces on pre-merge `main`; Next 16 + React 19 + `reactCompiler` but it is environment dif error (can skip)).
- Backend warnings include NuGet vulnerability advisories for AutoMapper/MailKit/MimeKit and nullable/unread-parameter warnings; see roadmap/security docs.

## Development Guidelines & Quality Rules

De dam bao chat luong code cho tat ca cac plan moi hoac tinh nang moi, cac quy tac duoi day la **BAT BUOC** va phai duoc ap dung lien tuc:

1. **Frontend (FE) - Linting & Typing**:
   - Bat buoc phai pass toan bo cac warning va error tu Linter va TypeScript compiler.
   - Khi hoan thanh code/tinh nang tren FE, luon luon phai chay `npm run lint` (hoac `npm run lint:fix`).
   - Chi chap nhan ket qua Linter la sach tuyet doi: **Khong co bat ky 1 warning hay error nao** (0 errors, 0 warnings).

2. **Backend (BE) - Testing**:
   - Voi moi doan code hoac tinh nang moi them vao backend, **bat buoc** phai viet testcase (Unit Test / Integration Test) tuong ung.
   - Code chi duoc tinh la hoan thanh ("pass") khi toan bo cac testcase deu chay thanh cong va bao phu day du cac truong hop, dam bao logic chay dung nhu design.

## Bo tai lieu

1. [Product PRD](01-product-prd.md)
   Tong quan san pham, muc tieu, personas, pham vi, success metrics, MVP/Non-MVP.

2. [Domain, Tenant & Deployment Spec](02-domain-tenant-deployment-spec.md)
   Kien truc domain, trang thai code hien tai, cau hinh can co, quyet dinh `business.scannow.site` future portal.

3. [Functional Specification](03-functional-spec.md)
   Dac ta chuc nang theo role/module: landing, admin, owner, branch manager, staff/waiter, kitchen, cashier, public customer QR.

4. [API Specification](04-api-spec.md)
   Tong hop API, realtime hubs, auth, request/response chinh, role access, han che hien tai.

5. [Architecture & Data Model](05-architecture-data-model.md)
   Kien truc backend/frontend, data model, tenant isolation, order/payment lifecycle, realtime architecture.

6. [Frontend Specification](06-frontend-specs.md)
   Dac ta 2 FE repo, route, screen, state management, API mapping, domain routing expectations.

7. [Operations, Security & NFR](07-operations-security-nfr.md)
   Cau hinh moi truong, CORS, secrets, auth/session, observability, performance, compliance, test strategy.

8. [Gaps, Risks & Roadmap](08-gaps-roadmap.md)
   Cac gap bat buoc truoc production, rui ro thiet ke, roadmap theo phase.

9. [ScanNow Presentation Source](09-scannow-presentation-source.md)
   Tai lieu tong hop dai, dung lam source cho NotebookLM/slide thuyet trinh: cong nghe, kien truc, flow, in-scope, gaps, future roadmap.

## Cach doc nhanh

Neu can tra loi cau hoi domain:

- Doc [Domain, Tenant & Deployment Spec](02-domain-tenant-deployment-spec.md).
- Sau do doc muc `P0` trong [Gaps, Risks & Roadmap](08-gaps-roadmap.md).

Neu can build tiep FE tenant:

- Doc [Frontend Specification](06-frontend-specs.md).
- Doc public customer API trong [API Specification](04-api-spec.md).

Neu can review backend tenant/security:

- Doc [Architecture & Data Model](05-architecture-data-model.md).
- Doc [Operations, Security & NFR](07-operations-security-nfr.md).
