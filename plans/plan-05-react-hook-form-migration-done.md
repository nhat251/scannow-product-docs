# Plan 05: Migration to React Hook Form for all FE controls

Ngay lap plan: 2026-06-13

Nguon doi chieu bat buoc:

- Audit file: `/home/nhat/fpt/scannow-product-docs/10-react-hook-form-audit.md`.
- Landing/Admin FE: `/home/nhat/fpt/scan-now-nextjs`.
- Tenant/Portal FE: `/home/nhat/fpt/scan-now-customer`.

Muc tieu:

1. Toan bo form va user-editable controls trong 2 FE repo duoc to chuc bang `react-hook-form` theo pham vi audit.
2. Moi field required theo validation/guard frontend hien tai co dau `*` mau do tren label bang reusable component/API.
3. Logic truoc/sau khong doi: payload, trim/null conversion, API call, toast, redirect, refetch/invalidate, reset, debounce, page reset, cart/session side effect phai giu nguyen.
4. Khong them package moi. 2 repo da co `react-hook-form`, `@hookform/resolvers`, `zod`.

## 1. Public UI/API changes

### 1.1 Extend shared `Label`

Thuc hien giong nhau trong ca 2 repo:

- File:
  - `/home/nhat/fpt/scan-now-nextjs/src/components/ui/label.tsx`
  - `/home/nhat/fpt/scan-now-customer/src/components/ui/label.tsx`
- Them prop:

```tsx
type LabelProps = React.ComponentProps<typeof LabelPrimitive.Root> & {
  required?: boolean;
  requiredClassName?: string;
};
```

- Render:

```tsx
function Label({ className, required, requiredClassName, children, ...props }: LabelProps) {
  return (
    <LabelPrimitive.Root ...>
      {children}
      {required ? (
        <span aria-hidden="true" className={cn("text-destructive", requiredClassName)}>
          *
        </span>
      ) : null}
    </LabelPrimitive.Root>
  );
}
```

Quy tac:

- `required` chi la visual marker. Khong set native `required`, khong doi browser validation, khong doi submit behavior.
- `FieldLabel` dang wrap `Label`, nen `FieldLabel required` se tu hoat dong. Khong can tao component moi rieng neu API tren du dung.
- Native `label` trong cac file chua dung `Label` nen migrate sang `Label`/`FieldLabel` khi co required marker.
- Optional labels khong render `*`.

### 1.2 Error display conventions

Khong thong nhat lai UX neu hien tai khac nhau:

- Form dang inline error thi tiep tuc inline error.
- Form dang warning toast thi tiep tuc warning toast.
- Form dang chi disabled submit khi thieu field thi tiep tuc disabled submit, khong them inline error moi.
- Server/API error mapping giu nguyen.

## 2. Global migration rules

Moi dev phai doc audit truoc khi sua tung file. Voi moi row trong audit:

1. Xac dinh field nao required theo cot `Required fields`.
2. Xac dinh current UX validation theo cot `Current validation / messages`.
3. Viet RHF sao cho output submit/action giong y het.
4. Neu field la filter/search, dung `useForm` + `useWatch` nhung van giu debounce/page reset/query mapping hien tai.
5. Neu field co side effect ngay khi change, nhu shared cart note, van goi side effect o `onChange`/watch dung thoi diem hien tai.
6. Neu la action dialog khong co input, khong migrate sang RHF.

Code conventions:

- Prefer `zodResolver` cho form dang co validation rules ro rang hoac inline errors.
- Voi validation dang la toast/guard thu cong, co the dung RHF khong resolver va validate trong submit handler de giu message/toast y het.
- Dung `Controller` cho Radix `Select`, dropdown radio/checkbox, popover-driven selection, checkbox group, and custom trigger buttons.
- Dung `register` cho plain `Input`/`Textarea` khi khong can custom controlled behavior.
- Dung `reset(...)` khi query data/edit record thay doi.
- Dung `watch`/`useWatch` de tinh disabled state, params, and labels.
- Khong doi types public/API payload neu khong can.

