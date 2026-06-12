# ScanNow Product Requirements Document

Cap nhat: 2026-06-12  
Pham vi: BE `test/deploy` + FE landing/admin + FE tenant/portal.

## 1. Tom tat san pham

ScanNow la mot SaaS cho nha hang/F&B, tap trung vao:

- Landing page de gioi thieu/gom lead.
- Platform admin de quan ly owner, restaurant/tenant, branch, menu, table, session.
- Tenant site theo subdomain cho tung nha hang.
- Quan tri nha hang theo role: owner, branch manager, staff/waiter, kitchen, cashier.
- QR table ordering: khach quet QR, join session, xem menu, dat mon, thanh toan.
- Van hanh order realtime: xac nhan mon, bep xu ly, phuc vu, thu ngan checkout.

Khai niem cot loi:

- `Tenant` = mot `Restaurant`.
- Mot `Restaurant` co nhieu `Branch`.
- Moi `Branch` co menu, category, table, QR session, order, payment config va paper voucher rieng.
- `scannow.site` la domain platform/marketing.
- `<restaurantSlug>.scannow.site` la tenant domain cua nha hang.

## 2. Problem Statement

Nha hang nho va vua thuong gap cac van de:

- Khach phai cho nhan vien den ban moi goi mon.
- Menu giay kho cap nhat, kho danh dau mon tam het.
- Order tu khach, nhan vien, bep va thu ngan khong dong bo realtime.
- Nhieu chi nhanh can chung brand/owner nhung van can quan ly rieng menu, ban, thue, phi dich vu, thanh toan.
- Chu nha hang can tao account staff/kitchen/cashier theo chi nhanh.
- Nha hang can QR ordering co the trien khai nhanh ma khong can app rieng.

ScanNow giai quyet bang tenant subdomain + QR session + role-based operations.

## 3. Product Goals

### 3.1 Business Goals

- Cho phep platform onboard nha hang moi nhanh qua admin.
- Cho phep moi nha hang co tenant domain rieng theo slug.
- Ho tro nha hang co nhieu chi nhanh.
- Tang toc order tai ban bang QR.
- Giam loi truyen dat giua khach, phuc vu, bep, thu ngan.
- Ho tro thu ngan cash/PayOS va voucher giay theo chi nhanh.
- Tao nen tang de mo rong sang loyalty, rating, notification, analytics sau nay.

### 3.2 User Goals

- Khach hang: quet QR, xem menu dung chi nhanh, dat mon, theo doi trang thai, thanh toan.
- Owner: quan ly restaurant, branch, staff, menu, table, thue/phi, voucher, bao cao.
- Branch manager: quan ly nhan su va van hanh trong chi nhanh duoc giao.
- Staff/waiter: mo/close ban, xac nhan order, tao order thu cong, phuc vu mon.
- Kitchen: xem mon can lam, group mon, xac nhan/mark ready.
- Cashier: xem order, ap voucher, checkout cash/PayOS, huy payment pending.
- Platform admin: quan ly owner, restaurant, branch, menu/table supervision.

## 4. Personas

### 4.1 Platform Admin

Nguoi van hanh ScanNow. Can:

- Tao owner account.
- Tao restaurant/tenant va slug.
- Xem danh sach restaurant/branch.
- Ban/unban owner/restaurant.
- Supervise menu, table, active sessions.
- Doc bao cao dashboard platform.

### 4.2 Restaurant Owner

Chu nha hang/chu chuoi. Can:

- Quan ly thong tin restaurant va slug.
- Tao/sua/tat mo chi nhanh.
- Tao branch manager/staff/kitchen/cashier.
- Xem toan bo chi nhanh cua restaurant.
- Quan ly cau hinh branch: VAT, service charge, payment config, voucher.
- Xem bao cao doanh thu theo branch/date/top items.

### 4.3 Branch Manager

Quan ly mot chi nhanh. Can:

- Quan ly users trong chi nhanh.
- Quan ly menu/table/order/payment config cua chi nhanh neu duoc cho phep.
- Theo doi van hanh live trong ca.
- Xem bao cao chi nhanh.

### 4.4 Staff/Waiter

Nhan vien phuc vu. Can:

- Xem branch cua minh.
- Mo ban de tao QR session.
- Xac nhan order moi.
- Tao order thu cong cho khach khong dung QR.
- Theo doi mon ready va mark served.

### 4.5 Kitchen

Bep. Can:

- Xem order/item dang cho xac nhan.
- Xac nhan item/order.
- Xem item group theo mon + trang thai + do uu tien.
- Mark item ready.

### 4.6 Cashier

Thu ngan. Can:

- Xem active/paid/all orders trong branch.
- Tim order theo order number, table, customer, phone, session code.
- Ap voucher.
- Checkout cash voi amount received/change.
- Tao PayOS payment link/QR.
- Huy pending payment khi khach doi phuong thuc.

