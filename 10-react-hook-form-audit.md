# React Hook Form Audit

Cap nhat: 2026-06-13

Pham vi audit:

- Landing/Admin FE: `/home/nhat/fpt/scan-now-nextjs`, branch `main`, commit `7d59f7c`.
- Tenant/Portal FE: `/home/nhat/fpt/scan-now-customer`, branch `main`, commit `2fae31f`.
- Product docs: `/home/nhat/fpt/scannow-product-docs`, branch `main`, commit `dda3845`.
- Yeu cau: audit tat ca user-editable controls (`input`, `textarea`, `select`, Radix `Select`, dropdown radio/checkbox, checkbox, popover filter, form-like button flow) trong 2 FE repo.
- Quy tac required: chi danh dau required dua tren validation/guard frontend hien tai. Khong suy luan them tu backend contract.

## 1. Baseline verification

Static checks da chay truoc migration:

| Repo | Command | Result |
| --- | --- | --- |
| `scan-now-nextjs` | `pnpm lint` | Pass: ESLint 0 warnings/errors, `tsc --noEmit` pass. |
| `scan-now-customer` | `pnpm lint` | Pass: ESLint 0 warnings/errors, `tsc --noEmit` pass. |

Browser smoke baseline:

- Chua chay duoc browser smoke trong moi truong hien tai vi khong co browser automation tool hoac browser binary kha dung.
- `pnpm exec playwright --version` trong `scan-now-customer` fail voi `Command "playwright" not found`.
- `chromium`, `chromium-browser`, `google-chrome`, `playwright` khong co trong `PATH`.
- Khong co bo credential/test account duoc xac nhan trong repo de smoke cac route auth-protected.
- Khi bat dau migration code, can chay lai matrix trong `plans/plan-05-react-hook-form-migration.md` tren local/prod-like browser co credential that. Cac scenario auth bi block phai duoc ghi ro vao ket qua test sau migration.

## 2. Quy uoc audit

- `RHF`: da dung `react-hook-form`.
- `Manual state`: dang dung `useState`/props callback/guard thu cong.
- `Control-only`: co user input/filter/select/checkbox nhung khong submit mutation.
- `Action dialog`: dialog co nut action nhung khong co input. Audit de khong bi bo sot, nhung khong can RHF neu khong co field.
- Required `*` sau migration la visual-only, khong tu dong them validation moi.

## 3. Required fields summary

Required theo frontend hien tai:

- Admin platform: admin login `identifier`, `password`; owner create/edit `fullName`, `username`, `email`, create-only `password`; restaurant create/edit `name`, `slug`, create-only owner select `ownerId`; lead capture `phone`, `email`, `location`.
- Tenant auth/customer/ops: tenant login `identifier`, `password`; waiter/cashier create order selected table; cashier cash amount.
- Owner/manager back office: owner restaurant `name`, `slug`; owner branch `name`, `slug`; owner/manager users `fullName`, `username`, `email`, `role`, `branchIds`, create-only `password`; table `tableNumber`, `capacity`; category `name`; menu item `categoryId`, `name`; update price `price`; branch settings selected `branchId`.

Explicit non-required:

- Customer contact fields: `customerName`, `customerPhone`, `customerNote`.
- Customer/menu item notes and special request textareas.
- All search/filter/sort/date controls unless explicitly guarded as required.
- Optional URLs: restaurant logo URL, category/menu item image URL.
- Optional phone fields unless schema says otherwise.
- Payment config/voucher fields in branch settings, because current frontend does not block submit based on those values.

## 4. `scan-now-nextjs` inventory

