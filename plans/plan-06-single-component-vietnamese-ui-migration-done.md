# Plan 06: Tách một component mỗi file và Việt hóa toàn bộ `scan-now-customer`

Ngày lập plan: 2026-06-14

## 0. Phạm vi và nguồn audit

Repository thực hiện:

```text
/home/nhat/fpt/scan-now-customer
```

Repository tài liệu:

```text
/home/nhat/fpt/scannow-product-docs
```

Baseline đã audit:

```text
scan-now-customer:
  branch: main
  commit: 6c61672

scannow-product-docs:
  branch: main
  commit: d814475
```

Phạm vi của plan này chỉ áp dụng cho tenant/customer frontend `scan-now-customer`, gồm:

- Public customer QR/menu/cart/order/payment.
- Owner portal.
- Branch manager portal.
- Staff/waiter portal.
- Kitchen portal.
- Cashier portal.
- Các route placeholder/admin còn nằm trong repository.
- Toàn bộ text người dùng nhìn thấy: heading, label, button, placeholder, toast, validation, dialog, empty state, error state, `aria-label`, `alt`, nội dung in hóa đơn và nội dung file Excel xuất ra.

Không thay đổi:

- Backend API contract.
- Route URL hiện tại.
- Role value, enum value hoặc payload gửi backend.
- Business flow, validation rule, query key, mutation order, redirect, cache invalidation, SignalR flow.
- Layout, responsive breakpoint, spacing, màu sắc, typography, DOM hierarchy trong phase tách component.

## 1. Mục tiêu bắt buộc

### 1.1 Một file chỉ chứa một component

Sau migration:

- Mỗi file `.tsx` chỉ được khai báo tối đa một React component.
- Áp dụng cho:
  - `src/app/**/*.tsx`
  - `src/components/**/*.tsx`
  - Component dùng `function`, arrow function, `forwardRef`, `memo`.
  - Component export và component private trong cùng file.
- Ngoại lệ duy nhất:
  - `src/components/ui/**` được tạo hoặc quản lý theo shadcn/ui.
- Ngoại lệ shadcn chỉ áp dụng cho quy tắc số lượng component, không miễn trừ yêu cầu tiếng Việt nếu primitive có text hoặc accessibility label mặc định hiển thị cho người dùng.

Ví dụ không còn được phép:

```tsx
export const Page = () => <Panel />;

const Panel = () => <div />;
```

Phải tách thành:

```text
page.tsx
panel.tsx
```

### 1.2 Page route phải mỏng

File `src/app/**/page.tsx` chỉ chịu trách nhiệm:

1. Đọc `params` hoặc `searchParams`.
2. Decode/normalize route params nếu cần.
3. Bọc `ProtectedRoute` nếu route yêu cầu auth.
4. Render đúng một page/container component.

Không đặt trong `page.tsx`:

- Query/mutation logic.
- Mapping enum.
- Form schema.
- Helper format.
- Component phụ.
- JSX nghiệp vụ dài.

Mục tiêu mềm:

- Route page đồng bộ: dưới 40 dòng khi có thể.
- Không thêm wrapper DOM nếu wrapper không tồn tại ở baseline.

### 1.3 Logic dùng chung phải đi vào hook

Logic có một trong các đặc điểm sau phải được cân nhắc tách thành custom hook:

- Quản lý nhiều state liên quan đến cùng một workflow.
- Kết hợp query + mutation + derived state.
- Có side effect theo branch/order/session.
- Dùng lại giữa mobile và desktop.
- Dùng lại giữa owner và manager.
- Dùng lại giữa waiter và cashier.
- Có reset flow phức tạp sau mutation.

Quy tắc:

- Hook đặt trong `src/hooks/**`.
- Tên hook bắt đầu bằng `use`.
- Một file hook có một hook chính.
- Hook không render JSX.
- Hook không chứa text UI nếu text có thể chuyển sang component hoặc presentation mapper.
- Hook trả về dữ liệu và action có tên rõ nghĩa, không trả một object state khổng lồ không có cấu trúc.

Ví dụ:

```text
src/hooks/cashier/use-cashier-dashboard.ts
src/hooks/cashier/use-cashier-manual-order.ts
src/hooks/cashier/use-cashier-payment.ts
src/hooks/waiter/use-waiter-dashboard.ts
src/hooks/reports/use-report-dashboard.ts
src/hooks/settings/use-branch-settings.ts
```

### 1.4 Hàm dùng chung phải đi vào helper

Pure function phải đặt trong file `.ts`, không để lẫn trong page/component lớn:

- Format tiền, ngày, giờ, phần trăm.
- Mapping status/role/method sang text.
- Tính toán tổng, phần trăm, chart series.
- Chuẩn hóa API value.
- Build request payload.
- Build query.
- Sort/filter.
- Download blob/file name.
- Receipt/Excel transformation không phụ thuộc React.

Quy tắc:

- Không tạo `helpers.tsx` chỉ để chứa function trả JSX.
- Function trả JSX được xem là component và phải ở file component riêng.
- Helper theo domain đặt gần domain:

```text
src/helpers/auth/
src/helpers/orders/
src/helpers/payments/
src/helpers/reports/
src/helpers/tables/
src/helpers/users/
src/helpers/api-errors/
```

- Helper thật sự dùng toàn app mới đặt ở `src/lib` hoặc `src/helpers`.
- Không gom mọi thứ vào một `helpers.ts` quá lớn.

### 1.5 Toàn bộ giao diện phải là tiếng Việt