### 4.7 Public Customer

Khach an tai ban. Can:

- Quet QR tai ban.
- Join session dang mo.
- Xem menu cua dung branch.
- Them mon vao gio hang chung.
- Dat order.
- Theo doi order realtime.
- Thanh toan PayOS hoac thanh toan tai quay/cash.

## 5. Domain Architecture Requirement

Kien truc chinh thuc cua san pham:

```text
scannow.site                  Landing page & Platform admin (/admin)
api.scannow.site              Backend API
<tenant>.scannow.site         Tenant site cua Restaurant (dung cho tat ca roles: owner, manager, staff, kitchen, cashier, customer)
```

`app.scannow.site` khong duoc su dung trong MVP (day la reserved domain duy nhat cho tuong lai neu can centralized portal). Tat ca tinh nang va role deu chay trong subdomain cua tung tenant (vi du `pho24.scannow.site`).

- Tenant = `Restaurant`.
- Mot `Restaurant` co nhieu `Branch` (chi nhanh ben trong tenant), khong dung branch subdomain.
- Platform admin chi dung `scannow.site/admin`.
- Restaurant users dang nhap tai `<tenant>.scannow.site/login`.
- Owner, manager, staff, kitchen, cashier va customer pages nam hoan toan trong tenant FE.

- Dang nhap mot lan roi chon nha hang/tenant.
- Mot user quan ly nhieu restaurant.
- Can portal trung tam cho support/internal ops.
- Can tach tenant public site va back-office portal.

## 6. Current Product Scope From Code

### 6.1 Da co trong backend `test/deploy`

- Auth: register/login/google login/refresh/logout/email verification/change password.
- Role: `ADMIN`, `OWNER`, `BRANCH_MANAGER`, `STAFF`, `KITCHEN`, `CASHIER`.
- Tenant resolution: `X-Tenant-Slug` va host subdomain.
- Tenant isolation: EF global query filter tren `Branch` theo `RestaurantId`.
- Admin owners: list/create/update/ban/unban/available owners.
- Admin restaurants: list/get by GUID/get by slug/create/update/ban/unban.
- Admin branch supervision: branch by GUID/slug, menu/category/table/session read.
- Owner restaurant: get/update own restaurant.
- Owner branches: create/list/get/update/active/inactive.
- Owner/manager user management.
- Menu/category management.
- Image upload endpoint cho menu item images.
- Table/QR management.
- Table open/close sessions.
- Public QR APIs: get table by QR token, join session, get menu, place order, checkout, payment status/cancel, get order detail.
- Shared cart SignalR hub.
- Order status SignalR hub.
- Kitchen APIs.
- Waiter APIs.
- Cashier APIs.
- Branch payment config.
- Paper voucher.
- Owner/manager/admin reports.

### 6.2 Da co trong FE landing/admin `scan-now-nextjs`

- Landing page i18n `vi/en`.
- Lead capture API route qua Resend.
- `/admin` login + admin shell.
- Owner management UI.
- Restaurant management UI.
- Restaurant detail by slug.
- Branch supervision tabs.
- Category/menu/table/session read views.

### 6.3 Da co trong FE tenant/portal `scan-now-customer`

- Login page.
- Tenant slug extraction tu browser hostname.
- Axios interceptor gui `X-Tenant-Slug`.
- Auth bootstrap + refresh token flow.
- Protected route theo role.
- Owner pages: restaurant, branches, users.
- Manager users page.
- Placeholder/dashboard routes cho admin/owner/manager/staff/kitchen.

### 6.4 Chua hoan tat end-to-end

- Public customer QR ordering UI chua co route trong FE tenant.
- Payment return/cancel pages chua co.
- Cashier UI chua co.
- Kitchen/staff dashboards hien chua day du API integration.
- QR URL generation chua dynamic theo restaurant slug.
- PayOS redirect URL chua dynamic theo tenant.
- Production domain config chua hoan tat trong appsettings/env.

## 7. MVP Definition

### 7.1 MVP Platform

- Landing page tai `scannow.site`.
- Admin tai `scannow.site/admin`.
- Admin login.
- Admin create owner.
- Admin create restaurant voi slug.
- Admin view restaurant/branch/menu/table/session.
- Admin ban/unban owner/restaurant.

### 7.2 MVP Tenant Back Office

- Tenant domain `<slug>.scannow.site`.
- Login tai `<slug>.scannow.site/login`.
- Owner xem/sua restaurant.
- Owner tao/sua/tat mo branch.
- Owner tao staff/kitchen/cashier/branch manager moi user dung 1 branch.
- Branch manager quan ly user trong branch.
- Staff/waiter mo/close table session.
- Kitchen xem va mark ready order items.
- Cashier checkout cash/PayOS.