## 3. Migration order

### Phase 0 - Guard rails

1. Tao branch rieng cho migration.
2. Doc lai audit file va ghi checklist local theo tung repo.
3. Chay baseline:
   - `pnpm lint` trong `scan-now-nextjs`.
   - `pnpm lint` trong `scan-now-customer`.
4. Chuan bi browser smoke:
   - Cai Playwright/Chromium hoac dung browser manual.
   - Co credential admin, owner, manager, staff/waiter, kitchen, cashier.
   - Co tenant/dev slug va backend API hoat dong.
5. Neu khong co credential/browser, ghi ro blocked scenario trong ket qua test. Khong pretend pass.

### Phase 1 - Shared required label

1. Sua `Label` trong ca 2 repo theo muc 1.1.
2. Verify:
   - Existing imports khong fail.
   - Existing label without `required` render giong cu.
   - `FieldLabel required` compiles vi prop truyen qua `Label`.
3. Chay `pnpm lint` ca 2 repo sau phase nay.

### Phase 2 - `scan-now-nextjs` existing RHF forms

Suu tam bat buoc tu audit section 4.

#### Admin login

- File: `src/components/admin/admin-login-card.tsx`.
- Giu schema:
  - `identifier`: trim min 1.
  - `password`: trim min 1.
- Them `required` vao labels `Email or username`, `Password`.
- Giu `isSubmitting`, `errorMessage`, localStorage JWT, user store, redirect `/admin`.
- Khong doi text error.

#### Create/edit owner dialog

- File: `src/components/admin/create-owner-dialog.tsx`.
- Labels required:
  - Always: Full name, Username, Email.
  - Create only: Password.
  - Phone number optional.
- Giu dynamic schema by `isEdit`.
- Giu reset khi open va close.
- Giu payload:
  - create `phoneNumber: values.phoneNumber || null`, `password: values.password || ""`.
  - edit no password.

#### Restaurant form

- File: `src/components/admin/restaurant-form.tsx`.
- Required labels:
  - Create only owner select.
  - Restaurant name, Slug.
- Optional labels:
  - Logo URL, Description.
- Giu slug auto-generation chi trong create mode.
- Giu owner select `watch("ownerId")`.
- Giu `No owners available...` helper.

#### Lead capture modal

- File: `src/components/molecules/lead-capture-modal.tsx`.
- Doi native `label` sang shared `Label` hoac them required support tu shared API.
- Required labels: phone, email, location.
- Giu i18n validation keys va `VIETNAM_LOCATIONS`.
- Giu reset on close and after successful submit.

### Phase 3 - `scan-now-nextjs` control-only filters

Muc tieu la RHF organization nhung query behavior khong doi.

#### Controlled filter components

Files:

- `src/components/admin/restaurant-filters.tsx`
- `src/components/admin/owner-filters.tsx`

Implementation decision:

- Cho component filter tu so huu `useForm` voi `defaultValues` tu props.
- Dung `useEffect` de `reset` form khi external props thay doi.
- Dung `useWatch` de goi callbacks khi user doi gia tri.
- Can co guard de tranh loop:
  - Chi goi callback neu watched value khac prop hien tai.
  - Page size parse bang `Number(value)` nhu cu.
- Required: none.

#### Local filter tab components

Files:

- `branch-tables-tab.tsx`
- `branch-categories-tab.tsx`
- `branch-menu-items-tab.tsx`
- `restaurant-branches-tab.tsx`

Implementation decision:

- Thay cac `useState` filter/search/sort bang mot `useForm<FilterValues>`.
- Dung `useWatch` de build query params.
- Giu `pageNumber` va `activeView` la normal state vi day la pagination/view state, khong phai form data.
- Giu logic reset page:
  - Khi search/status/category/etc thay doi, `setPageNumber(1)`.
  - Sort direction toggle co the dung `setValue("sortDirection", next)`.