| Area / component | Fields / controls | Current state | Required fields | Current validation / messages | Submit / action behavior | Reset / side effects | Migration notes |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `src/components/admin/admin-login-card.tsx` / `AdminLoginCard` | `identifier`, `password` | RHF + `zodResolver` | `identifier`, `password` | `Email or username is required`; `Password is required` | `loginAdmin(values)`, store JWT in `localStorage`, update user store, redirect `window.location.href = "/admin"` | Clears local `errorMessage` before submit; sets `isSubmitting`; on catch shows fixed admin credential error | Keep schema and submit logic. Replace labels with reusable required label only. |
| `src/components/admin/create-owner-dialog.tsx` / `CreateOwnerDialog` | `fullName`, `username`, `email`, `phoneNumber`, create-only `password` | RHF + dynamic Zod schema | Create: `fullName`, `username`, `email`, `password`. Edit: `fullName`, `username`, `email` | Email required/format; phone optional regex; create password min 6 + lower/upper/non-alphanumeric | Create calls `onCreate`; edit calls `onUpdate(owner.userId, payload)` with `phoneNumber || null` | `reset` on open using owner data; close resets and calls `onClose` | Keep dynamic schema. Add required labels according to mode. |
| `src/components/admin/restaurant-form.tsx` / `RestaurantForm` | create-only owner `Select`, `name`, `slug`, `logoUrl`, `description` | RHF + Zod; Radix `Select` controlled by `setValue/watch` | Create: `ownerId`, `name`, `slug`. Edit: `name`, `slug` | owner/name/slug required; `logoUrl` optional `http://`/`https://` URL | `onSave(values)` from page; cancel calls `onCancel` | Reset from `restaurant`; new create resets to empty; create mode auto-generates slug from name | Keep `generateSlug` side effect and edit/create differences. Add required labels. |
| `src/components/molecules/lead-capture-modal.tsx` / `LeadCaptureModal` | `phone`, `email`, `location` | RHF + Zod + `Controller` for `Select` | `phone`, `email`, `location` | i18n validation: phone required/invalid, email required/invalid, location required | Calls prop `onSubmit(values)`; parent sends `/api/lead-capture`, success/error toast, closes modal | Reset on close and after successful submit | Keep i18n schema and reset timing. Add required label to native `label` or switch to shared `Label`. |
| `src/components/organisms/sales-landing-actions.tsx` | Lead modal open source, pricing/chat buttons | Manual state around modal | None | None | Opens modal with source; WhatsApp/contact link actions; modal submit sends lead email | `isSubmitting`, `leadSource`, modal open state | No RHF needed except modal itself. Must preserve lead source in payload. |
| `src/components/molecules/globals/language-switcher.tsx` | Locale Radix `Select` (`vi`, `en`) | Manual derived state from `useLocale` | None | Guards unsupported/same locale | Rewrites pathname locale segment, preserves query/hash, `router.replace` | No reset; route side effect only | Control-only. If migrating all controls, use RHF watch/Controller but keep identical route rewrite. |
| `src/components/admin/restaurant-filters.tsx` | Search input, status select | Controlled props | None | None | `onSearchChange`, `onStatusChange` update parent query | Parent controls pagination/query | Filter-only. Preserve immediate change behavior. |
| `src/components/admin/owner-filters.tsx` | Search input, status select, page-size select | Controlled props | None | None | callbacks update parent query/page size | Parent controls pagination/query | Filter-only. Preserve page-size numeric conversion. |
| `src/components/admin/branch-tables-tab.tsx` | Search, status select, active view buttons, refresh button | Manual local `useState` | None | None | Search/status pass to `BranchTablesGrid`; refresh invalidates table/session queries | Search/status reset `pageNumber` to 1; refresh invalidates React Query cache | Filter-only. Preserve page reset and invalidate behavior. |
| `src/components/admin/branch-categories-tab.tsx` | Search, active select, sort select, sort direction button | Manual local `useState` | None | None | Builds `CategoryListParams` for query; view button routes to detail | Search/active reset page; sort direction toggle changes query | Filter-only. Preserve default `name:asc` semantics where default params are omitted. |
| `src/components/admin/branch-menu-items-tab.tsx` | Search, active/available/featured/category/sort selects, sort direction button | Manual local `useState` | None | None | Builds menu-item query params; routes to item detail | Most filters reset page; sort defaults omitted | Filter-only. Preserve category query dependency. |
| `src/components/admin/restaurant-branches-tab.tsx` | Search input | Manual local `useState` | None | None | Builds branch list query; routes to branch supervision | Search resets page | Filter-only. Preserve page reset. |

## 5. `scan-now-customer` inventory: auth and public customer