### 7.3 MVP Customer Ordering

- QR URL tro ve `<restaurantSlug>.scannow.site/tables/{qrCodeToken}`.
- Page table join session.
- Page menu theo session.
- Shared cart.
- Place order.
- Order status realtime.
- Payment option: PayOS hoac pay at cashier.
- Payment return/cancel page.

## 8. Non-MVP / Future Scope

- Marketplace/integration voi POS ngoai.
- Multi-restaurant owner account.
- Loyalty, membership, points.
- Customer account/login.
- Rating/review UI.
- Inventory management.
- Reservation/table booking.
- Advanced analytics/export.
- Multi-language tenant menu.
- Notification push/SMS.
- Offline/PWA full mode.
- Webhook PayOS callback hardening neu can.

## 9. Success Metrics

### 9.1 Onboarding Metrics

- Time-to-create tenant: duoi 5 phut tu admin.
- Time-to-create first branch/table/menu: duoi 30 phut.
- QR scan success rate: > 99%.
- Tenant slug conflict rate duoi nguong chap nhan.

### 9.2 Ordering Metrics

- Customer can place order in under 90 seconds after scan.
- Order appears in kitchen/waiter views within 2 seconds.
- Realtime update delivery success > 99% trong dieu kien ket noi binh thuong.
- Checkout success rate PayOS > 95% excluding gateway issues.

### 9.3 Operations Metrics

- Staff/kitchen/cashier can complete order lifecycle without admin support.
- Cash checkout records amount received/change accurately.
- Voucher overuse = 0.
- Cross-tenant data exposure = 0.

## 10. Key Assumptions

- Moi restaurant co slug global unique.
- Moi branch slug unique trong restaurant.
- Moi managed user thuoc exactly one branch theo validator hien tai.
- Tenant subdomain la restaurant slug, khong phai branch slug.
- Khach order trong context cua branch/table/session, khong chi theo tenant.
- Platform admin khong bi tenant filter vi request tren reserved domain/subdomain khong resolve tenant.
- Tenant FE duoc deploy voi wildcard domain de moi `<slug>.scannow.site` tra ve cung mot app.

## 11. Critical Product Decisions

### 11.1 Tenant = Restaurant, Branch = Operational Unit

Quyet dinh nay phu hop voi code:

- `Restaurant.Slug` la tenant slug.
- `Branch.RestaurantId` gan branch vao tenant.
- `Branch` chua VAT/service charge/payment config/table/menu/order.
- Tenant query filter ap theo `Branch.RestaurantId`.

### 11.2app.scannow.site khong su dung trong MVP (Reserved Only)

Quyet dinh chinh thuc:

- Platform admin chay tren `scannow.site/admin`.
- Moi nha hang co tenant domain rieng `<tenant>.scannow.site`.
- Owner/manager/staff/kitchen/cashier va ca public customer deu truy cap truc tiep trong tenant domain cua nha hang.
- `app.scannow.site` khong duoc implement trong MVP. Day la reserved domain cho tuong lai neu co nhu cau:
  - Central login hoac tenant selector.
  - Mot user quan ly nhieu restaurant.
  - Centralized support portal.

### 11.3 QR URL phai la tenant-aware

Backend hien tao QR URL tu `App:FrontendBaseUrl` + `App:QrTablePath`. De dung voi tenant domain, QR URL can:

```text
https://{restaurantSlug}.scannow.site/tables/{qrCodeToken}
```

Neu khong, khach co the vao domain sai va `X-Tenant-Slug` khong khop tenant.

### 11.4 Payment redirect phai tenant-aware

CheckoutService va CashierService hien build redirect URL tu `App:ClientUrl`/`App:FrontendBaseUrl`. Public customer payment phai quay ve tenant domain, khong phai mot base URL chung.

## 12. Acceptance Criteria Tong

- `scannow.site` hien landing, SEO base URL dung `https://scannow.site`.
- `scannow.site/admin` login duoc admin va CRUD owner/restaurant.
- Admin tao restaurant slug `pho24`.
- DNS wildcard `*.scannow.site` tro ve tenant FE.
- `https://pho24.scannow.site/login` gui `X-Tenant-Slug: pho24`.
- Backend resolve tenant `pho24` thanh RestaurantId.
- Request tren tenant khong doc duoc branch cua restaurant khac.
- Owner/manager/staff/kitchen/cashier role flow hoat dong tren tenant domain.
- QR cua branch trong `pho24` tro ve `https://pho24.scannow.site/tables/{token}`.
- Khach quet QR, join session, xem menu dung branch, place order, theo doi status.
- Cashier checkout cash sinh payment `SUCCESS`; PayOS sinh pending payment va hoan tat khi gateway paid.
- Payment return/cancel route ton tai tren tenant FE.

