# Plan Cập Nhật Docs + Code Cho Tenant Domain MVP

## Summary

Chốt kiến trúc chính thức:

scannow.site -> landing page
scannow.site/admin -> platform admin của ScanNow
api.scannow.site -> backend API
<restaurantSlug>.scannow.site -> toàn bộ app của một nhà hàng

Trong tenant domain sẽ chứa tất cả role/app của nhà hàng:

<tenant>.scannow.site/login
<tenant>.scannow.site/owner/...
<tenant>.scannow.site/manager/...
<tenant>.scannow.site/staff/...
<tenant>.scannow.site/kitchen/...
<tenant>.scannow.site/cashier/...
<tenant>.scannow.site/tables/{qrCodeToken}
<tenant>.scannow.site/sessions/{sessionCode}/menu
<tenant>.scannow.site/payment/return
<tenant>.scannow.site/payment/cancel

business.scannow.site không implement trong MVP. Chỉ reserve cho tương lai nếu cần business account quản lý nhiều nhà hàng, tenant switcher, business-level reports/billing, hoặc nhiều user quản lý cùng một nhóm restaurant.

## Docs Update Plan

Cập nhật bộ docs trong scannow-product-docs để không còn mơ hồ về business.scannow.site.

Các thay đổi nội dung bắt buộc:

- README.md: ghi rõ domain decision cuối cùng: landing/admin ở scannow.site, API ở api.scannow.site, mọi restaurant role nằm trong <tenant>.scannow.site.
- 01-product-prd.md: cập nhật product model: tenant = restaurant, branch = chi nhánh bên trong tenant, không có branch subdomain.
- 02-domain-tenant-deployment-spec.md: đổi business.scannow.site từ “optional” sang “reserved only, not used in MVP”; thêm route map đầy đủ cho owner/manager/staff/kitchen/cashier/customer.
- 03-functional-spec.md: mỗi role phải được mô tả là chạy trong tenant domain, không chạy ở business.scannow.site.
- 06-frontend-specs.md: tenant FE scan-now-customer là app chung cho owner/manager/staff/kitchen/cashier/customer; landing/admin FE scan-now-nextjs chỉ giữ landing và platform admin.
- 08-gaps-roadmap.md: cập nhật gap hiện tại: owner/manager có một phần UI, staff/kitchen placeholder, cashier có placeholder nhưng thiếu UI đầy đủ, customer QR ordering đã có route cơ bản và cần smoke test production.
- Tất cả docs phải thống nhất: business.scannow.site = reserved future domain, không phải nơi chứa staff portal trong MVP.

Acceptance docs:

- Search không còn câu nào nói staff/owner/cashier/kitchen nằm ở business.scannow.site trong MVP.
- Có route map rõ ràng cho <tenant>.scannow.site.
- Có bảng trạng thái code hiện tại: supported, partial, missing.
- Có roadmap code theo thứ tự P0/P1/P2.

## Backend Code Plan

Thêm tenant-aware URL generation.

Config chuẩn:

App**TenantBaseDomain=scannow.site
App**ClientUrl=https://scannow.site
App**FrontendBaseUrl=https://scannow.site
App**ProductionDomain=scannow.site
App\_\_QrTablePath=/tables

Implement URL builder dùng chung:

BuildTenantBaseUrl(slug) -> https://{slug}.scannow.site
BuildTenantTableUrl(slug, qrCodeToken) -> https://{slug}.scannow.site/tables/{qrCodeToken}
BuildTenantPaymentUrl(slug, result, query) -> https://{slug}.scannow.site/payment/{result}?...
BuildPlatformUrl(path) -> https://scannow.site/{path}

Rules:

- Slug normalize bằng trim/lowercase.
- Reserved slugs không được build tenant URL: www, api, admin, app, business, staging, localhost.
- Nếu thiếu slug hoặc thiếu TenantBaseDomain, fallback về App:ClientUrl / App:FrontendBaseUrl.
- Query params phải encode đúng.

Sửa QR:

- CreateTableAsync và RegenerateQrAsync phải sinh QR URL dạng:

https://{restaurantSlug}.scannow.site/tables/{qrCodeToken}

- QR image tiếp tục dùng table.QrCodeUrl.

Sửa customer PayOS redirect:

https://{restaurantSlug}.scannow.site/payment/return?sessionCode={sessionCode}&orderId={orderId}
https://{restaurantSlug}.scannow.site/payment/cancel?sessionCode={sessionCode}&orderId={orderId}

Sửa cashier PayOS redirect:

https://{restaurantSlug}.scannow.site/payment/return?orderId={orderId}&source=cashier
https://{restaurantSlug}.scannow.site/payment/cancel?orderId={orderId}&source=cashier

Thêm public API join bằng QR token:

POST /api/public/tables/{qrCodeToken}/join

Behavior:

- Tìm table bằng QR token.
- Table, branch, restaurant phải active.
- Phải có active session đã được staff/cashier mở.
- Nếu chưa mở bàn, trả lỗi rõ: Table is not open yet.
- Response phải có sessionCode, tableId, branchId, tableNumber, branchName, expiresAt.
- Request từ tenant A không được join QR/session của tenant B.

## Tenant FE Code Plan

scan-now-customer là app tenant duy nhất cho restaurant users và customers.

Route map cần có:

/login
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

Auth/role routing:

- OWNER -> /owner/users
- MANAGER -> /manager/users
- BRANCH_MANAGER -> /manager/users
- STAFF -> /staff/dashboard
- KITCHEN -> /kitchen/dashboard
- CASHIER -> /cashier/dashboard
- ADMIN không phải tenant role chính; platform admin dùng scannow.site/admin.

Tenant behavior:

- FE lấy slug từ hostname pho24.scannow.site -> pho24.
- FE gửi X-Tenant-Slug: pho24 cho mọi API request tenant.
- Reserved subdomain app/business trả null, không resolve tenant. Nếu code hiện tại chưa reserve `business`, phải thêm trước khi route `business.scannow.site` vào tenant stack.
- Local dev dùng NEXT_PUBLIC_DEV_TENANT_SLUG.

Customer QR flow:

- /tables/[qrCodeToken] gọi table info và join-by-QR.
- Nếu bàn chưa mở: hiển thị state gọi nhân viên.
- Nếu join thành công: redirect sang /sessions/{sessionCode}/menu.
- /sessions/{sessionCode}/menu fetch menu, add cart, đặt món.
- Cart MVP dùng localStorage key cart:{tenantSlug}:{sessionCode}.
- Submit order bằng POST /api/public/sessions/{sessionCode}/orders.
- Checkout bằng POST /api/public/sessions/{sessionCode}/checkout.
- PayOS redirect sang gateway, return/cancel về tenant domain.
- Payment result page đọc trạng thái từ backend, không trust query string.

Cashier MVP:

- Thêm route /cashier/dashboard và /cashier/orders.
- Tối thiểu cashier xem branch orders, xem detail, checkout cash/PayOS, cancel pending payment.
- Nếu chưa đủ thời gian làm UI đầy đủ, phải có protected placeholder không dead-end và role redirect vẫn đúng.

Staff/kitchen MVP:

- Staff/kitchen hiện có placeholder; plan giữ trong tenant domain.
- P1 mới hoàn thiện realtime staff/kitchen nếu scope không đủ cho đợt đầu.

## Landing/Admin FE Plan

scan-now-nextjs chỉ phục vụ:

scannow.site
scannow.site/admin

Yêu cầu:

- Landing metadata/base URL dùng https://scannow.site.
- Admin API URL dùng https://api.scannow.site.
- /admin không bị locale middleware redirect.
- Admin tạo restaurant phải có slug.
- Admin detail restaurant hiển thị tenant URL:

https://{slug}.scannow.site

Không thêm route, redirect, hoặc deployment cho business.scannow.site.

## Test Plan

dotnet build ScanNow/ScanNow.Web/ScanNow.Web.csproj --no-restore
dotnet test

Test cases bắt buộc:

- QR URL create/regenerate dùng tenant domain.
- Customer PayOS return/cancel dùng tenant domain.
- Cashier PayOS return/cancel dùng tenant domain.
- Join by QR token thành công khi table có active session.
- Join by QR token lỗi rõ khi bàn chưa open.
- Tenant A không join/read được QR/session/order của Tenant B.
- api, app, admin, www không resolve thành tenant.
- CORS accept https://pho24.scannow.site.

Frontend tenant:

pnpm lint
pnpm build

Manual smoke:

- pho24.scannow.site/login login owner/staff/kitchen/cashier đúng redirect.
- pho24.scannow.site/tables/{token} join được bàn đã open.
- Customer order + checkout PayOS return về pho24.scannow.site/payment/return.
- highlands.scannow.site không đọc được data của pho24.

Frontend landing/admin:

pnpm lint
pnpm build

Manual smoke:

- scannow.site mở landing.
- scannow.site/admin mở admin.
- Admin tạo restaurant slug pho24.
- Admin hiển thị tenant URL https://pho24.scannow.site.

## Assumptions

- business.scannow.site không dùng trong MVP.
- Staff, kitchen, cashier, owner, manager đều nằm trong cùng tenant domain.
- Tenant = restaurant.
- Branch là chi nhánh thuộc restaurant, không có subdomain riêng.
- Khách chỉ join được session sau khi staff/cashier mở bàn.
- Customer cart MVP dùng localStorage, chưa bắt buộc SignalR shared cart.
- Payment success phải xác nhận từ backend/payment status, không dựa vào query string.

làm theo plan:

1. docs: /home/nhat/fpt/scannow-product-docs
2. landing/admin: /home/nhat/fpt/scan-now-nextjs
3. BE: /home/nhat/fpt/ScanNow
4. FE customer, tenant, staff: /home/nhat/fpt/scan-now-nextjs