| Area / component | Fields / controls | Current state | Required fields | Current validation / messages | Submit / action behavior | Reset / side effects | Migration notes |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `src/components/auth/login-form.tsx` / `LoginForm` | `identifier`, `password`, `rememberMe`, show password toggle | Manual state | `identifier`, `password` via disabled submit guard | No inline required message; API errors mapped by `mapLoginErrorMessage` | `useLoginMutation`, payload `{ identifier: identifier.trim(), password }`, route via `getLoginRedirectPath` | Redirects away if already authenticated; `rememberMe` currently not submitted | RHF must preserve disabled behavior and server error placement. Do not add new inline required messages unless product accepts UX change. |
| `src/components/customer/menu/place-order-modal.tsx` / `PlaceOrderModal` | `customerName`, `customerPhone`, `customerNote` | RHF + Zod + `Controller` | None | Optional name max 100; optional phone regex or empty; optional note max 300 | Calls parent `onSubmit` with empty string fallbacks | Component returns `null` when closed; no explicit reset on close in current code | Already migrated. Only normalize required-label support; no required star. |
| `src/components/customer/session-menu-item-page.tsx` | `note` textarea, local quantity stepper | Manual state | None | Quantity min 1 by UI; note optional | `handleSaveToCart` updates shared cart line with `specialRequest: note.trim() || null` and `quantity: localQuantity` | Syncs note/localQuantity from cart line; success toast | RHF can own `note` and `localQuantity`, but cart update remains explicit on save. |
| `src/components/customer/session-checkout-page.tsx` | Per-item special request textareas; `customerName`, `customerPhone`, `customerNote` | Manual state | None for fields; cart non-empty guard | If cart empty: `Giỏ hàng đang trống...`; API errors mapped | Submit creates public order, persists order id, clears cart, routes to order page | Per-item notes update shared cart immediately on change | RHF migration must keep immediate `updateCart` side effect for item notes. |
| `src/components/customer/session-cart-sheet.tsx` | Cart item special request textarea; checkout link | Manual shared cart state | None | Checkout link prevents navigation if cart empty/updating | Textarea updates cart; link routes checkout | Recalculate shared cart | If migrated, preserve immediate shared cart updates. |

## 6. `scan-now-customer` inventory: owner/manager back office forms

| Area / component | Fields / controls | Current state | Required fields | Current validation / messages | Submit / action behavior | Reset / side effects | Migration notes |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `owner/restaurant/owner-restaurant-page.tsx` + `owner-restaurant-form.tsx` | `name`, `slug`, `logoUrl`, `description` | Parent manual value/errors passed to form | `name`, `slug` | Inline errors: `Tên nhà hàng là bắt buộc.`, `Slug là bắt buộc.` | `useUpdateOwnerRestaurantMutation(toUpdateRestaurantPayload(value))` | Loads query data into form; clears field error on change | RHF should live in page or form with reset from query data; keep payload trim/null conversion. |
| `owner/branches/owner-branch-detail-page.tsx` + `owner-branch-form.tsx` | `name`, `slug`, `email`, `phone`, `address`, `openTime`, `closeTime`, VAT/service charge numbers | Parent manual value/errors passed to form | `name`, `slug` | Inline errors for name/slug required; optional email checks includes `@` only | Create mutates then routes branch list; edit mutates current branch | Query edit data resets form; field change clears error | RHF must preserve numeric conversion: empty strings become `undefined`; address/phone/email empty become `null`. |
| `owner/users/owner-users-page.tsx` + `owner-user-form-dialog.tsx` | `fullName`, `username`, `email`, `phoneNumber`, create-only `password`, `role`, `branchIds` | Parent manual form/errors; dialog controlled | `fullName`, `username`, `email`, `role`, `branchIds`, create-only `password` | Inline errors for required fields; no email-format validation | Create/update owner-scoped users, refetch, clamp page, close dialog | Open create resets to `EMPTY_FORM`; edit preloads selected user; close resets | RHF + Controller for role dropdown and branch checkbox items. Preserve `phoneNumber.trim() || undefined`. |
| `organisms/manager-users/index.tsx` + `user-form-dialog.tsx` | `fullName`, `username`, `email`, `phoneNumber`, create-only `password`, `role`, hidden/optional branch selector | Manual form; no inline errors | `fullName`, `username`, `email`, `role`, `branchIds`, create-only `password` | Warning toast: `Please complete all required fields.` | Create/update manager users; invalidates/refetches; success/error toasts | Create pre-fills all managed branches; edit loads user branchIds; close resets | RHF must keep toast validation style, not introduce inline error UI unless intentionally approved. |
| `manage-menu/category-form-page.tsx` | `name`, `displayOrder`, `imageUrl`, `description` | Manual state | `name` | Warning toasts: category name required, display order >= 0, image URL valid | Create routes list; edit updates in place | Edit query populates form | RHF must preserve warning toasts and payload: `displayOrder: Number(value || 0)`. |
| `manage-menu/menu-item-form-page.tsx` | `categoryId`, `name`, `imageUrl`, upload file, `description`, `price`, `costPrice`, `preparationTime`, `displayOrder`, `isAvailable`, `isFeatured` | Manual state | `categoryId`, `name` | Warning toasts: category/name required, create price >= 0, other numeric >= 0, image URL valid; upload image type/size checks | Create routes list; edit updates; upload sets `imageUrl`; edit price disabled and updated via price dialog | Create auto-selects first category when available | RHF must preserve create/edit payload difference: update includes `categoryId`, create sends category in route and data without categoryId. |
| `manage-menu/update-price-dialog.tsx` | `price`, `note` | Manual state | `price` | Warning toast: `New price must be greater than 0.` | `PATCH` price with `price: Number(price)`, `note.trim() || null`; closes dialog | On open, price resets to current price and note clears | RHF reset must happen only when dialog opens/current price changes. |
| `owner/tables/owner-table-create-page.tsx` + `owner-table-form.tsx` | `tableNumber`, `capacity` | Parent manual form; shared form component | `tableNumber`, `capacity` | Warning toasts: table number required; capacity finite and >= 1 | Create table; show success panel; download/view actions | Created table stored separately; form not auto-cleared in current code | RHF can be in parent or shared form. Preserve success panel and QR actions. |
| `owner/tables/owner-table-detail-page.tsx` + `owner-table-form.tsx` | `tableNumber`, `capacity`; status buttons; active toggle; QR actions | Manual form for table info; action buttons for rest | `tableNumber`, `capacity` | Same table validation toasts | Update table then refetch; status/active/QR are separate actions | Query data resets form; history pagination separate | RHF only for editable table info fields. Action dialogs without input stay non-RHF. |
| `settings/branch-settings-page.tsx` payment config | `branchId`, `payOsEnabled`, PayOS client/API/checksum keys, `defaultMethod` | Manual state | `branchId` for save action | No frontend validation for PayOS keys; helpers say required when enabled but submit is not blocked | Upsert payment config with `cashEnabled: true`; default method forced to CASH when PayOS disabled; invalidates query; success toast | Branch auto-selects first branch; payment config query resets fields, leaves secret fields blank | Do not add required stars to PayOS keys under current rule. |
| `settings/branch-settings-page.tsx` paper voucher | `code`, `name`, `description`, `discountType`, numeric amount/quantity fields, dates | Manual state | None by current frontend validation | No frontend validation before submit | Create voucher with code uppercase, dates converted to day start/end, reset form, invalidate query, success toast | Form resets to `emptyVoucher` after create | Required stars remain absent unless validation is added intentionally later. |