- Giu refresh/invalidate actions ngoai RHF.

#### Language switcher

- File: `src/components/molecules/globals/language-switcher.tsx`.
- Neu strict "all controls", wrap locale select bang `useForm<{ locale: SupportedLocale }>` and `Controller`.
- Giu `handleLocaleChange` y het:
  - guard unsupported/same locale.
  - rewrite locale segment.
  - preserve query and hash.
  - `router.replace(href)`.
- Required: none.

## 4. `scan-now-customer` migration

### Phase 4 - Tenant auth and existing RHF

#### Login form

- File: `src/components/auth/login-form.tsx`.
- RHF fields: `identifier`, `password`, `rememberMe`.
- Required labels: Identifier, Password.
- Do not add new required error messages.
- Preserve disabled logic:

```tsx
const values = useWatch({ control });
const isDisabled =
  loginMutation.isPending ||
  !values.identifier.trim() ||
  !values.password.trim();
```

- Submit payload must remain:

```ts
{
  identifier: values.identifier.trim(),
  password: values.password,
}
```

- `rememberMe` currently not submitted. Keep it not submitted.
- Show password toggle remains normal state.
- Preserve auth bootstrap redirect and error banner.

#### PlaceOrderModal

- File: `src/components/customer/menu/place-order-modal.tsx`.
- Already RHF. Only clean imports/format if needed and use shared labels.
- Required: none.
- Preserve optional validation exactly:
  - name max 100.
  - phone regex optional or empty.
  - note max 300.
- Preserve no explicit close reset unless intentionally accepted. Audit says current code does not reset on close.

### Phase 5 - Customer public cart/order forms

#### Session menu item detail

- File: `src/components/customer/session-menu-item-page.tsx`.
- RHF fields: `note`, `localQuantity`.
- Required: none.
- On cart line change, call `reset({ note: cartLine?.specialRequest ?? "", localQuantity: Math.max(cartLine?.quantity ?? 1, 1) })`.
- Quantity buttons use `setValue("localQuantity", next)`.
- `handleSaveToCart` uses `getValues()`.
- Preserve payload:
  - `specialRequest: note.trim() || null`.
  - `quantity: localQuantity`.
- Preserve toast text, including current text mismatch if any. Do not "fix" copy during migration.

#### Session checkout page

- File: `src/components/customer/session-checkout-page.tsx`.
- RHF fields: `customerName`, `customerPhone`, `customerNote`.
- Required: none.
- Per-item `specialRequest` textareas:
  - Either keep controlled by cart state with `Controller`, or keep as direct controlled inputs if RHF would delay side effect.
  - Required behavior: `updateSpecialRequest` still runs on every textarea change.
- Submit behavior unchanged:
  - If cart empty, set same `formError`.
  - Payload trims fields to `null`.
  - Items map to `{ menuItemId, quantity, note: specialRequest?.trim() || null }`.
  - Persist order, clear cart, route to order page.

#### Session cart sheet

- File: `src/components/customer/session-cart-sheet.tsx`.
- Controls are cart note textareas and checkout link.
- If migrated, use RHF only for local note fields while still calling shared cart update immediately.
- Required: none.
- Preserve link guard when cart empty/updating.

### Phase 6 - Owner restaurant and branches

#### OwnerRestaurantPage/Form

- Files:
  - `owner/restaurant/owner-restaurant-page.tsx`
  - `owner/restaurant/owner-restaurant-form.tsx`
- Decision: move RHF ownership to page because query data and mutation live there.
- Define schema:
  - `name`: trim min 1 with `Tên nhà hàng là bắt buộc.`
  - `slug`: trim min 1 with `Slug là bắt buộc.`
  - `logoUrl`, `description`: optional strings, no new URL validation.