Sau migration:

- Không còn text giao diện tiếng Anh trong `scan-now-customer`.
- Role backend phải map sang tiếng Việt.
- Enum/status/method backend phải map sang tiếng Việt.
- Không render raw enum khi mapper không nhận ra giá trị.
- Không đưa raw backend English error ra UI.
- Không dùng `en-US` để format dữ liệu hiển thị.

Các thuật ngữ được phép giữ nguyên theo allowlist:

- `ScanNow`
- `PayOS`
- `QR`
- `VAT`
- `API` khi thật sự cần hiển thị
- `URL`
- `Email`
- Tên riêng, tên nhà hàng, tên chi nhánh, tên món, username và dữ liệu do người dùng nhập.

Các từ không nằm trong allowlist phải Việt hóa, ví dụ:

- `Dashboard` -> `Tổng quan`
- `Owner` -> `Chủ nhà hàng`
- `Branch Manager` -> `Quản lý chi nhánh`
- `Staff` -> `Nhân viên phục vụ`
- `Kitchen` -> `Bộ phận bếp`
- `Cashier` -> `Thu ngân`
- `Menu` -> `Thực đơn`
- `Settings` -> `Cài đặt`
- `Order` -> `Đơn hàng`
- `Payment` -> `Thanh toán`

## 2. Kết quả audit hiện tại

### 2.1 Quy mô code

Tại baseline:

```text
196 file TSX
55 file TS
55 route page.tsx
76 component TSX ngoài src/components/ui
```

Repo chưa có:

- Unit test framework.
- Component test framework.
- E2E test framework.
- Visual regression test.
- Script tự động chặn file có nhiều component.
- Script tự động chặn text tiếng Anh hoặc raw enum trên UI.

### 2.2 File vi phạm quy tắc một component mỗi file

AST audit tìm thấy tối thiểu:

```text
12 file chứa nhiều hơn một component
51 component phụ phải được tách
```

Danh sách bắt buộc:

| File hiện tại | Số component | Component phát hiện |
|---|---:|---|
| `src/components/cashier/cashier-dashboard-page.tsx` | 15 | `CashierPill`, `SectionCard`, `EmptyState`, `CashierDashboardPage`, `MetricCard`, `OrdersListPanel`, `OrderDetailPanel`, `PaymentPanel`, `TablesPanel`, `CreateOrderPanel`, `HistoryPanel`, `ReportPanel`, `MixRow`, `LegendDot`, `SummaryRow` |
| `src/components/reports/report-dashboard-page.tsx` | 15 | `ReportDashboardPage`, `DateField`, `PeriodButton`, `ChartPanel`, `MetricBarChart`, `MetricBar`, `PaymentMethodChart`, `OrderStatusSummary`, `PeakHourLineChart`, `DonutChart`, `MetricRow`, `InsightCard`, `RankedList`, `RankedListRow`, `FixedTooltip` |
| `src/components/waiter/waiter-dashboard-page.tsx` | 12 | `Pill`, `Card`, `EmptyState`, `WaiterDashboardPage`, `OrdersView`, `OrderDetailSheetContent`, `TableMapView`, `CreateOrderView`, `MenuView`, `ProfileView`, `LegendDot`, `InfoRow` |
| `src/components/auth/portal-shell.tsx` | 4 | `PortalStatCard`, `PortalShell`, `PortalNavLink`, `ButtonIcon` |
| `src/components/me/my-menu-item-detail-page.tsx` | 3 | `DetailRow`, `MenuItemDetailImage`, `MyMenuItemDetailPage` |
| `src/components/me/my-branch-menu-page.tsx` | 2 | `MenuItemImage`, `MyBranchMenuPage` |
| `src/components/owner/tables/owner-table-detail-page.tsx` | 2 | `InfoRow`, `OwnerTableDetailPage` |
| `src/components/owner/orders/owner-branch-orders-page.tsx` | 2 | `FilterSelect`, `OwnerBranchOrdersPage` |
| `src/components/settings/branch-settings-page.tsx` | 2 | `BranchSettingsPage`, `VoucherMetric` |
| `src/components/me/my-table-detail-page.tsx` | 2 | `InfoRow`, `MyTableDetailPage` |
| `src/components/me/my-branch-detail-page.tsx` | 2 | `InfoRow`, `MyBranchDetailPage` |
| `src/components/me/waiter-mobile-shell.tsx` | 2 | `WaiterMobileShell`, `WaiterNavItem` |

Audit này phải được chạy lại sau mỗi phase. Con số trên là baseline, không phải danh sách được phép hard-code vĩnh viễn.

### 2.3 File chỉ có một component nhưng vẫn quá lớn hoặc trộn quá nhiều trách nhiệm

Các file sau có thể không vi phạm số component hiện tại, nhưng cần tách logic/section để đạt clean code:

| File | Vấn đề chính |
|---|---|
| `src/components/customer/session-menu-page.tsx` | Query menu, search/filter, cart, modal, responsive layout trong cùng page |
| `src/components/me/my-branch-kitchen-page.tsx` | SignalR refresh, chuông cảnh báo, pending orders, grouped queue, selection và mutation trong một component |
| `src/components/customer/session-order-page.tsx` | Realtime order, payment, timeline, item status, PayOS và action trong một component |
| `src/components/me/my-branch-tables-page.tsx` | Query/filter/open/close session/table card trong một component |
| `src/components/manage-menu/menu-item-list-page.tsx` | Query, filter, selection, reorder, bulk mutation và table UI |
| `src/components/manage-menu/category-list-page.tsx` | Query, filter, reorder, active mutation và table UI |
| `src/components/owner/tables/owner-table-list-page.tsx` | Query/filter/QR action/dialog/table UI |
| `src/components/organisms/manager-users/index.tsx` | Query/filter/form/mutation/dialog/stat trong một component |
| `src/components/owner/users/owner-users-page.tsx` | Query/filter/form schema/mutation/dialog state |
| `src/components/customer/session-checkout-page.tsx` | Shared cart, form, payload, mutation, redirect và order summary |
| `src/components/customer/session-cart-sheet.tsx` | Sheet UI, note update, quantity update, checkout guard |
| `src/components/settings/branch-settings-page.tsx` | Hai form độc lập, query branch, payment config, voucher list và mutation |

Không bắt buộc tách component chỉ để giảm số dòng. Chỉ tách khi có boundary rõ ràng, props rõ ràng và không làm thay đổi DOM/layout.

### 2.4 Audit tiếng Anh

Static audit ban đầu tìm thấy tối thiểu:

```text
55 file có chuỗi tiếng Anh khả nghi
355 chuỗi tiếng Anh khả nghi
```

Các nhóm có mật độ cao:

| Module | Ví dụ hiện tại |
|---|---|
| Auth | `Sign in`, `Forgot password?`, `Current user`, `Role` |
| Role | `Owner`, `Branch Manager`, `Staff`, `Kitchen`, `Cashier`, `Unknown role` |
| Menu management | `Create Menu Item`, `Category`, `Available`, `Featured`, `Price History` |
| Category management | `Create Category`, `Display Order`, `Loading categories...` |
| Owner table | `Tables & QR`, `Occupied`, `Current Session`, `QR Management` |
| Owner orders | `Orders & Invoices`, `Pending confirmation`, `No payment`, `View details` |
| Manager users | `Create user`, `Full name`, `Select branches`, `No users found` |
| Kitchen | `Kitchen Queue`, `Pending Confirmation`, `Confirm Selected`, `Alert` |
| Waiter | `Loading orders...`, `Loading tables...`, `Branch` |
| Cashier | `Refresh`, `Active order` |
| Toast/mutation | Nhiều success/error message tiếng Anh |
| Accessibility | `aria-label="Loading"` trong spinner |

### 2.5 Raw enum/backend value đang có thể lộ ra giao diện

Các pattern phải bị loại bỏ:

```tsx
{order.status}
{item.status}
{order.paymentStatus}
{order.paymentMethod}
map[value] ?? value
ROLE_LABELS[role] ?? role
```

Một số vị trí hiện tại:

- `src/components/me/my-table-detail-page.tsx`
- `src/components/waiter/waiter-dashboard-page.tsx`
- `src/components/reports/report-dashboard-page.tsx`
- `src/components/cashier/cashier-dashboard-page.tsx`
- `src/components/owner/tables/owner-table-detail-page.tsx`
- `src/components/owner/orders/owner-branch-orders-page.tsx`
- `src/constants/roleLabels.ts`

Fallback đúng:

```text
Không xác định
Trạng thái không xác định
Phương thức không xác định
Vai trò không xác định
```

Không fallback về raw backend value trên UI.

### 2.6 Backend error đang bị truyền thẳng ra UI

Hiện có nhiều helper/mutation trả trực tiếp:

```ts
error.response?.data?.message
error.response?.data?.detail
error.response?.data?.title
error.message
```

Điều này có thể làm tiếng Anh từ backend xuất hiện trên frontend.

Các vùng cần sửa:

- Auth login.
- Owner restaurant/branch/user/table.
- Manager users.
- Manage menu/category.
- Me menu/table.
- Customer session.
- Cashier.
- Branch settings.
- Order mutations.

Yêu cầu sau migration:

- UI chỉ nhận message tiếng Việt.
- Raw backend error chỉ được log ở development/observability.
- Ưu tiên map theo `code` nếu API trả code.
- Nếu chưa có code, normalize và map known backend message.
- Unknown backend error dùng fallback tiếng Việt theo action.

### 2.7 Locale tiếng Anh đang dùng cho dữ liệu hiển thị

Các vị trí cần audit:

- `src/components/manage-menu/helpers.tsx`: `en-US`
- `src/components/organisms/manager-users/helpers.ts`: `en-US`
- `src/components/reports/report-dashboard-page.tsx`: `en-CA`

Quy tắc:

- Dữ liệu hiển thị dùng `vi-VN`.
- `en-CA` chỉ được giữ nếu dùng nội bộ để tạo `YYYY-MM-DD`, không render ra UI, và phải có comment giải thích. Tốt hơn là dùng formatter kỹ thuật không phụ thuộc locale.

## 3. Kiến trúc đích

### 3.1 Cấu trúc component

Ví dụ cho cashier:

```text
src/components/cashier/dashboard/
├── cashier-dashboard-page.tsx
├── cashier-pill.tsx
├── cashier-section-card.tsx
├── cashier-empty-state.tsx
├── cashier-metric-card.tsx
├── cashier-orders-list-panel.tsx
├── cashier-order-detail-panel.tsx
├── cashier-payment-panel.tsx
├── cashier-tables-panel.tsx
├── cashier-create-order-panel.tsx
├── cashier-history-panel.tsx
├── cashier-report-panel.tsx
├── cashier-mix-row.tsx
├── cashier-legend-dot.tsx
└── cashier-summary-row.tsx
```