## 7. `scan-now-customer` inventory: operations, cashier, waiter

| Area / component | Fields / controls | Current state | Required fields | Current validation / messages | Submit / action behavior | Reset / side effects | Migration notes |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `waiter/waiter-dashboard-page.tsx` branch/order/menu controls | Branch select, order filter buttons, order search, menu search/category buttons | Manual state | None | None | Controls drive active branch, filtered orders/menu | Changing branch clears selected order/view-specific state | Filter/control migration must preserve branch selection side effects. |
| `waiter/waiter-dashboard-page.tsx` create order flow | Selected table, menu search/category, cart quantities, `customerName`, `customerNote` | Manual state + cart array | Selected table; cart non-empty guard | Button disabled if no selected table/cart/submitting; `submitOrder` returns early for missing branch/table/cart | Opens table session if absent, creates order, clears cart/name/note, selects new order, routes back orders, refreshes data | Cart quantity changes immediate local state | RHF can own selectedTable/customer fields/search; cart array remains side-effect state. |
| `cashier/cashier-dashboard-page.tsx` branch/order/search controls | Branch select, order search, view tabs, category/menu search/filter | Manual state | None | None | Drive visible orders/menu/table map | Branch/view changes reset several selected states | Preserve view/selection behavior. |
| `cashier/cashier-dashboard-page.tsx` create manual order | Selected table, menu search/category, cart quantities, `customerName`, `customerNote` | Manual state + cart array | Selected table; cart non-empty guard | Button disabled if no selected table/cart/submitting; submit returns early if missing branch/table/cart | Opens table if no session, creates order, clears cart/name/note, selects order, clears PayOS payment, routes orders, refreshes | Cart quantity changes immediate local state | Same RHF boundary as waiter. |
| `cashier/cashier-dashboard-page.tsx` cash payment dialog | `amountReceivedInput` | Manual state | `amountReceivedInput` when confirming CASH | Inline message if amount entered but `< total`; confirm button disabled until amount >= total | `confirmCashPayment` calls checkout CASH with amount | Dialog close controlled separately; selected order drives totals | RHF should preserve computed `cashChange`, disabled state, and message. |
| `cashier/cashier-dashboard-page.tsx` order detail voucher | `voucherCode` | Manual state | None | Backend/API handles voucher errors | Applies voucher through order detail action flow | `voucherCode` owned in parent dashboard state | RHF optional, but no required star. |
| `me/my-branch-orders-page.tsx` waiter service queue | Pending confirm buttons; ready-item checkboxes | Manual selected arrays | None | Actions disabled when no selected ready IDs | Confirm order, mark served selected ready IDs | Selection arrays update per checkbox/toggle-all | Checkbox groups are operational selection controls, not submit form fields. |
| `me/my-branch-kitchen-page.tsx` kitchen queue | Pending item checkboxes, confirmed group/item checkboxes, filter buttons | Manual selected arrays | None | Actions disabled when no selected IDs | Confirm selected/order, mark ready group/selected | Selection arrays filtered after mutation | RHF migration optional for checkbox groups; preserve exact selected ID arrays. |
| `me/my-branch-menu-page.tsx` menu availability | Search, availability select/buttons, category popover, sort popover, select-visible checkbox, per-item checkboxes/toggles | Manual state | None | Bulk actions disabled with no selection | Toggle availability, bulk availability, refetch | Selection arrays clear after bulk | Control-only; preserve mobile/desktop duplicate controls. |
| `me/my-branch-tables-page.tsx` table controls | Search, status select, active select, sort select, open/close table buttons | Manual state | None | None | Filter query; open table; close session; copy session code | Opening table stores opened session banner | Filter-only plus action buttons. |