- Required labels: `Tên nhà hàng`, `Slug`.
- On restaurant query data, `reset(toOwnerRestaurantFormValues(data))`.
- Form component should receive RHF methods or become self-contained with `FormProvider`.
- Submit calls `handleSubmit(async values => updateMutation.mutateAsync(toUpdateRestaurantPayload(values)))`.
- Preserve `toUpdateRestaurantPayload`: trim, empty logo/description to `null`.

#### OwnerBranchDetailPage/Form

- Files:
  - `owner/branches/owner-branch-detail-page.tsx`
  - `owner/branches/owner-branch-form.tsx`
- RHF owner in page.
- Schema:
  - `name`: required message `Tên chi nhánh là bắt buộc.`
  - `slug`: required message `Slug là bắt buộc.`
  - `email`: optional but if present must include `@`, message `Email không hợp lệ.`
  - Other fields optional strings.
- Required labels: `Tên chi nhánh`, `Slug`.
- Submit preserves:
  - create: mutate then `router.push(PATH.owner.branches)`.
  - edit: if branchId, update.
  - `toBranchPayload` conversions exactly as current.
- Numeric strings remain strings in RHF until payload conversion.

### Phase 7 - Owner and manager users

#### Owner users

- Files:
  - `owner/users/owner-users-page.tsx`
  - `owner/users/owner-user-form-dialog.tsx`
- Move dialog form state/errors to RHF.
- Keep parent state for filters, pagination, query result, selected editing user, dialog open/mode.
- On open create: `reset(EMPTY_FORM)`.
- On open edit: `reset({ user fields... })`.
- Required labels:
  - Full name, Username, Email, Role, Branch assignment.
  - Password only in create mode.
- Use `Controller` for:
  - role dropdown radio group.
  - branch assignment checkbox/dropdown items.
- Inline `FieldError` remains because current owner dialog has inline errors.
- Validation messages exactly:
  - `Họ tên là bắt buộc.`
  - `Tên đăng nhập là bắt buộc.`
  - `Email là bắt buộc.`
  - `Mật khẩu là bắt buộc.`
  - `Chọn ít nhất một chi nhánh.`
- Preserve create/update payloads and refetch/close flow.

#### Manager users

- Files:
  - `organisms/manager-users/index.tsx`
  - `organisms/manager-users/user-form-dialog.tsx`
  - `organisms/manager-users/branch-multi-select.tsx`
  - `organisms/manager-users/filter-dropdown.tsx`
- Move dialog form to RHF.
- Required labels:
  - Full name, Username, Email, Role.
  - Password only in create mode.
  - Branches required in data even when `showBranchSelection=false`; if branch selector is hidden, do not show a visible label star for hidden UI.
- Preserve current validation UX:
  - On invalid required submit, show warning toast `Please complete all required fields.`
  - Do not add inline FieldErrors unless separately approved.
- Create defaults branchIds to all managed branches.
- Submit payload must preserve:
  - create password is `form.password.trim()`.
  - update omits password.
  - phone uses `trim() || undefined`.
- Preserve `handleApiError`, forbidden banner, query invalidation, success toasts.

### Phase 8 - Menu/category/table management

#### Category form

- File: `manage-menu/category-form-page.tsx`.
- RHF fields: `name`, `description`, `imageUrl`, `displayOrder`.
- Required label: Category name.
- Preserve warning toasts, not inline errors.
- Submit flow:
  - run current `validate` logic using RHF values.
  - create then route list.
  - edit update.
- `displayOrder` remains string in form and `Number(value || 0)` in payload.

#### Menu item form

- File: `manage-menu/menu-item-form-page.tsx`.
- RHF fields: all current `ManageMenuItemFormValues`.
- Required labels: Category, Name.
- Optional labels: Image URL, Description, Price, Cost Price, Preparation Time, Display Order, flags.
- Preserve warning toasts exactly.
- Use `Controller` for category select and checkboxes.
- Preserve create category default:
  - If mode create, no categoryId, and categories[0] exists, `setValue("categoryId", categories[0].categoryId)`.