Logic:

```text
src/hooks/cashier/
├── use-cashier-dashboard.ts
├── use-cashier-manual-order.ts
└── use-cashier-payment.ts

src/helpers/cashier/
├── cashier-dashboard.helpers.ts
├── cashier-payment.helpers.ts
├── cashier-receipt.helpers.ts
└── cashier-dashboard.types.ts
```

Không bắt buộc giữ đúng tên folder trên nếu code thực tế cho thấy boundary khác tốt hơn. Bắt buộc giữ:

- Một component/file.
- Không tạo circular import.
- Không tạo barrel file gây import cycle.
- Shared component chỉ được tạo khi markup và behavior thật sự dùng chung.

### 3.2 Cấu trúc waiter

```text
src/components/waiter/dashboard/
├── waiter-dashboard-page.tsx
├── waiter-pill.tsx
├── waiter-card.tsx
├── waiter-empty-state.tsx
├── waiter-orders-view.tsx
├── waiter-order-detail-sheet-content.tsx
├── waiter-table-map-view.tsx
├── waiter-create-order-view.tsx
├── waiter-menu-view.tsx
├── waiter-profile-view.tsx
├── waiter-legend-dot.tsx
└── waiter-info-row.tsx

src/hooks/waiter/
├── use-waiter-dashboard.ts
└── use-waiter-manual-order.ts

src/helpers/waiter/
├── waiter-order.helpers.ts
├── waiter-table.helpers.ts
└── waiter-dashboard.types.ts
```

### 3.3 Cấu trúc reports

```text
src/components/reports/dashboard/
├── report-dashboard-page.tsx
├── report-date-field.tsx
├── report-period-button.tsx
├── report-chart-panel.tsx
├── report-metric-bar-chart.tsx
├── report-metric-bar.tsx
├── report-payment-method-chart.tsx
├── report-order-status-summary.tsx
├── report-peak-hour-line-chart.tsx
├── report-donut-chart.tsx
├── report-metric-row.tsx
├── report-insight-card.tsx
├── report-ranked-list.tsx
├── report-ranked-list-row.tsx
└── report-fixed-tooltip.tsx

src/hooks/reports/
└── use-report-dashboard.ts

src/helpers/reports/
├── report-date.helpers.ts
├── report-series.helpers.ts
├── report-export.helpers.ts
└── report-dashboard.types.ts
```

Excel formatting và data transformation phải ở helper, không nằm trong component.

### 3.4 Portal shell

```text
src/components/auth/portal-shell/
├── portal-shell.tsx
├── portal-stat-card.tsx
├── portal-nav-link.tsx
└── portal-icon-button.tsx
```

Phải giữ compatibility import trong một commit riêng hoặc cập nhật toàn bộ import cùng lúc.

Không dùng `index.tsx` chứa component. Nếu cần barrel:

```text
index.ts
```

và file đó chỉ re-export.

### 3.5 Các file vi phạm nhỏ

Tách theo bảng:

| Component phụ | File đích gợi ý |
|---|---|
| `MenuItemImage` | `src/components/me/menu-item-image.tsx` |
| `DetailRow` | `src/components/me/menu-item-detail-row.tsx` |
| `InfoRow` của table | `src/components/me/table-info-row.tsx` hoặc domain-specific file riêng |
| `WaiterNavItem` | `src/components/me/waiter-nav-item.tsx` |
| `FilterSelect` | `src/components/owner/orders/order-filter-select.tsx` |
| `VoucherMetric` | `src/components/settings/voucher-metric.tsx` |
| Owner table `InfoRow` | `src/components/owner/tables/owner-table-info-row.tsx` |

Không gộp các `InfoRow`, `EmptyState`, `Card` chỉ vì trùng tên. Trước khi dùng chung phải chứng minh:

- Cùng semantic.
- Cùng markup.
- Cùng responsive classes.
- Cùng prop contract.
- Không cần conditional style theo domain.

Nếu không chắc, giữ component domain-specific để tránh đổi layout.

### 3.6 Presentation mapper tập trung

Tạo layer mapping không phụ thuộc component:

```text
src/helpers/presentation/
├── role-labels.ts
├── order-status-labels.ts
├── order-item-status-labels.ts
├── payment-status-labels.ts
├── payment-method-labels.ts
├── table-status-labels.ts
├── discount-type-labels.ts
├── order-source-labels.ts
├── user-status-labels.ts
└── api-error-labels.ts
```

Có thể giữ `src/constants/roleLabels.ts` để giảm churn, nhưng nội dung phải dùng tiếng Việt và tất cả mapper khác phải theo cùng pattern.

API đề xuất:

```ts
getRoleLabel(role)
getOrderStatusLabel(status)
getOrderItemStatusLabel(status)
getPaymentStatusLabel(status)
getPaymentMethodLabel(method)
getTableStatusLabel(status)
getDiscountTypeLabel(type)
getOrderSourceLabel(source)
getUserStatusLabel(value)
```

Mỗi mapper phải:

- Nhận `string | null | undefined` nếu data backend có thể không chặt.
- Map đủ known enum.
- Trả fallback tiếng Việt.
- Không trả raw input.
- Có unit test cho toàn bộ known values và unknown value.

Role mapping chuẩn đề xuất:

```text
ADMIN          -> Quản trị viên hệ thống
OWNER          -> Chủ nhà hàng
MANAGER        -> Quản lý
BRANCH_MANAGER -> Quản lý chi nhánh
STAFF          -> Nhân viên phục vụ
KITCHEN        -> Nhân viên bếp
CASHIER        -> Thu ngân
unknown        -> Vai trò không xác định
```