## 8. `scan-now-customer` inventory: list/report/filter controls

| Area / component | Fields / controls | Current state | Required fields | Current validation / messages | Submit / action behavior | Reset / side effects | Migration notes |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `owner/branches/owner-branches-toolbar.tsx` | Search, status filter | Controlled props | None | None | Parent debounces search and builds branch query | Search debounce resets page via parent effect | Keep debounce/page reset. |
| `owner/users/owner-users-toolbar.tsx` | Search, role/branch/status filters | Controlled props + dropdown helper | None | None | Parent query filters, reset page on filter changes | Search debounce resets page via parent effect | Use RHF watch in parent or toolbar while keeping callbacks. |
| `organisms/manager-users/manager-users-toolbar.tsx` | Search, role/branch/status filters | Controlled props + dropdown helper | None | None | Parent query filters, reset page on filter changes | Search debounce resets page via parent effect | Same as owner users toolbar. |
| `owner/tables/owner-table-list-page.tsx` | Search, status, active, capacity, sort | Manual local state | None | None | Builds `OwnerTablesQuery`; QR download/regenerate/active actions separate | `setFilter` resets page | Filter-only. Preserve numeric capacity conversion to `undefined` when empty. |
| `manage-menu/category-list-page.tsx` | Search, status, sort | Manual local state | None | None | Builds category list query; reorder and active toggle actions | `setPageNumber(1)` on filter changes | Filter-only. Preserve reorder current-page behavior. |
| `manage-menu/menu-item-list-page.tsx` | Search, category/status/availability/featured filters, select-visible, per-item checkboxes | Manual local state | None | None | Builds menu query; bulk availability; reorder; active/available/featured toggles | Filter changes reset page; bulk clears selected IDs | Filter/selection controls. Preserve selected ID semantics. |
| `owner/orders/owner-branch-orders-page.tsx` | Search, table number, status/payment/sort/date/method filters | Manual local state | None | None | Builds invoice query; refresh/clear filters; selected order dialog | `setFilter` resets page; clear resets filters/sort/page | Filter-only. |
| `reports/report-dashboard-page.tsx` | Period buttons, branch select, from/to date, export/refresh buttons | Manual local state | None | None | Builds report query and Excel export | Preset changes dates; date changes set preset custom | Preserve period/date coupling. |

## 9. Migration risk notes

- Many controls are duplicated for mobile/desktop (cashier/waiter, `me/my-branch-menu-page`). A RHF migration must avoid desync between duplicate UI controls.
- Some existing manual validations use warning toasts rather than inline errors. Preserving UX means keeping warning toasts unless a separate UX change is approved.
- Several fields are required only by disabled button/guard, not error messages. Required label can be added without adding new errors.
- Payment/voucher forms likely have backend-required fields, but current frontend does not validate them. Under this audit rule they stay non-required.
- Customer cart notes update shared realtime cart on every change. Do not defer those updates to submit during RHF migration.
- Waiter/cashier create-order opens table sessions before creating orders. RHF must not reorder or remove that side effect.
- Existing `PlaceOrderModal` already uses RHF and should not be regressed while normalizing labels.