- Preserve image upload:
  - file type and 5MB checks.
  - upload mutation result sets `imageUrl`.
  - file input value clears after selection.
- Preserve edit price disabled and price update via `UpdatePriceDialog`.

#### Update price dialog

- File: `manage-menu/update-price-dialog.tsx`.
- RHF fields: `price`, `note`.
- Required label: New Price.
- On open/current price change, `reset({ price: String(currentPrice), note: "" })`.
- Preserve warning toast and payload.
- Close only after successful mutation.

#### Owner table form

- Files:
  - `owner/tables/owner-table-form.tsx`
  - create/detail parent pages.
- Decision: shared `OwnerTableForm` becomes RHF-aware via props:
  - Either receive `form` object from parent, or wrap with `FormProvider`.
  - Use the same component for create and detail.
- Required labels: Table Number, Capacity.
- Preserve toast validation style:
  - table number required.
  - capacity finite and >= 1.
- Preserve create success panel and detail refetch after save.
- Do not migrate QR regenerate confirm dialog; no input.

### Phase 9 - Branch settings

- File: `settings/branch-settings-page.tsx`.
- Use separate RHF forms:
  1. `branchSelectorForm` or one shared top-level form for `branchId`.
  2. `paymentConfigForm`.
  3. `voucherForm`.
- Required label:
  - `Chi nhánh` only.
- Do not mark PayOS keys required because current frontend does not validate them.
- Do not mark voucher fields required because current frontend does not validate them.
- Payment config reset:
  - When config query data loads, `reset` to current config, blank secret fields.
- Payment submit preserves:
  - `cashEnabled: true`.
  - `defaultMethod: payOsEnabled ? current default : "CASH"`.
  - invalidate payment config query.
  - success toast.
- Voucher submit preserves:
  - code uppercase.
  - name trim.
  - description trim or null.
  - date start/end conversion.
  - reset to `emptyVoucher`.
  - invalidate voucher query.
  - success toast.

### Phase 10 - Waiter/cashier operational flows

#### Shared rule

Do not force cart arrays into RHF. Cart items are operational state with side effects and computed totals.

#### Waiter create order

- File: `waiter/waiter-dashboard-page.tsx`.
- RHF fields:
  - `selectedTableId`
  - `customerName`
  - `customerNote`
  - `categorySearch`/menu search if converted in same component.
- Required label/marker:
  - selected table in create order UI.
- Preserve `submitOrder` order:
  1. return if no branch/table/cart.
  2. find table.
  3. open table if no current session.
  4. create order.
  5. clear cart/name/note.
  6. select order, set view orders.
  7. refresh data.
- Mobile cart sheet and desktop create panel must read/write the same RHF values.

#### Cashier create order

- File: `cashier/cashier-dashboard-page.tsx`.
- Same pattern as waiter.
- Preserve `setPayOsPayment(null)` after manual order create.
- Preserve `setView("orders")`.

#### Cash payment dialog

- File: `cashier/cashier-dashboard-page.tsx`.
- RHF field: `amountReceivedInput`.
- Required label: `Tiền khách đưa`.
- Preserve:
  - `cashChange` computation.
  - inline message if amount entered but invalid.
  - confirm disabled until amount >= total.
  - checkout payload for CASH with amount.
- Reset amount only where current behavior resets it. If current behavior only changes on selected order/dialog open, preserve that timing.

#### Cashier voucher code

- RHF field can live in order detail area or parent form.
- Required: none.
- Preserve backend-driven error/success behavior.

### Phase 11 - Filter/list/report controls in `scan-now-customer`

This phase should not change any mutation behavior.

Files and rules:

- `owner/branches/owner-branches-toolbar.tsx`
  - search/status via RHF, preserve parent debounce and page reset.