### 3.7 Error mapper tiếng Việt

Tạo một entry point:

```ts
getVietnameseApiErrorMessage(error, fallback)
```

Thứ tự:

1. Map `code` backend nếu có.
2. Map validation errors theo field.
3. Map known normalized English message.
4. Map theo HTTP status khi phù hợp.
5. Trả fallback tiếng Việt của action.

Không làm:

```ts
return response.data.message ?? error.message;
```

Có thể log raw error:

```ts
Log.error("owner-user-update", error);
```

nhưng không render raw message.

## 4. Quy tắc bảo toàn layout

### 4.1 Hai baseline tách biệt

Yêu cầu “tách component nhưng layout giống hệt” và yêu cầu “đổi toàn bộ text sang tiếng Việt” phải kiểm thử theo hai phase:

#### Baseline A: structural refactor

- Chụp screenshot trước khi tách component.
- Tách component/hook/helper.
- Không đổi bất kỳ text nào trong phase này.
- Screenshot sau refactor phải pixel-identical.
- `maxDiffPixels = 0`.
- `threshold = 0`.

#### Baseline B: localization

Đổi tiếng Anh sang tiếng Việt chắc chắn làm pixel chữ thay đổi, nên không thể yêu cầu ảnh byte-identical với baseline tiếng Anh.

Trong phase localization:

- Không đổi `className`.
- Không đổi CSS.
- Không thêm/bớt wrapper.
- Không đổi thứ tự DOM.
- Không đổi breakpoint.
- Không đổi icon.
- Chỉ thay copy và mapper.
- Screenshot diff phải được review và approve vì chỉ có text thay đổi.
- Phải có geometry/overflow assertions để chứng minh layout không vỡ.

### 4.2 Những thứ cấm thay đổi khi tách

- `className` string.
- Thứ tự class nếu snapshot/tool hiện tại nhạy cảm.
- DOM tag (`div`, `section`, `main`, `button`, `a`).
- Wrapper element.
- `key`.
- Portal target.
- `position: fixed/sticky/absolute`.
- `z-index`.
- Grid/flex structure.
- Responsive breakpoint.
- Event timing.
- Form submit order.
- Query enabled condition.
- Mutation payload.
- Reset timing.
- Redirect URL.
- Local storage key.
- SignalR join/leave timing.

### 4.3 Quy tắc move JSX

Khi tách component:

1. Copy nguyên JSX block.
2. Xác định props từ biến free-variable.
3. Không rename CSS class trong cùng commit.
4. Không “cleanup” copy hoặc style trong commit structural.
5. Không thêm fragment/wrapper nếu không cần.
6. Event handler có business logic giữ ở hook/container; child nhận callback.
7. Derived presentation data có thể tính ở helper, nhưng output phải test bằng equality.

### 4.4 Layout geometry assertions

Với mỗi visual route, lưu `getBoundingClientRect()` cho các landmark:

- Header.
- Sidebar/mobile nav.
- Main content.
- Primary action.
- Table/list container.
- Dialog/sheet.
- Sticky footer/cart bar.

Assertions:

- Không có horizontal overflow:

```ts
document.documentElement.scrollWidth <= document.documentElement.clientWidth
```

- Button text không bị clip.
- Dialog nằm trong viewport.
- Sticky/fixed element không che CTA.
- Sidebar width và content offset không đổi trong phase structural.
- Card grid column count không đổi theo viewport.

## 5. Test tooling phải thêm trước migration

### 5.1 Dependency đề xuất

Unit/component:

```text
vitest
jsdom
@testing-library/react
@testing-library/user-event
@testing-library/jest-dom
vite-tsconfig-paths
msw
```

E2E/visual:

```text
@playwright/test
```

Accessibility tùy chọn nhưng khuyến nghị:

```text
@axe-core/playwright
```

