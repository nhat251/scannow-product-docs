# Kế hoạch Cô lập Dữ liệu Hoàn toàn (Strict Data Isolation Plan)

Hiện tại, hệ thống ScanNow đang dựa vào việc truyền `branchId` đúng qua API Route để truy vấn và filter dữ liệu. Mặc dù backend đã có `TenantResolutionMiddleware` và EF DbContext filter cho bảng `Branch`, hệ thống vẫn tiềm ẩn rủi ro Data Leakage. Cụ thể:

1. **Rủi ro rò rỉ dữ liệu:** Nếu dev quên filter theo `BranchId` hoặc `Order` ở một API nào đó, dữ liệu của toàn bộ các nhà hàng khác sẽ bị lộ do EF Core không tự động áp dụng `Branch` filter lên các sub-entities.
2. **Rủi ro Access Control:** Người dùng (User) không thuộc nhà hàng vẫn có thể Login thành công vào domain của nhà hàng đó, do bảng `ApplicationUser` là bảng global và hàm `LoginAsync` không thực hiện việc validate chéo với Tenant.

Vì yêu cầu thiết kế là **không thay đổi cấu trúc DB Schema (không thêm `RestaurantId` vào mọi bảng)** để giữ app nhẹ nhàng, chúng ta sẽ áp dụng **Application-level Isolation** thông qua Entity Framework Core (Global Query Filters) và Authentication Validation.

---

## 1. Tầng Security & Auth (Ngăn chặn Login trái phép)

Kiểm soát ngay từ "cổng vào" của hệ thống: Chặn User không thuộc Tenant được cấp phép ngay tại thời điểm Login.

### 1.1 Chi tiết thay đổi: `ScanNow.Application/Features/Auth/AuthService.cs`
- Inject `ITenantContext` và `ApplicationDbContext` (hoặc Repository liên quan).
- Trong hàm `LoginAsync` và `LoginGoogleAsync`, sau khi xác thực thành công thông tin đăng nhập (Password/Google Token), thêm logic xác thực Tenant:
  - Nếu `_tenantContext.IsResolved` là `true` (nghĩa là đang truy cập qua một subdomain của một nhà hàng cụ thể):
    - Lấy `RestaurantId` hiện tại từ context.
    - Truy vấn DB kiểm tra xem User có thỏa mãn **1 trong 2** điều kiện sau không:
      1. Là `Owner` của `Restaurant` hiện tại.
      2. Nằm trong bảng `BranchStaff` của **bất kỳ** `Branch` nào thuộc `Restaurant` này.
    - Nếu cả 2 đều **KHÔNG** -> Quăng lỗi `UnauthorizedException("Bạn không có quyền truy cập vào nhà hàng này.")`.

---

## 2. Tầng Data Access (EF Core Global Query Filters)

Đảm bảo mọi query `SELECT`, `UPDATE`, `DELETE` gọi qua EF Core đều bị ép thêm điều kiện Tenant ở tầng thấp nhất, loại bỏ hoàn toàn việc phụ thuộc vào ý thức của Dev ở tầng API.

### 2.1 Chi tiết thay đổi: `ScanNow.Infrastructure/ApplicationDbContext.cs`
Thay thế filter hiện tại (chỉ áp dụng cho mỗi bảng `Branch`) bằng cách thiết lập filter cho **toàn bộ** các Entity liên quan đến Tenant. Cụ thể, trong hàm `OnModelCreating`, cấu hình rõ ràng như sau:

```csharp
var tenantId = _tenantContext?.RestaurantId;
var isResolved = _tenantContext?.IsResolved == true;

// 1. Filter trực tiếp cho Restaurant (Aggregate Root lớn nhất)
builder.Entity<Restaurant>().HasQueryFilter(e => !isResolved || e.Id == tenantId);

// 2. Filter cho các Entity cấp 1 (Có chứa BranchId)
builder.Entity<Branch>().HasQueryFilter(e => !isResolved || e.RestaurantId == tenantId);
builder.Entity<BranchStaff>().HasQueryFilter(e => !isResolved || e.Branch.RestaurantId == tenantId);
builder.Entity<Category>().HasQueryFilter(e => !isResolved || e.Branch.RestaurantId == tenantId);
builder.Entity<MenuItem>().HasQueryFilter(e => !isResolved || e.Branch.RestaurantId == tenantId);
builder.Entity<RestaurantTable>().HasQueryFilter(e => !isResolved || e.Branch.RestaurantId == tenantId);
builder.Entity<Order>().HasQueryFilter(e => !isResolved || e.Branch.RestaurantId == tenantId);
builder.Entity<QrSession>().HasQueryFilter(e => !isResolved || e.Branch.RestaurantId == tenantId);
builder.Entity<ItemRating>().HasQueryFilter(e => !isResolved || e.Branch.RestaurantId == tenantId);
builder.Entity<DiscountCode>().HasQueryFilter(e => !isResolved || e.Branch.RestaurantId == tenantId);
builder.Entity<BranchPaymentConfig>().HasQueryFilter(e => !isResolved || e.Branch.RestaurantId == tenantId);
builder.Entity<PaperVoucher>().HasQueryFilter(e => !isResolved || e.Branch.RestaurantId == tenantId);

// 2.1 Các Entity cấp 1 (Nullable BranchId)
builder.Entity<Notification>().HasQueryFilter(e => !isResolved || (e.Branch != null && e.Branch.RestaurantId == tenantId));
builder.Entity<AuditLog>().HasQueryFilter(e => !isResolved || (e.Branch != null && e.Branch.RestaurantId == tenantId));

// 3. Filter cho các Entity cấp 2 (Các entity không có BranchId, filter qua Navigation Property)
builder.Entity<OrderItem>().HasQueryFilter(e => !isResolved || e.Order.Branch.RestaurantId == tenantId);
builder.Entity<Payment>().HasQueryFilter(e => !isResolved || e.Order.Branch.RestaurantId == tenantId);
builder.Entity<MenuItemPriceHistory>().HasQueryFilter(e => !isResolved || e.MenuItem.Branch.RestaurantId == tenantId);
```

> **Lưu ý Kiến trúc (Architecture Note):**
> Việc liệt kê tường minh (explicit) như trên rất phù hợp cho ứng dụng quy mô vừa và nhỏ. Nó đảm bảo hiệu năng và dễ dàng debug (vì EF Core sẽ tự động inject các câu lệnh `LEFT JOIN` vào các truy vấn DB) mà không phải dùng Reflection phức tạp hay thay đổi Schema Database.

---

## 3. Câu hỏi mở cần chốt (Open Questions)

Để tôi có thể bắt đầu code chính xác nhất, bạn vui lòng chốt giúp tôi ý sau:

1. **Filter Entities con:** Các thực thể cấp sâu như `OrderItem` hay `MenuItemPriceHistory` đã được tôi bổ sung vào kế hoạch để filter thông qua Navigation property. Tuy nhiên điều này sẽ tạo ra lệnh `JOIN` dưới Database khi truy vấn. Bạn xác nhận giữ nguyên hướng tiếp cận bao trùm tới tận các entity con này nhé?

---

## 4. Kế hoạch Kiểm thử (Verification Plan)

Sau khi hoàn tất code, các Test Case sau sẽ được sử dụng để xác minh tính đúng đắn của giải pháp:

- **TC1 - Cross-Tenant Login:** Dùng tài khoản nhân viên của Nhà hàng A để đăng nhập vào domain của Nhà hàng B -> **Kỳ vọng:** Bị chặn với lỗi `401 Unauthorized` hoặc `403 Forbidden`.
- **TC2 - Data Leakage Prevention:** Viết một đoạn code test query `dbContext.Orders.ToList()` trên API của Nhà hàng A mà không dùng bất kỳ lệnh `.Where()` nào -> **Kỳ vọng:** Chỉ trả về các Orders của Nhà hàng A, tuyệt đối không có dữ liệu của Nhà hàng B, C.