- `owner/users/owner-users-toolbar.tsx`
  - search/role/branch/status via RHF, preserve parent callbacks.
- `organisms/manager-users/manager-users-toolbar.tsx`
  - same.
- `owner/tables/owner-table-list-page.tsx`
  - filters in RHF; `pageNumber`, `pageSize`, regenerate dialog state stay normal state.
  - capacity converts to number only for query.
- `manage-menu/category-list-page.tsx`
  - search/status/sort in RHF; selected/reorder state unchanged.
- `manage-menu/menu-item-list-page.tsx`
  - filters in RHF; selectedIds remain normal state.
- `owner/orders/owner-branch-orders-page.tsx`
  - filters in RHF; selected order dialog state unchanged.
  - clear button resets RHF filters and page.
- `reports/report-dashboard-page.tsx`
  - branch/date controls in RHF; period preset remains normal state or controlled RHF field, but date-preset coupling must remain.
- `me/my-branch-menu-page.tsx`
  - filters in RHF; selectedIds and item toggles remain normal state.
  - mobile and desktop duplicate controls bind same RHF values.
- `me/my-branch-tables-page.tsx`
  - filters in RHF; opened session banner/action state unchanged.
- `me/my-branch-orders-page.tsx`
  - checkbox selected IDs may stay normal state because they represent action selection, not persisted form data.
- `me/my-branch-kitchen-page.tsx`
  - same for selected pending/confirmed IDs and filter buttons.

## 5. Required label checklist

After migration, visually inspect these labels show red `*`:

`scan-now-nextjs`:

- Admin login: Email or username, Password.
- Owner dialog create: Full name, Username, Email, Password.
- Owner dialog edit: Full name, Username, Email.
- Restaurant create: Owner, Restaurant name, Slug.
- Restaurant edit: Restaurant name, Slug.
- Lead modal: Phone, Email, Location.

`scan-now-customer`:

- Login: Identifier, Password.
- Owner restaurant: Ten nha hang, Slug.
- Owner branch: Ten chi nhanh, Slug.
- Owner user create/edit: Full name, Username, Email, Role, Branch assignment; Password only create.
- Manager user create/edit: Full name, Username, Email, Role; Password only create; visible branch label only if branch selector shown.
- Category form: Category name.
- Menu item form: Category, Name.
- Update price: New Price.
- Table form: Table Number, Capacity.
- Branch settings: Chi nhanh.
- Waiter/cashier create order: selected table control.
- Cashier cash dialog: Tien khach dua.

Confirm these do not show `*`:

- Optional phone fields.
- Optional URLs.
- Customer contact fields.
- Customer notes/menu notes.
- Search/filter/sort/date controls.
- Voucher/payment config fields without current frontend validation.

## 6. Test plan

### 6.1 Static checks

Run after each major phase and before final handoff:

```bash
cd /home/nhat/fpt/scan-now-nextjs && pnpm lint
cd /home/nhat/fpt/scan-now-customer && pnpm lint
```

Expected:

- 0 ESLint errors.
- 0 ESLint warnings.
- `tsc --noEmit` pass.

### 6.2 Browser smoke before and after migration

Run the same matrix before code migration and after code migration. Record results in the PR or a follow-up doc section.

Admin/Landing (`scan-now-nextjs`):

1. `/admin` login:
   - Empty required fields blocked/errors same as before.
   - Invalid credentials show same error.
   - Valid login redirects to `/admin`.
2. Owner create/edit dialog:
   - Create missing required fields shows same errors.
   - Edit does not show password field.
   - Phone optional and invalid phone still errors.
3. Restaurant create/edit:
   - Create owner/name/slug required.
   - Name auto-generates slug only in create.
   - Edit reset loads restaurant data.
4. Lead capture:
   - Missing phone/email/location same errors.
   - Success toast and modal close/reset same.
5. Filters:
   - Search/status/page-size/category/sort changes produce same query/list behavior.
   - Page resets to 1 where it did before.
