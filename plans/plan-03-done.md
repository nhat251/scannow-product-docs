# Plan 03: Cấu trúc lại Form với React Hook Form & Zod

## 1. Mục tiêu

Refactor các form đã code ở FE (cụ thể là form thông tin khách hàng trong `PlaceOrderModal`) để sử dụng `react-hook-form` cùng với các UI components của Shadcn thay vì thẻ HTML input thuần và state tĩnh.

Phương pháp này giúp code dễ bảo trì hơn, hỗ trợ validation phức tạp với `zod` và thống nhất design system của project.

## 2. Phân tích hiện trạng

- File hiện tại: `src/components/customer/menu/place-order-modal.tsx`
- Đang dùng `useState` cho `customerName`, `customerPhone`, `customerNote`.
- Đang dùng `<input>` và `<textarea>` HTML thuần.
- Trong codebase đã cài đặt `react-hook-form`, `@hookform/resolvers`, `zod`.
- Hệ thống UI components cho field đã có sẵn tại `src/components/ui/field.tsx` (gồm `Field`, `FieldLabel`, `FieldContent`, `FieldError`) và `src/components/ui/input.tsx`, `src/components/ui/textarea.tsx`.

## 3. Các bước thực hiện chi tiết

### Bước 3.1: Tạo Zod Schema

Tạo schema validation (có thể đặt chung trong modal hoặc file schema riêng) để giới hạn điều kiện nhập liệu.

```typescript
import { z } from "zod";

export const placeOrderSchema = z.object({
  customerName: z
    .string()
    .max(100, "Tên không được vượt quá 100 ký tự")
    .optional(),
  customerPhone: z
    .string()
    .regex(/(84|0[3|5|7|8|9])+([0-9]{8})\b/g, "Số điện thoại không hợp lệ")
    .optional()
    .or(z.literal("")),
  customerNote: z
    .string()
    .max(300, "Ghi chú không được dài quá 300 ký tự")
    .optional(),
});

export type PlaceOrderFormValues = z.infer<typeof placeOrderSchema>;
```

### Bước 3.2: Tích hợp `useForm` vào `PlaceOrderModal`

Sử dụng `useForm` với `zodResolver`:

```typescript
const form = useForm<PlaceOrderFormValues>({
  resolver: zodResolver(placeOrderSchema),
  defaultValues: {
    customerName: "",
    customerPhone: "",
    customerNote: "",
  },
});
```

Xóa bỏ các state cũ (`customerName`, `customerPhone`, `customerNote`).

### Bước 3.3: Dùng `Controller` và Shadcn UI Components

Thay thế HTML thuần bằng `Controller` và cấu trúc `Field` chuẩn của project:

```tsx
import { Controller } from "react-hook-form";
import {
  Field,
  FieldLabel,
  FieldContent,
  FieldError,
} from "@/components/ui/field";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";

// Ví dụ cho field Tên khách hàng:
<Controller
  name="customerName"
  control={form.control}
  render={({ field, fieldState }) => (
    <Field invalid={!!fieldState.error}>
      <FieldLabel className="text-slate-400">
        <User className="mr-1.5 h-3.5 w-3.5" /> Tên khách hàng (Không bắt buộc)
      </FieldLabel>
      <FieldContent>
        <Input
          {...field}
          placeholder="Nhập tên của bạn..."
          className="bg-slate-950 border-slate-800 text-white"
        />
      </FieldContent>
      <FieldError errors={[fieldState.error]} />
    </Field>
  )}
/>;
```

### Bước 3.4: Wrap form và xử lý Submit

- Bọc toàn bộ body của modal trong thẻ `<form onSubmit={form.handleSubmit(onSubmit)}>` thay vì dùng sự kiện onClick trên button.
- Khi validation qua, dữ liệu sạch sẽ được truyền vào `onSubmit` callback.

## 4. Finalize

- Lint project và sửa đảm bảo ko còn 1 lỗi nào

## 5. Bàn giao

Bản plan này hướng dẫn cụ thể cách chuyển từ React State thuần sang React Hook Form + Zod bằng cách sử dụng các components được build sẵn trong hệ thống (như `Field`, `Input`, `Textarea`). Khi bạn sẵn sàng, chúng ta có thể bắt đầu tiến hành refactor theo plan này!
