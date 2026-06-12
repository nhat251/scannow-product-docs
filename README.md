# ScanNow Product Documentation

Cap nhat: 2026-06-12  
Nguon doc:

- Backend: `/home/nhat/fpt/ScanNow`, branch `test/deploy`, commit `bb89f19` (`feat/multitenant` merged).
- Landing/Admin FE: `/home/nhat/fpt/scan-now-nextjs`, branch `main`, commit `e923a6e`.
- Tenant/Portal FE: `/home/nhat/fpt/scan-now-customer`, branch `main`, commit `522e1c6`.

## Ket luan nhanh ve kien truc domain

Kien truc chinh thuc cua san pham:

```text
scannow.site                    -> landing page
scannow.site/admin              -> platform admin cua ScanNow
api.scannow.site                -> backend API
<tenant>.scannow.site           -> toan bo app cua mot nha hang (owner, manager, staff, kitchen, cashier, customer)
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

`app.scannow.site` khong duoc su dung/implement trong MVP. Subdomain nay duoc reserved cho tuong lai neu can central login, tenant switcher, hoac user thuoc nhieu nha hang.

Kien truc nay la huong dung va backend `test/deploy` da ho tro phan cot loi cua tenant:

- Tenant = `Restaurant`.
- Mot tenant/restaurant co nhieu `Branch` (chi nhanh ben trong tenant), khong co branch subdomain.
- Backend resolve tenant tu `X-Tenant-Slug` hoac subdomain host.
- EF Core co global query filter tren `Branch` theo `RestaurantId`.
- Admin APIs da co route theo slug: `/api/admin/restaurants/by-slug/{slug}`.
- CORS co co che allow wildcard subdomain neu cau hinh `App:ProductionDomain=scannow.site`.

Nhung chua the goi la hoan tat end-to-end vi:

- Tenant FE (scan-now-customer) la app chung cho ca owner/manager/staff/kitchen/cashier/customer.
- Landing/admin FE (scan-now-nextjs) chi giu landing va platform admin.
- Quy trinh QR ordering, payment return/cancel dang duoc tich hop truc tiep tren tenant domain.
- app.scannow.site chi la reserved future domain, khong phai noi chua staff portal trong MVP.

## Bo tai lieu

1. [Product PRD](01-product-prd.md)  
   Tong quan san pham, muc tieu, personas, pham vi, success metrics, MVP/Non-MVP.

2. [Domain, Tenant & Deployment Spec](02-domain-tenant-deployment-spec.md)  
   Kien truc domain mong muon, trang thai code hien tai, cau hinh can co, quyet dinh co can `app.scannow.site` hay khong.

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