### 5.2 Script bắt buộc trong `package.json`

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "playwright test --grep-invert @visual",
    "test:visual": "playwright test --grep @visual",
    "test:all": "pnpm lint && pnpm test && pnpm test:e2e && pnpm test:visual && pnpm build",
    "audit:components": "node scripts/audit-single-component.mjs",
    "audit:vi": "node scripts/audit-vietnamese-ui.mjs"
  }
}
```

Tên script có thể điều chỉnh theo convention repo, nhưng capability phải đầy đủ.

### 5.3 Guard script một component mỗi file

Tạo:

```text
scripts/audit-single-component.mjs
```

Script dùng TypeScript AST:

- Scan `src/**/*.tsx`.
- Bỏ qua `src/components/ui/**`.
- Phát hiện top-level:
  - Function component.
  - Arrow component.
  - `memo`.
  - `forwardRef`.
- Fail nếu một file có hơn một component.
- In file, line và component names.
- Exit code khác 0.

Không dùng regex thuần vì dễ bỏ sót `forwardRef`, `memo` và multiline declaration.

### 5.4 Guard script tiếng Việt

Tạo:

```text
scripts/audit-vietnamese-ui.mjs
```

Script AST scan:

- JSX text.
- `placeholder`.
- `title`.
- `aria-label`.
- `alt`.
- Object property thường dùng cho UI:
  - `label`
  - `description`
  - `message`
  - `helper`
  - `caption`
  - `heading`
- Toast/notification message.
- Validation message.

Script có allowlist:

```text
ScanNow
PayOS
QR
VAT
API
URL
Email
```

Script phải fail khi:

- Có candidate tiếng Anh ngoài allowlist.
- JSX render raw `.status`, `.role`, `.paymentMethod`, `.paymentStatus`.
- Có fallback mapper dạng `map[value] ?? value`.
- Có `Intl.DateTimeFormat("en-US")` cho output UI.
- Có raw backend message pass-through vào notify/toast/component.

False positive phải được giải quyết bằng allowlist có lý do, không tắt rule cho cả file.

## 6. Test matrix sau migration

### 6.1 Static verification

Chạy sau từng phase:

```bash
pnpm audit:components
pnpm audit:vi
pnpm lint
pnpm build
```

Expected:

- Không file nào ngoài shadcn có hơn một component.
- Không English UI candidate ngoài allowlist.
- ESLint 0 error, 0 warning.
- TypeScript pass.
- Production build pass.

### 6.2 Unit test cho mapper

Test bắt buộc:

#### Role

- Tất cả role known trả tiếng Việt.
- `MANAGER` legacy vẫn map.
- `null`, `undefined`, empty, unknown không trả raw value.

#### Order status

Test:

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
unknown
```

#### Order item status

Test:

```text
Pending
Confirmed
Cooking
Ready
Served
Cancelled
unknown
```

#### Payment

Method:

```text
CASH
PAYOS
unknown
```

Status:

```text
NO_PAYMENT
PENDING
SUCCESS
FAILED
REFUNDED
unknown
```

#### Table

Test cả string và numeric legacy:

```text
AVAILABLE / 0
OCCUPIED / 1
RESERVED / 2
DISABLED / 3
unknown
```

#### Voucher/discount

```text
PERCENT
FIXED_AMOUNT
unknown
```

### 6.3 Unit test helper logic

Các helper được move khỏi component phải test trước/sau:

- Cashier payment label.
- Cashier cash change.
- Cashier report totals.
- Waiter order filter grouping.
- Table state.
- Report date preset.
- Revenue aggregation by day/week/month.
- Payment method aggregation.
- Voucher payload date start/end.
- Voucher discount formatting.
- Pagination visible pages.
- Owner table form payload.
- Menu/category reorder payload.
- Customer order timeline index.
- Receipt data generation.
- Excel report data transformation.

Mục tiêu coverage:

- 100% branches cho mapper enum.
- 90% statements/branches cho helper nghiệp vụ được tách.
- Không ép coverage toàn repository ngay lập tức nếu phần legacy chưa có test, nhưng changed files phải đạt threshold.

### 6.4 Component interaction test

React Testing Library + MSW:

- Render component đã tách với props baseline.
- Kiểm tra text/role/aria.
- Click action gọi callback đúng một lần.
- Disabled/loading state giữ nguyên.
- Dialog open/close giữ nguyên.
- Form field vẫn kết nối React Hook Form.
- Mobile/desktop control cùng dùng một source of truth.
- Không remount làm mất input/selection không mong muốn.

Các component ưu tiên:

- Cashier payment panel.
- Cashier create order panel.
- Waiter create order view.
- Waiter order detail.
- Report period/date controls.
- Branch settings payment form.
- Voucher form.
- Owner table status section.
- Owner order filters.
- Kitchen pending/group selection.
- Customer cart sheet.
- Customer checkout summary.

### 6.5 Contract/payload test

MSW hoặc request spy phải xác nhận payload không đổi:

- Login.
- Owner restaurant update.
- Owner branch create/update.
- Owner/manager user create/update.
- Category create/update/reorder.
- Menu item create/update/reorder/bulk availability.
- Price update.
- Table create/update/status/active/regenerate QR.
- Payment config.
- Voucher create/update.
- Customer place order.
- Customer checkout.
- Waiter create/confirm/serve.
- Kitchen confirm/ready.
- Cashier checkout CASH/PAYOS.
- Cancel pending payment.

So sánh field-for-field, không chỉ kiểm tra endpoint được gọi.

### 6.6 Visual regression test

Playwright phải dùng:

- Cùng Chromium version.
- Cùng OS/container trong CI.
- Font đã load xong:

```ts
await page.evaluate(() => document.fonts.ready);
```

- Disable animation/caret.
- Fixed timezone: `Asia/Bangkok` hoặc `Asia/Ho_Chi_Minh`, chọn một và dùng thống nhất.
- Fixed clock.
- Mock deterministic API.
- Mock stable image URL/local asset.
- Không phụ thuộc production data.

Viewports tối thiểu:

```text
mobile: 375 x 812
tablet: 768 x 1024
desktop: 1440 x 900
```

### 6.7 Visual route/state matrix

#### Public customer

- Login.
- QR loading.
- QR invalid.
- QR table not open.
- Menu populated.
- Menu empty.
- Cart open.
- Menu item detail.
- Checkout with items.
- Checkout empty/error.
- Order pending.
- Order preparing.
- Order ready.
- Order completed.
- Payment return success/pending/failed.
- Payment cancel.

#### Owner

- Dashboard report.
- Restaurant detail/edit.
- Branch list.
- Branch create/edit.
- User list.
- User create/edit dialog.
- Category list/form.
- Menu item list/form.
- Price history/dialog.
- Table list.
- Table detail with no session.
- Table detail with session/history.
- Branch orders list/detail.
- Settings payment form.
- Settings voucher form/list.

#### Manager

- Dashboard.
- Users.
- Assigned branch menu/table/order/settings.
- Empty/no assigned branch.
- Forbidden state.

#### Staff/waiter

- Orders.
- Order detail sheet.
- Table map.
- Create order.
- Menu.
- Profile.

#### Kitchen

- Pending orders.
- High priority alert.
- Empty pending.
- Grouped queue.
- Selected items.

#### Cashier

- Orders list.
- Order detail.
- Cash payment.
- PayOS payment.
- Tables.
- Create manual order.
- History.
- Report.

### 6.8 Functional E2E

Không chỉ screenshot. Test flow:

1. Login từng role và redirect đúng.
2. Owner mở dialog user, validate, submit payload.
3. Manager filter và cập nhật user.
4. Owner/manager quản lý category/menu.
5. Staff mở bàn.
6. Customer scan QR, thêm món, đặt đơn.
7. Kitchen xác nhận và mark ready.
8. Waiter mark served.
9. Cashier thanh toán.
10. Order/status text hiển thị tiếng Việt ở mọi bước.
11. Unknown enum fixture không lộ raw enum.
12. Backend trả English error fixture nhưng UI hiển thị fallback tiếng Việt.

### 6.9 Accessibility và overflow

Test:

- `aria-label` tiếng Việt, trừ brand/technical allowlist.
- Form label gắn đúng input.
- Dialog có title/description.
- Icon-only button có accessible name tiếng Việt.
- Không horizontal overflow ở ba viewport.
- Focus order không đổi sau khi tách.
- Keyboard interaction của dropdown/dialog/sheet không đổi.

## 7. Trình tự triển khai

### Phase 0: Baseline và test harness

1. Tạo branch riêng.
2. Chạy:

```bash
pnpm lint
pnpm build
```

3. Cài Vitest, RTL, MSW, Playwright.
4. Tạo deterministic fixtures.
5. Chụp Baseline A cho các route/state P0.
6. Tạo geometry snapshot.
7. Không refactor trước khi baseline test chạy được.

Exit criteria:

- Test harness chạy trên local.
- Baseline screenshots được tạo từ code trước refactor.
- Có test ít nhất cho role/status mapper hiện tại để lock behavior kỹ thuật.

### Phase 1: Guard rails

1. Thêm `audit-single-component.mjs`.
2. Thêm `audit-vietnamese-ui.mjs`.
3. Baseline audit được phép fail và lưu report.
4. CI chạy hai script ở chế độ blocking sau khi migration hoàn tất.

Exit criteria:

- Script tìm được đúng 12 file vi phạm baseline.
- Script in component names và line.
- Script language có allowlist rõ ràng.

### Phase 2: Tách shell và component nhỏ

Thứ tự:

1. `portal-shell.tsx`.
2. `waiter-mobile-shell.tsx`.
3. Các `InfoRow`.
4. `MenuItemImage`.
5. `FilterSelect`.
6. `VoucherMetric`.

Quy tắc:

- Không đổi text.
- Không đổi CSS.
- Chạy exact screenshot.

Exit criteria:

- Các file trên còn một component.
- Visual diff bằng 0.

### Phase 3: Tách report dashboard

1. Move pure date/series/export function sang helper.
2. Tách 14 chart/UI component.
3. Tạo `use-report-dashboard`.
4. Giữ exact chart SVG/DOM/classes.
5. Unit test aggregation và export rows.

Exit criteria:

- `report-dashboard-page.tsx` chỉ còn page orchestration.
- Screenshot desktop/tablet/mobile diff bằng 0.
- Excel output fields không đổi.

### Phase 4: Tách waiter và cashier

Thực hiện từng module, không làm song song trong cùng commit nếu gây review quá lớn.

Waiter:

- Tách view component.
- Tách manual order hook.
- Tách mapper/helper.
- Test open table trước create order.

Cashier:

- Tách panel component.
- Tách payment hook.
- Tách manual order hook.
- Tách receipt/report helper.
- Test CASH/PAYOS/voucher/cancel.

Exit criteria:

- Mỗi file một component.
- Request payload giống baseline.
- Screenshot diff bằng 0.
- Không mất state khi đổi tab/view.

### Phase 5: Tách owner/me/settings

1. Owner order filter.
2. Owner table info.
3. Branch settings voucher metric.
4. Me branch/menu/table details.
5. Kitchen workflow.
6. Các large single-component page có boundary rõ ràng.

Exit criteria:

- Component audit pass cho các module đã làm.
- Selection/bulk action vẫn giữ đúng IDs.
- Query/mutation invalidation không đổi.

### Phase 6: Chuẩn hóa mapper tiếng Việt

1. Role mapper.
2. Order status.
3. Order item status.
4. Payment method/status.
5. Table status.
6. Discount type.
7. Order source.
8. User status.
9. Error mapper.

Trong phase này chưa đổi toàn bộ static copy, chỉ thay nơi raw backend data được render.

Exit criteria:

- Không raw enum trong JSX.
- Unknown values dùng fallback tiếng Việt.
- Unit test mapper 100% branch.

### Phase 7: Việt hóa static copy theo module

Thứ tự:

1. Auth và shell.
2. Owner.
3. Manager.
4. Menu/category/table.
5. Me/waiter.
6. Kitchen.
7. Cashier.
8. Customer public.
9. Toast/validation/error.
10. Accessibility label.
11. Receipt/Excel.

Mỗi module:

- Chạy `audit:vi`.
- Chạy component test.
- Chạy visual snapshots.
- Review overflow.

Exit criteria:

- Không còn candidate tiếng Anh ngoài allowlist.
- Không backend message tiếng Anh xuất hiện trong fixture tests.

### Phase 8: Final regression

Chạy:

```bash
pnpm audit:components
pnpm audit:vi
pnpm lint
pnpm test
pnpm test:coverage
pnpm test:e2e
pnpm test:visual
pnpm build
```

Manual smoke với backend thật:

- Owner.
- Branch manager.
- Staff.
- Kitchen.
- Cashier.
- Customer QR.

## 8. Chiến lược commit/PR

Không gom toàn bộ vào một commit.

Đề xuất:

1. `test: add component and visual regression harness`
2. `chore: add component and vietnamese ui audits`
3. `refactor(auth): split portal shell components`
4. `refactor(reports): split report dashboard components and helpers`
5. `refactor(waiter): split dashboard views and workflow hooks`
6. `refactor(cashier): split dashboard panels and workflow hooks`
7. `refactor(owner): split order table and settings components`
8. `refactor(me): split branch operation components`
9. `refactor(presentation): centralize backend value mappers`
10. `feat(i18n): translate auth owner and manager surfaces`
11. `feat(i18n): translate menu table and operations surfaces`
12. `feat(i18n): translate customer cashier toast and errors`
13. `test: update approved vietnamese visual snapshots`

Mỗi structural commit phải:

- Không đổi copy.
- Không đổi class.
- Visual diff bằng 0.

Mỗi localization commit phải:

- Không refactor logic.
- Không đổi class/DOM.
- Chỉ đổi copy/mapping/test snapshots.

## 9. Rủi ro và cách kiểm soát

### 9.1 Prop drilling sau khi tách

Không giải quyết bằng context mới ngay lập tức.

Ưu tiên:

- Container hook.
- Props typed rõ.
- Context chỉ khi có nhiều tầng thật sự và state cùng lifecycle.

### 9.2 Component dùng chung quá sớm

Rủi ro làm layout các role bị đồng nhất sai.

Kiểm soát:

- Tách domain-specific trước.
- Chỉ merge shared component sau khi visual test chứng minh tương đương.

### 9.3 Hook khổng lồ thay cho component khổng lồ

Không chuyển toàn bộ 500 dòng từ component sang một hook.

Tách theo workflow:

- Query/read model.
- Selection.
- Form/manual order.
- Payment.
- Realtime.

### 9.4 Translation làm tràn layout

Kiểm soát:

- Chọn copy tiếng Việt ngắn, rõ nghĩa.
- Test mobile 375px.
- Check button/dialog/table overflow.
- Không tự ý giảm font hoặc spacing để “chữa” nếu chưa review design.
- Nếu copy bắt buộc dài, ghi nhận intentional layout adjustment riêng, không trộn vào structural refactor.

### 9.5 Backend thêm enum mới

Mapper unknown không được render raw.

Test fixture unknown value phải luôn tồn tại để bảo đảm fallback tiếng Việt.

### 9.6 Backend error không có code

Map known message chỉ là giải pháp chuyển tiếp.

Khuyến nghị backend tương lai trả error code ổn định. FE vẫn phải có fallback tiếng Việt.

## 10. Definition of Done

Plan chỉ được đánh dấu done khi toàn bộ điều kiện sau đạt:

### Cấu trúc

- `pnpm audit:components` pass.
- Không file `.tsx` ngoài `src/components/ui/**` chứa hơn một component.
- Route page chỉ làm route composition.
- Không có pure helper lớn còn nằm trong component dashboard.
- Hook được tách theo workflow, không tạo god hook.

### Ngôn ngữ

- `pnpm audit:vi` pass.
- Role hiển thị hoàn toàn tiếng Việt.
- Tất cả backend enum có mapper tiếng Việt.
- Unknown enum không render raw value.
- Backend English message không truyền thẳng ra UI.
- Toast, validation, empty/error/loading state là tiếng Việt.
- Accessible label là tiếng Việt ngoài allowlist.
- Receipt và Excel export dùng tiếng Việt.

### Logic

- Payload tests pass.
- Query enabled condition không đổi.
- Mutation/reset/redirect/cache invalidation không đổi.
- SignalR join/leave/reconnect không đổi.
- Cart/localStorage behavior không đổi.
- Role guard/redirect không đổi.

### Layout

- Structural phase screenshot diff bằng 0.
- Localization screenshots đã review.
- Không horizontal overflow ở 375, 768, 1440.
- Không CTA bị che/cắt.
- Không dialog/sheet vượt viewport.
- DOM geometry critical landmarks đạt assertions.

### Quality gate

```bash
pnpm audit:components
pnpm audit:vi
pnpm lint
pnpm test
pnpm test:e2e
pnpm test:visual
pnpm build
```

Tất cả phải pass trước khi merge.

## 11. Deliverables cuối cùng

Code:

- Component tree đã tách.
- Hook/helper architecture sạch.
- Central presentation mapper.
- Central Vietnamese API error mapper.
- Toàn bộ UI tiếng Việt.

Tooling:

- Component audit.
- Vietnamese UI audit.
- Vitest/RTL/MSW.
- Playwright E2E.
- Playwright visual snapshots.
- Geometry/overflow assertions.

Tài liệu:

- Plan này.
- Báo cáo baseline.
- Danh sách file đã migrate.
- Visual snapshot approval record.
- Blocked scenarios nếu không có credential/backend data.
- File `plan-06-single-component-vietnamese-ui-migration-done.md` sau khi hoàn tất, ghi rõ test command và kết quả thực tế.