6. Language switcher:
   - Locale switch preserves path/query/hash.

Tenant (`scan-now-customer`):

1. Login:
   - Empty identifier/password keeps submit disabled.
   - Invalid credentials show same mapped error.
   - Valid credentials redirect by role.
2. Owner restaurant/branch:
   - Required errors same.
   - Payload trim/null/numeric conversion unchanged.
3. Owner/manager user dialogs:
   - Required fields create/edit same.
   - Create password required; edit no password.
   - Branch assignment required behavior same.
   - Success refetch/closes dialog same.
4. Menu/category/table forms:
   - Same warning toasts.
   - Same redirects/refetch/success panels.
   - Upload image validation unchanged.
   - Edit menu price disabled and price dialog works.
5. Branch settings:
   - Branch select required by disabled button behavior.
   - Payment config save same.
   - Voucher create payload/reset same.
6. Customer flow:
   - Menu item note/quantity save updates cart.
   - Cart/checkout notes update shared cart at same timing.
   - Checkout empty cart error same.
   - Order submit clears cart and redirects same.
7. Waiter/cashier create order:
   - Selected table required and button disabled same.
   - Cart quantity behavior same.
   - Opening table session before order create preserved.
   - Submit clears fields/cart and refreshes same.
8. Cashier cash payment:
   - Amount required; invalid amount message and disabled state same.
   - Confirm sends same CASH payload.
9. Reports/list filters:
   - Search/filter/sort/date/debounce/page reset behavior same.
10. `me/*` operational controls:
   - Kitchen/waiter checkbox selected IDs and bulk actions same.
   - Menu availability toggles and bulk update same.
   - Table open/close session controls same.

### 6.3 Network/payload checks

For each mutation form, compare before/after network payload:

- Admin owner create/update.
- Admin restaurant create/update.
- Lead capture API route.
- Tenant login.
- Owner restaurant update.
- Owner branch create/update.
- Owner/manager user create/update.
- Category create/update.
- Menu item create/update.
- Price update.
- Table create/update.
- Payment config upsert.
- Paper voucher create.
- Public/customer order create.
- Waiter/cashier order create.
- Cashier checkout CASH.

Payloads must match field-for-field except for property order.

### 6.4 Blocked scenario policy

If browser/auth/backend is unavailable:

- Do not mark scenario pass.
- Mark as `Blocked` with exact reason, for example:
  - no browser automation available.
  - no admin credential.
  - no tenant slug.
  - backend API unavailable.
  - missing seed branch/table/menu data.
- Still run static checks and any unauthenticated browser checks possible.

## 7. Acceptance criteria

Migration is complete only when:

- Both docs and code reference the audit.
- Every row in audit is either migrated to RHF or explicitly documented as no-input action dialog.
- Every required field in checklist has reusable red `*`.
- No optional field listed in audit gained a required star accidentally.
- Static checks pass in both repos.
- Browser smoke matrix is executed or blocked with concrete reasons.
- No mutation payload/redirect/toast/reset/refetch behavior changes.

## 8. Suggested implementation commits

Recommended small commit order:

1. `docs: add react hook form audit and migration plan`
2. `feat(ui): add reusable required label marker`
3. `refactor(admin): normalize existing rhf forms and labels`
4. `refactor(admin): migrate admin filters to rhf`
5. `refactor(auth-customer): migrate auth and public customer controls`
6. `refactor(owner): migrate owner restaurant branch user forms`
7. `refactor(manager): migrate manager user forms`
8. `refactor(menu): migrate category menu table price forms`
9. `refactor(settings): migrate branch settings controls`
10. `refactor(ops): migrate waiter cashier operational controls`
11. `refactor(filters): migrate remaining report/list/me filters`
12. `test: record browser smoke results`

Do not combine all migration work in one large commit unless unavoidable; it will make behavioral review too hard.
