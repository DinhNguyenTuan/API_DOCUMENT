# 📘 DeHat VN - API Documentation cho Flutter Frontend

> **Base URL:** `http://localhost:3000/api`  
> **Format:** JSON (UTF-8)  
> **Auth:** Bearer Token (JWT)

---

## 📌 Cấu trúc Response chuẩn

### Thành công (200, 201)
```json
{
  "success": true,
  "message": "Thành công",
  "data": { ... }
}
```

### Thành công có phân trang
```json
{
  "success": true,
  "message": "Thành công",
  "data": [ ... ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "totalItems": 50,
    "totalPages": 5
  }
}
```

### Lỗi (400, 401, 403, 404, 500)
```json
{
  "success": false,
  "message": "Mô tả lỗi bằng tiếng Việt",
  "errorCode": "VALIDATION_ERROR",
  "errors": [
    { "field": "phone", "message": "Số điện thoại không hợp lệ" }
  ],
  "requestId": "uuid..."
}
```

| errorCode | HTTP | Mô tả |
|-----------|------|-------|
| `VALIDATION_ERROR` | 400 | Dữ liệu không hợp lệ |
| `DUPLICATE_ERROR` | 409 | Trùng dữ liệu (phone, email...) |
| `TOKEN_EXPIRED` | 401 | Access token hết hạn |
| `TOKEN_INVALID` | 401 | Token không hợp lệ |
| `PAYLOAD_TOO_LARGE` | 413 | Body > 1MB |
| `RATE_LIMIT` | 429 | Quá nhiều request |
| `INTERNAL_ERROR` | 500 | Lỗi server |

### Query Parameters chung (phân trang)
| Param | Mô tả | Default |
|-------|--------|---------|
| `page` | Trang hiện tại | `1` |
| `limit` | Số item/trang | `10` |

### Headers
```
Authorization: Bearer <access_token>
Content-Type: application/json
```

---

## 🔐 1. Authentication (`/api/auth`)

### POST `/api/auth/register` — Đăng ký
**Auth:** Không cần

**Body:**
```json
{
  "full_name": "Lý Văn Tám",
  "phone": "0901234567",
  "email": "tam@gmail.com",
  "password": "Abc12345"
}
```

**Response (201):**
```json
{
  "success": true,
  "message": "Đăng ký thành công",
  "data": {
    "user": {
      "id": "e0000000-0000-0000-0000-000000000001",
      "full_name": "Lý Văn Tám",
      "phone": "0900000008",
      "email": "tam@gmail.com",
      "avatar_url": null,
      "address": null,
      "province_id": null,
      "district_id": null,
      "ward_id": null,
      "status": "active",
      "verified_at": null,
      "created_at": "2026-02-15T08:00:00.000Z",
      "updated_at": "2026-02-15T08:00:00.000Z"
    },
    "accessToken": "eyJhbGci...",
    "refreshToken": "eyJhbGci..."
  }
}
```

**Validation (Joi):**
| Field | Rule | Lỗi |
|-------|------|-----|
| `full_name` | **Bắt buộc**, 2-100 ký tự, trim | `Họ tên phải có ít nhất 2 ký tự` |
| `phone` | **Bắt buộc**, regex `0[3|5|7|8|9]XXXXXXXX` | `Số điện thoại không hợp lệ. Vui lòng nhập SĐT Việt Nam` |
| `email` | Optional, email format, lowercase | `Email không đúng định dạng` |
| `password` | **Bắt buộc**, 8-128 ký tự, 1 hoa + 1 thường + 1 số | `Mật khẩu phải chứa ít nhất 1 chữ hoa, 1 chữ thường và 1 số` |

> ⚠️ `stripUnknown: true` — Mọi field không khai báo sẽ bị loại (chống Mass Assignment)

**Logic:**
- Kiểm tra phone đã tồn tại → `400` "Số điện thoại đã được đăng ký"
- Kiểm tra email đã tồn tại → `400` "Email đã được đăng ký"
- Hash password bằng bcrypt (12 rounds)
- Tự động gán role **farmer** (role_id = 6)
- Trả về access token (15m) + refresh token (7d)

---

### POST `/api/auth/login` — Đăng nhập
**Auth:** Không cần

**Body:**
```json
{
  "phone": "0900000008",
  "password": "123456"
}
```

**Response (200):**
```json
{
  "success": true,
  "message": "Đăng nhập thành công",
  "data": {
    "user": {
      "id": "e0000000-0000-0000-0000-000000000001",
      "full_name": "Lý Văn Tám",
      "phone": "0900000008",
      "email": "tam@gmail.com",
      "avatar_url": null,
      "status": "active",
      "roles": [
        { "id": 6, "name": "farmer", "display_name": "Nông dân" }
      ]
    },
    "accessToken": "eyJhbGci...",
    "refreshToken": "eyJhbGci..."
  }
}
```

**Validation (Joi):**
| Field | Rule |
|-------|------|
| `phone` | **Bắt buộc**, regex VN |
| `password` | **Bắt buộc**, max 128 ký tự |

**Logic:**
- Kiểm tra user tồn tại theo phone
- Kiểm tra `status !== 'active'` → `401` "Tài khoản đã bị vô hiệu hóa"
- So sánh password hash
- Trả về user kèm roles + tokens

**Rate Limit:** 15 request / 15 phút / IP (áp chung cho `/api/auth/*`)

---

### POST `/api/auth/refresh` — Làm mới token
**Auth:** Không cần

**Body:**
```json
{
  "refreshToken": "eyJhbGci..."
}
```

**Response (200):**
```json
{
  "success": true,
  "message": "Làm mới token thành công",
  "data": {
    "accessToken": "eyJhbGci...(mới)",
    "refreshToken": "eyJhbGci...(mới)"
  }
}
```

**Logic:** Refresh token rotation — token cũ bị xóa, tạo token mới.

---

### POST `/api/auth/logout` — Đăng xuất
**Auth:** 🔒 Bearer Token

**Response:** `{ "success": true, "message": "Đăng xuất thành công" }`

**Logic:** Xóa tất cả refresh tokens của user.

---

### GET `/api/auth/me` — Thông tin user hiện tại
**Auth:** 🔒 Bearer Token

**Response (200):**
```json
{
  "success": true,
  "data": {
    "id": "e0000000-0000-0000-0000-000000000001",
    "full_name": "Lý Văn Tám",
    "phone": "0900000008",
    "email": "tam@gmail.com",
    "avatar_url": null,
    "address": "Ấp Phú Hòa, Phú Tân",
    "province_id": 1,
    "district_id": 10,
    "ward_id": 11,
    "status": "active",
    "roles": [
      { "id": 6, "name": "farmer", "display_name": "Nông dân" }
    ],
    "permissions": [
      { "id": 10, "name": "product.view", "module": "product" },
      { "id": 17, "name": "order.view", "module": "order" },
      { "id": 18, "name": "order.create", "module": "order" },
      { "id": 26, "name": "booking.view", "module": "booking" },
      { "id": 27, "name": "booking.create", "module": "booking" },
      { "id": 29, "name": "booking.cancel", "module": "booking" },
      { "id": 31, "name": "ai.analyze", "module": "ai" },
      { "id": 32, "name": "ai.view_history", "module": "ai" }
    ]
  }
}
```

---

## 👤 2. Users (`/api/users`)

### GET `/api/users` — DS người dùng
**Auth:** 🔒 Bearer Token | **Query:** `?page=1&limit=10&status=active`

### GET `/api/users/:id` — Chi tiết user

### PUT `/api/users/:id` — Cập nhật profile
**Body:**
```json
{
  "full_name": "Lý Văn Tám",
  "avatar_url": "/avatars/tam-new.jpg",
  "address": "Ấp Phú Hòa, Phú Tân, An Giang",
  "province_id": 1,
  "district_id": 10,
  "ward_id": 11
}
```

---

## 🏪 3. Stores (`/api/stores`)

### GET `/api/stores` — DS cửa hàng
**Auth:** Không cần | **Query:** `?page=1&limit=10&status=active`

**Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": "11111111-1111-1111-1111-111111111111",
      "name": "Vật tư Nông nghiệp Mai Phát",
      "phone": "0297 386 1234",
      "address": "45 Nguyễn Huệ, Châu Đốc, An Giang",
      "province_id": 1,
      "district_id": 2,
      "ward_id": 3,
      "latitude": 10.7069,
      "longitude": 105.1178,
      "logo_url": "/logos/mai-phat.jpg",
      "description": "Chuyên cung cấp thuốc BVTV...",
      "status": "active",
      "owner": {
        "id": "b0000000-0000-0000-0000-000000000001",
        "full_name": "Trần Thị Mai",
        "phone": "0900000002"
      }
    }
  ],
  "pagination": { "page": 1, "limit": 10, "totalItems": 2, "totalPages": 1 }
}
```

### GET `/api/stores/:id` — Chi tiết cửa hàng (kèm staff)

### POST `/api/stores` — Tạo cửa hàng
**Auth:** 🔒 Role: `store_owner`, `admin`

**Body:**
```json
{
  "name": "Cửa hàng ABC",
  "phone": "0900123456",
  "address": "123 Nguyễn Huệ",
  "province_id": 1,
  "district_id": 1,
  "ward_id": 1,
  "latitude": 10.7069,
  "longitude": 105.1178,
  "logo_url": "/logos/abc.jpg",
  "description": "Mô tả cửa hàng"
}
```

### PUT `/api/stores/:id` — Cập nhật (chỉ chủ cửa hàng)

### POST `/api/stores/:id/staff` — Thêm nhân viên
**Body:** `{ "user_id": "uuid...", "position": "Nhân viên bán hàng" }`

---

## 📦 4. Products (`/api/products`)

### GET `/api/products` — DS sản phẩm
**Auth:** Không bắt buộc  
**Query:** `?page=1&limit=10&status=active&store_id=uuid&category_id=1&search=Actara`

**Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": "a1000000-0000-0000-0000-000000000001",
      "sku": "TTS-001",
      "name": "Actara 25WG - Thuốc trừ sâu",
      "slug": "actara-25wg",
      "description": "Thuốc trừ sâu phổ rộng...",
      "usage_instructions": "Pha 1g/8 lít nước...",
      "unit": "gói 1g",
      "price": "8000.00",
      "original_price": "10000.00",
      "manufacturer": "Syngenta",
      "origin_country": "Thụy Sĩ",
      "expiry_months": 24,
      "status": "active",
      "images": [
        { "id": 1, "image_url": "/products/actara-25wg-1.jpg", "is_primary": true, "sort_order": 0 },
        { "id": 2, "image_url": "/products/actara-25wg-2.jpg", "is_primary": false, "sort_order": 1 }
      ],
      "category": { "id": 1, "name": "Thuốc trừ sâu", "slug": "thuoc-tru-sau" },
      "store": { "id": "11111111-...", "name": "Vật tư NN Mai Phát" }
    }
  ],
  "pagination": { "page": 1, "limit": 10, "totalItems": 16, "totalPages": 2 }
}
```

### GET `/api/products/:id` — Chi tiết SP (kèm inventory, store info)

### POST `/api/products` — Tạo sản phẩm
**Auth:** 🔒 Bearer Token

**Body:**
```json
{
  "store_id": "11111111-1111-1111-1111-111111111111",
  "category_id": 1,
  "sku": "TTS-003",
  "name": "Sản phẩm mới",
  "slug": "san-pham-moi",
  "description": "Mô tả sản phẩm",
  "usage_instructions": "Hướng dẫn sử dụng",
  "unit": "chai 100ml",
  "price": 50000,
  "original_price": 60000,
  "manufacturer": "ABC",
  "origin_country": "Việt Nam",
  "expiry_months": 24,
  "initial_quantity": 100,
  "images": [
    { "image_url": "/products/sp-moi-1.jpg" },
    { "image_url": "/products/sp-moi-2.jpg" }
  ]
}
```

**Logic:**
- Tự tạo ProductImage records (ảnh đầu = `is_primary: true`)
- Tự tạo Inventory record với `initial_quantity`

### PUT `/api/products/:id` — Cập nhật SP
### DELETE `/api/products/:id` — Xóa SP (soft delete → `status: 'inactive'`)

---

## 📂 5. Categories (`/api/categories`)

### GET `/api/categories` — DS danh mục (cây thư mục)
**Auth:** Không cần

**Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": 1, "name": "Thuốc trừ sâu", "slug": "thuoc-tru-sau", "parent_id": null,
      "icon": null, "sort_order": 1, "status": "active"
    },
    {
      "id": 4, "name": "Phân bón", "slug": "phan-bon", "parent_id": null,
      "children": [
        { "id": 9, "name": "Phân NPK", "slug": "phan-npk", "parent_id": 4 },
        { "id": 10, "name": "Phân Urê", "slug": "phan-ure", "parent_id": 4 },
        { "id": 11, "name": "Phân DAP", "slug": "phan-dap", "parent_id": 4 },
        { "id": 12, "name": "Phân Kali", "slug": "phan-kali", "parent_id": 4 },
        { "id": 13, "name": "Phân hữu cơ", "slug": "phan-huu-co", "parent_id": 4 }
      ]
    }
  ]
}
```

### GET `/api/categories/:id` — Chi tiết danh mục
### POST `/api/categories` — Tạo danh mục (🔒 admin, store_owner)
### PUT `/api/categories/:id` — Cập nhật danh mục

---

## 📊 6. Inventory (`/api/inventory`)

### GET `/api/inventory` — Danh sách tồn kho
**Auth:** 🔒 | **Query:** `?page=1&limit=10&store_id=uuid`

**Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "product_id": "a1000000-0000-0000-0000-000000000001",
      "store_id": "11111111-1111-1111-1111-111111111111",
      "quantity": 500,
      "min_quantity": 50,
      "product": {
        "id": "a1000000-0000-0000-0000-000000000001",
        "name": "Actara 25WG",
        "sku": "TTS-001",
        "unit": "gói 1g",
        "price": "8000.00"
      }
    }
  ]
}
```

### POST `/api/inventory/import` — Nhập kho
**Auth:** 🔒

**Body:**
```json
{
  "product_id": "a1000000-0000-0000-0000-000000000001",
  "store_id": "11111111-1111-1111-1111-111111111111",
  "quantity": 200,
  "unit_price": 6000,
  "supplier_id": 1,
  "batch_number": "LOT-2026-005",
  "expiry_date": "2028-01-01",
  "note": "Nhập lô hàng mới"
}
```

**Logic:** Tăng `quantity` trong Inventory + tạo InventoryMovement (type: `import`)

### POST `/api/inventory/export` — Xuất kho
**Body:** Tương tự nhập kho, `quantity` sẽ bị trừ

### PUT `/api/inventory/adjust` — Điều chỉnh tồn kho (kiểm kê)
**Auth:** 🔒
**Body:**
```json
{
  "product_id": "a1000000-0000-0000-0000-000000000001",
  "store_id": "11111111-1111-1111-1111-111111111111",
  "new_quantity": 480,
  "note": "Kiểm kê hụt 20 sản phẩm"
}
```
**Logic:** Thay thế `quantity` hiện tại bằng `new_quantity` + tạo InventoryMovement (type: `adjustment`) để ghi vết khoản chênh lệch.

### PUT `/api/inventory/:id/min-quantity` — Cập nhật ngưỡng tối thiểu
**Auth:** 🔒 Role: `store_owner` (phải là chủ cửa hàng) hoặc `admin`
**Body:**
```json
{
  "min_quantity": 100
}
```
**Logic:** Cập nhật `min_quantity` cảnh báo tồn kho thấp. Response trả về boolean `is_low_stock`.

### GET `/api/inventory/history` — Lịch sử nhập/xuất
**Query:** `?page=1&limit=10&store_id=uuid&type=import`

---

## 🛒 7. Orders (`/api/orders`)

### GET `/api/orders` — DS đơn hàng
**Auth:** 🔒 | **Query:** `?page=1&limit=10&status=pending&store_id=uuid&customer_id=uuid`

**Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": "00d00000-0000-0000-0000-000000000001",
      "order_code": "DH20260201-0001",
      "store_id": "11111111-1111-1111-1111-111111111111",
      "customer_name": "Lý Văn Tám",
      "customer_phone": "0900000008",
      "shipping_address": "Ấp Phú Hòa, Phú Tân, An Giang",
      "subtotal": "546000.00",
      "discount_amount": "0.00",
      "shipping_fee": "30000.00",
      "total_amount": "576000.00",
      "status": "delivered",
      "payment_status": "paid",
      "payment_method": "cash",
      "note": "Giao buổi sáng",
      "store": { "id": "11111111-...", "name": "Vật tư NN Mai Phát" },
      "customer": { "id": "e0000000-...", "full_name": "Lý Văn Tám", "phone": "0900000008" }
    }
  ]
}
```

### GET `/api/orders/:id` — Chi tiết đơn (kèm items, payments)

**Response (200):**
```json
{
  "success": true,
  "data": {
    "id": "00d00000-0000-0000-0000-000000000001",
    "order_code": "DH20260201-0001",
    "subtotal": "546000.00",
    "total_amount": "576000.00",
    "status": "delivered",
    "items": [
      {
        "id": 1,
        "product_name": "Actara 25WG",
        "product_sku": "TTS-001",
        "unit": "gói 1g",
        "quantity": 12,
        "unit_price": "8000.00",
        "discount_amount": "0.00",
        "total_price": "96000.00",
        "product": { "id": "a1000000-...", "name": "Actara 25WG", "sku": "TTS-001" }
      }
    ],
    "payments": [
      {
        "id": "0a000000-0000-0000-0000-000000000001",
        "amount": "576000.00",
        "payment_method": "cash",
        "status": "paid",
        "paid_at": "2026-02-01T10:30:00.000Z"
      }
    ]
  }
}
```

### POST `/api/orders` — Tạo đơn hàng
**Auth:** 🔒

**Body:**
```json
{
  "store_id": "11111111-1111-1111-1111-111111111111",
  "customer_name": "Lý Văn Tám",
  "customer_phone": "0900000008",
  "shipping_address": "Ấp Phú Hòa, Phú Tân, An Giang",
  "payment_method": "cash",
  "discount_amount": 0,
  "shipping_fee": 30000,
  "note": "Giao buổi sáng",
  "items": [
    { "product_id": "a1000000-0000-0000-0000-000000000001", "quantity": 12 },
    { "product_id": "a1000000-0000-0000-0000-000000000006", "quantity": 1 }
  ]
}
```

**Logic:**
- Validate từng product, lấy giá hiện tại
- Tính `subtotal`, `total_amount` tự động
- Snapshot `product_name`, `product_sku`, `unit_price` vào order_item
- Tạo order_code: `DH{YYYYMMDD}{4_random_digits}`

### PUT `/api/orders/:id/status` — Cập nhật trạng thái
**Body:** `{ "status": "confirmed" }`

| Status flow | Mô tả |
|-------------|-------|
| `pending` → `confirmed` | Xác nhận đơn |
| `confirmed` → `shipping` | Đang giao |
| `shipping` → `delivered` | Đã giao |
| `pending/confirmed` → `cancelled` | Hủy đơn |

---

## 🚁 8. Drones (`/api/drones`)

### GET `/api/drones` — DS drone
**Auth:** Không bắt buộc | **Query:** `?page=1&limit=10&province_id=1&owner_id=uuid&status=active`

**Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": "dd000000-0000-0000-0000-000000000001",
      "name": "DJI Agras T40 - Sơn AG01",
      "model": "DJI Agras T40",
      "registration_number": "AG-DRONE-001",
      "capacity_liters": 40,
      "spray_width_meters": "11.00",
      "max_area_per_flight": "8.50",
      "description": "Drone nông nghiệp DJI Agras T40...",
      "images": ["\/drones\/agras-t40-1.jpg"],
      "province_id": 1,
      "district_id": 3,
      "latitude": 10.5894,
      "longitude": 105.2317,
      "status": "active",
      "rating_avg": "4.75",
      "total_bookings": 15,
      "owner": { "id": "d0000000-...", "full_name": "Huỳnh Thanh Sơn", "phone": "0900000006" },
      "services": [
        { "id": 1, "service_type": "spraying", "name": "Phun thuốc trừ sâu/bệnh", "price_per_hectare": "150000.00", "min_area": "0.50", "status": "active" },
        { "id": 2, "service_type": "fertilizing", "name": "Phun phân bón lá", "price_per_hectare": "120000.00", "min_area": "0.50", "status": "active" },
        { "id": 3, "service_type": "seeding", "name": "Sạ lúa giống", "price_per_hectare": "200000.00", "min_area": "1.00", "status": "active" }
      ]
    }
  ]
}
```

### GET `/api/drones/:id` — Chi tiết (kèm services, schedules)
### POST `/api/drones` — Đăng ký drone (🔒)
### PUT `/api/drones/:id` — Cập nhật (🔒)

---

## 📅 9. Bookings (`/api/bookings`)

### GET `/api/bookings` — DS lịch đặt drone
**Auth:** 🔒 | **Query:** `?page=1&limit=10&status=pending&drone_id=uuid&farmer_id=uuid`

**Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": "bb000000-0000-0000-0000-000000000001",
      "booking_code": "BK20260201-0001",
      "scheduled_date": "2026-02-01",
      "scheduled_time": "07:00",
      "area_hectares": "2.50",
      "field_address": "Ấp Phú Hòa, Phú Tân",
      "crop_type": "Lúa OM18",
      "pesticide_provided": true,
      "pesticide_details": "Actara 25WG - 30 gói",
      "price_per_hectare": "150000.00",
      "total_amount": "375000.00",
      "status": "completed",
      "payment_status": "paid",
      "completed_at": "2026-02-01T09:30:00.000Z",
      "actual_area": "2.50",
      "drone": { "id": "dd000000-...", "name": "DJI Agras T40", "model": "DJI Agras T40" },
      "service": { "id": 1, "name": "Phun thuốc trừ sâu/bệnh", "service_type": "spraying" },
      "farmer": { "id": "e0000000-...", "full_name": "Lý Văn Tám", "phone": "0900000008" }
    }
  ]
}
```

### GET `/api/bookings/:id` — Chi tiết (kèm drone owner, field, status logs, review)

### POST `/api/bookings` — Đặt lịch drone
**Auth:** 🔒

**Body:**
```json
{
  "drone_id": "dd000000-0000-0000-0000-000000000001",
  "drone_service_id": 1,
  "field_id": "ff000000-0000-0000-0000-000000000001",
  "scheduled_date": "2026-02-20",
  "scheduled_time": "07:00",
  "area_hectares": 2.5,
  "field_address": "Ấp Phú Hòa, Phú Tân",
  "field_latitude": 10.5123,
  "field_longitude": 105.3456,
  "crop_type": "Lúa OM18",
  "pesticide_provided": true,
  "pesticide_details": "Actara 25WG - 30 gói",
  "price_per_hectare": 150000,
  "total_amount": 375000,
  "note": "Phun sáng sớm"
}
```

**Logic:**
- Tự tạo `booking_code`: `BK{YYYYMMDD}{4_digits}`
- Tự gán `farmer_id` từ user đăng nhập
- Tạo BookingStatusLog (null → pending)

### PUT `/api/bookings/:id/status` — Cập nhật trạng thái
**Body:**
```json
{
  "status": "confirmed",
  "note": "Chủ drone xác nhận",
  "actual_area": 2.5          // chỉ khi status = completed
}
```

| Status flow | Ai thực hiện |
|-------------|-------------|
| `pending` → `confirmed` | Chủ drone |
| `confirmed` → `in_progress` | Chủ drone |
| `in_progress` → `completed` | Chủ drone |
| `pending` → `cancelled` | Nông dân |

**Logic:**
- Tạo BookingStatusLog cho mỗi lần đổi trạng thái
- Khi `completed`: ghi `completed_at`, `actual_area`, tăng `drone.total_bookings`

### POST `/api/bookings/:id/review` — Đánh giá
**Auth:** 🔒

**Body:**
```json
{
  "rating": 5,
  "comment": "Phun rất đều, nhiệt tình!",
  "images": ["/reviews/bk-review-1.jpg"]
}
```

**Logic:**
- Chỉ đánh giá booking `completed`
- Mỗi booking chỉ đánh giá 1 lần
- Tự động tính lại `rating_avg` của drone

---

## 🤖 10. AI Analysis (`/api/ai`)

### GET `/api/ai/diseases` — DS bệnh trong thư viện
**Auth:** Không cần

**Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "name_vi": "Đạo ôn lúa",
      "name_en": "Rice Blast",
      "scientific_name": "Magnaporthe oryzae",
      "slug": "dao-on-lua",
      "description": "Bệnh nấm phổ biến...",
      "symptoms": "Vết bệnh hình mắt én...",
      "causes": "Do nấm Magnaporthe oryzae...",
      "prevention": "Chọn giống kháng...",
      "treatment": "Phun thuốc đặc trị...",
      "images": ["/diseases/dao-on-1.jpg"],
      "severity_level": "high",
      "affected_crops": ["Lúa"],
      "status": "active",
      "treatments": [
        {
          "product_id": "a1000000-0000-0000-0000-000000000003",
          "dosage": "3g/bình 16 lít",
          "application_method": "Phun khi bệnh mới xuất hiện...",
          "effectiveness": "high",
          "priority": 1
        }
      ]
    }
  ]
}
```

### GET `/api/ai/diseases/:id` — Chi tiết bệnh

### POST `/api/ai/diseases` — Tạo bệnh mới (🔒 admin only)

### POST `/api/ai/analyze` — Phân tích ảnh
**Auth:** 🔒

**Body:**
```json
{
  "image_url": "/uploads/analysis/my-image.jpg",
  "field_id": "ff000000-0000-0000-0000-000000000001",
  "location_latitude": 10.5123,
  "location_longitude": 105.3456,
  "note": "Lá lúa có vết lạ"
}
```

**Response (201):**
```json
{
  "success": true,
  "message": "Đang phân tích ảnh",
  "data": {
    "id": "aa000000-0000-0000-0000-000000000004",
    "status": "processing",
    "image_url": "/uploads/analysis/my-image.jpg",
    "detected_disease_id": null,
    "confidence_score": null,
    "ai_response": null
  }
}
```

### GET `/api/ai/history` — Lịch sử phân tích
**Auth:** 🔒 | **Query:** `?page=1&limit=10`

**Response (200):** Trả về danh sách phân tích kèm thông tin bệnh detected
```json
{
  "data": [
    {
      "id": "aa000000-...",
      "image_url": "/uploads/analysis/img-001.jpg",
      "detected_disease_id": 1,
      "confidence_score": 0.9234,
      "status": "completed",
      "ai_response": {
        "disease": "Đạo ôn lúa",
        "confidence": 0.9234,
        "severity": "moderate",
        "recommendations": ["Phun Nativo 750WG ngay", "Giảm bón đạm"]
      },
      "disease": { "id": 1, "name_vi": "Đạo ôn lúa" }
    }
  ]
}
```

---

## 🌾 11. Fields (`/api/fields`)

### GET `/api/fields` — DS ruộng
**Auth:** 🔒

**Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": "ff000000-0000-0000-0000-000000000001",
      "name": "Ruộng Phú Hòa - Thửa 1",
      "area_hectares": "2.50",
      "address": "Ấp Phú Hòa, Phú Tân, An Giang",
      "latitude": 10.5123,
      "longitude": 105.3456,
      "polygon_coordinates": [[10.512,105.344],[10.514,105.344],[10.514,105.347],[10.512,105.347]],
      "soil_type": "Phù sa",
      "irrigation_type": "Tưới ngập",
      "status": "active",
      "seasons": [
        {
          "id": "f5000000-...",
          "season_id": 4,
          "crop_type": "Lúa",
          "seed_variety": "OM18",
          "planting_date": "2025-11-20",
          "harvest_date": "2026-03-01",
          "expected_yield": "6.50",
          "status": "growing"
        }
      ]
    }
  ]
}
```

### GET `/api/fields/:id` — Chi tiết ruộng

### POST `/api/fields` — Tạo ruộng
**Body:**
```json
{
  "name": "Ruộng mới",
  "area_hectares": 2.0,
  "address": "Ấp ABC, Phú Tân",
  "province_id": 1,
  "district_id": 10,
  "ward_id": 11,
  "latitude": 10.5123,
  "longitude": 105.3456,
  "polygon_coordinates": [[10.512,105.344],[10.514,105.344],[10.514,105.347],[10.512,105.347]],
  "soil_type": "Phù sa",
  "irrigation_type": "Tưới ngập"
}
```

### PUT `/api/fields/:id` — Cập nhật ruộng

### POST `/api/fields/:fieldId/seasons` — Thêm mùa vụ cho ruộng
**Body:**
```json
{
  "season_id": 4,
  "crop_type": "Lúa",
  "seed_variety": "OM18",
  "planting_date": "2025-11-20",
  "harvest_date": "2026-03-01",
  "expected_yield": 6.5,
  "note": "Vụ Đông Xuân 2026"
}
```

---

## 🔔 12. Notifications (`/api/notifications`)

### GET `/api/notifications` — DS thông báo
**Auth:** 🔒 | **Query:** `?page=1&limit=20`

**Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": "cc000000-0000-0000-0000-000000000003",
      "title": "Lịch đặt drone đã xác nhận",
      "body": "Chủ drone đã xác nhận lịch phun ngày 15/02/2026",
      "type": "booking",
      "data": { "booking_id": "bb000000-...", "status": "confirmed" },
      "is_read": false,
      "read_at": null,
      "created_at": "2026-02-15T08:00:00.000Z"
    }
  ]
}
```

### GET `/api/notifications/unread-count` — Số thông báo chưa đọc
**Response:** `{ "success": true, "data": { "count": 3 } }`

### PUT `/api/notifications/:id/read` — Đánh dấu đã đọc
### PUT `/api/notifications/read-all` — Đánh dấu tất cả đã đọc

---

## 📤 13. Upload (`/api/upload`)

### POST `/api/upload` — Upload 1 file
**Auth:** 🔒 Bearer Token

**Body:** `multipart/form-data`
| Field | Type | Mô tả |
|-------|------|-------|
| `file` | File | **Bắt buộc** — File ảnh (JPG, PNG, WebP, GIF) |
| `category` | String | Optional — Thư mục phân loại: `products`, `avatars`, `drones`, `diseases`, `reviews`, `general` |

**Giới hạn:** 5MB / file

**Response (200):**
```json
{
  "success": true,
  "message": "Upload thành công",
  "data": {
    "url": "/uploads/products/2026-02/a1b2c3d4-1708876800.jpg",
    "originalName": "san-pham.jpg",
    "size": 245000,
    "mimetype": "image/jpeg"
  }
}
```

**Logic:**
- Chỉ cho phép: `image/jpeg`, `image/png`, `image/webp`, `image/gif`
- Tên file tự động tạo: `UUID-timestamp.ext` (chống trùng, chống path traversal)
- Lưu theo: `/uploads/{category}/{YYYY-MM}/`

**Lỗi:**
| Trường hợp | Status | Message |
|-----------|--------|---------|
| Không có file | 400 | `Không có file nào được upload` |
| Sai MIME type | 400 | `Chỉ cho phép upload file ảnh (JPG, PNG, WebP, GIF)` |
| Quá 5MB | 400 | `File quá lớn. Tối đa 5MB` |

---

### POST `/api/upload/multiple` — Upload nhiều file
**Auth:** 🔒 Bearer Token

**Body:** `multipart/form-data`
| Field | Type | Mô tả |
|-------|------|-------|
| `files` | File[] | **Bắt buộc** — Tối đa 5 file ảnh |
| `category` | String | Optional |

**Response (200):**
```json
{
  "success": true,
  "message": "Upload thành công 3 file",
  "data": {
    "urls": [
      { "url": "/uploads/products/2026-02/uuid1.jpg", "originalName": "img1.jpg", "size": 120000, "mimetype": "image/jpeg" },
      { "url": "/uploads/products/2026-02/uuid2.png", "originalName": "img2.png", "size": 85000, "mimetype": "image/png" }
    ]
  }
}
```

---

## 📊 14. Stats — Super Admin (`/api/stats/admin`)

### GET `/api/stats/admin/overview` — Tổng quan hệ thống
**Auth:** 🔒 Role: `admin`

**Response (200):**
```json
{
  "success": true,
  "message": "Thống kê tổng quan hệ thống",
  "data": {
    "users": { "total": 10, "active": 9 },
    "stores": { "total": 2, "active": 2 },
    "orders": { "total": 8, "thisMonth": 3 },
    "bookings": { "total": 6, "completed": 4 },
    "revenue": {
      "month": "2026-02",
      "orderRevenue": 2500000,
      "bookingRevenue": 1875000,
      "totalRevenue": 4375000
    },
    "extras": {
      "activeDrones": 3,
      "totalProducts": 16,
      "totalAnalyses": 5
    }
  }
}
```

---

## 🏪 15. Stats — Store Owner (`/api/stats/store`)

### GET `/api/stats/store/revenue` — Doanh thu theo tháng
**Auth:** 🔒 Role: `store_owner`, `admin`
**Query:** Không cần (Tự nhận diện `store_id` từ JWT)

> 💡 Trả về doanh thu từng tháng trong **năm hiện tại**. Logic GROUP BY tối ưu.

**Response (200):**
```json
{
  "success": true,
  "message": "Doanh thu cửa hàng theo tháng",
  "data": {
    "storeName": "Vật tư NN Mai Phát",
    "year": 2026,
    "totalRevenue": 12500000,
    "totalOrders": 45,
    "chart": [
      { "month": "01", "label": "T1/2026", "monthNum": 1, "year": 2026, "revenue": 850000, "orderCount": 3 },
      { "month": "02", "label": "T2/2026", "monthNum": 2, "year": 2026, "revenue": 1200000, "orderCount": 5 },
      "...đủ 12 tháng"
    ]
  }
}
```

> 💡 Flutter dùng `chart` array để vẽ `BarChart` hoặc `LineChart`. Các tháng chưa có dữ liệu sẽ trả về `0`.

---

### GET `/api/stats/store/products` — Top SP + cảnh báo tồn kho
**Auth:** 🔒 Role: `store_owner`, `admin`
**Query:** Không cần (Tự nhận diện `store_id` từ JWT)

**Response (200):**
```json
{
  "success": true,
  "message": "Thống kê sản phẩm cửa hàng",
  "data": {
    "storeName": "Vật tư NN Mai Phát",
    "topProducts": [
      { "rank": 1, "productId": "uuid", "name": "Actara 25WG", "sku": "TTS-001", "totalSold": 120, "totalRevenue": 960000 },
      { "rank": 2, "productId": "uuid", "name": "Nativo 750WG", "sku": "TTS-002", "totalSold": 85, "totalRevenue": 1360000 }
    ],
    "lowStock": [
      { 
        "inventoryId": 12,
        "productId": "uuid", "name": "Nativo 750WG", "sku": "TTS-002", "unit": "gói 12g", 
        "quantity": 5, "minQuantity": 50, "deficit": 45
      }
    ],
    "expiringProducts": [
      { "id": "uuid", "name": "Anvil 5SC", "sku": "TTN-001", "unit": "chai 250ml", "expiryDate": "2026-04-15", "daysLeft": 49 }
    ]
  }
}
```

| Field | Mô tả |
|-------|-------|
| `topProducts` | Top 5 SP bán chạy theo SUM(quantity) từ đơn 'delivered' |
| `lowStock` | SP có tồn kho ≤ min_quantity. `deficit` là số lượng cần nhập thêm. |
| `expiringProducts` | SP còn < 90 ngày hết hạn. |

---

## 🚁 16. Stats — Drone Owner (`/api/stats/drone`)

### GET `/api/stats/drone/bookings` — Thống kê chuyến bay drone
**Auth:** 🔒 Role: `drone_owner`, `admin`
**Query:** `?season_id=4` (optional — filter theo mùa vụ)

> Controller tự lấy TẤT CẢ drone thuộc user đăng nhập.

**Response (200):**
```json
{
  "success": true,
  "message": "Thống kê drone của bạn",
  "data": {
    "totalDrones": 2,
    "totalFlights": 15,
    "completedFlights": 12,
    "totalRevenue": 3750000,
    "totalAreaHectares": 25.5,
    "byStatus": {
      "completed": 12,
      "cancelled": 1,
      "pending": 2
    },
    "byServiceType": [
      { "serviceType": "spraying", "serviceName": "Phun thuốc trừ sâu/bệnh", "count": 8, "revenue": 2400000, "areaHectares": 18.0 },
      { "serviceType": "fertilizing", "serviceName": "Phun phân bón lá", "count": 4, "revenue": 1350000, "areaHectares": 7.5 }
    ],
    "droneDetails": [
      {
        "droneId": "dd000000-...",
        "name": "DJI Agras T40 - Sơn AG01",
        "model": "DJI Agras T40",
        "status": "active",
        "ratingAvg": 4.75,
        "completedFlights": 8,
        "revenue": 2500000,
        "areaHectares": 17.0
      }
    ],
    "filter": {
      "seasonId": 4
    }
  }
}
```

| Field | Mô tả |
|-------|-------|
| `byStatus` | Số booking theo từng trạng thái |
| `byServiceType` | Thống kê theo loại dịch vụ (spraying/fertilizing/seeding) |
| `droneDetails` | Chi tiết từng drone: flights, revenue, area |
| `filter.seasonId` | Nếu có filter theo mùa vụ |

---

## 🏢 17. Suppliers (`/api/suppliers`)

### GET `/api/suppliers` — Danh sách nhà cung cấp
**Auth:** 🔒 Role: `store_owner`, `admin`
**Query:** `?page=1&limit=10&store_id=uuid&name=abc&status=active`

**Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "store_id": "11111111-1111-1111-1111-111111111111",
      "name": "Công ty BVTV An Giang",
      "phone": "0901234567",
      "email": "contact@bvtv-ag.com",
      "address": "Long Xuyên, An Giang",
      "tax_code": "0101234567",
      "status": "active"
    }
  ],
  "pagination": {
    "total": 1,
    "page": 1,
    "limit": 10,
    "total_pages": 1
  }
}
```

### POST `/api/suppliers` — Tạo nhà cung cấp
**Auth:** 🔒 Role: `store_owner`, `admin`
**Body:**
```json
{
  "store_id": "11111111-1111-1111-1111-111111111111",
  "name": "Công ty BVTV An Giang",
  "phone": "0901234567",
  "email": "contact@bvtv-ag.com",
  "address": "Long Xuyên, An Giang",
  "tax_code": "0101234567"
}
```

### GET `/api/suppliers/:id` — Chi tiết nhà cung cấp
### PUT `/api/suppliers/:id` — Cập nhật nhà cung cấp
### DELETE `/api/suppliers/:id` — Xóa nhà cung cấp (Soft delete)

---

## 🗺️ 18. Locations (`/api`)

### GET `/api/provinces` — DS tỉnh/TP
```json
{ "data": [{ "id": 1, "name": "An Giang", "code": "AG" }] }
```

### GET `/api/provinces/:id/districts` — DS huyện theo tỉnh
```json
{ "data": [{ "id": 1, "province_id": 1, "name": "Long Xuyên", "code": "LX" }] }
```

### GET `/api/districts/:id/wards` — DS xã theo huyện
```json
{ "data": [{ "id": 1, "district_id": 1, "name": "Mỹ Bình", "code": "MB" }] }
```

### GET `/api/seasons` — DS mùa vụ
**Query:** `?year=2026`
```json
{ "data": [{ "id": 4, "name": "Đông Xuân", "year": 2026, "start_date": "2025-11-15", "end_date": "2026-03-15" }] }
```

### GET `/api/health` — Health check
```json
{ "success": true, "message": "DeHat VN API is running", "timestamp": "2026-02-15T08:00:00.000Z" }
```

---

## 📋 19. Tài khoản test mẫu

| Email | Phone | Vai trò | Password |
|-------|-------|---------|----------|
| `admin@dehat.vn` | `0900000001` | Super Admin | `123456` |
| `mai.tran@dehat.vn` | `0900000002` | Chủ cửa hàng | `123456` |
| `phuc.le@dehat.vn` | `0900000003` | Chủ cửa hàng | `123456` |
| `tuan.pham@dehat.vn` | `0900000004` | NV cửa hàng | `123456` |
| `son.huynh@dehat.vn` | `0900000006` | Chủ drone | `123456` |
| `duc.ngo@dehat.vn` | `0900000007` | Chủ drone | `123456` |
| `tam.ly@dehat.vn` | `0900000008` | Nông dân | `123456` |
| `lan.dang@dehat.vn` | `0900000009` | Nông dân | `123456` |
| `hung.bui@dehat.vn` | `0900000010` | Nông dân | `123456` |
| `bao.trinh@dehat.vn` | `0900000011` | Nông dân | `123456` |

> **Đăng nhập bằng phone + password**

---

## 🎨 20. ENUMs Reference

| ENUM | Values |
|------|--------|
| `user_status` | `active`, `inactive`, `blocked` |
| `store_status` | `active`, `inactive`, `pending` |
| `order_status` | `pending`, `confirmed`, `processing`, `shipping`, `delivered`, `cancelled`, `returned` |
| `payment_status` | `unpaid`, `paid`, `refunded`, `failed` |
| `payment_method` | `cash`, `transfer`, `momo`, `vnpay`, `zalopay` |
| `booking_status` | `pending`, `confirmed`, `in_progress`, `completed`, `cancelled` |
| `booking_payment_status` | `unpaid`, `deposit`, `paid`, `refunded` |
| `service_type` | `spraying`, `fertilizing`, `seeding` |
| `drone_status` | `active`, `inactive`, `maintenance` |
| `analysis_status` | `processing`, `completed`, `failed` |
| `field_season_status` | `planning`, `growing`, `harvested`, `cancelled` |
| `severity_level` | `low`, `medium`, `high`, `critical` |
| `effectiveness_level` | `low`, `medium`, `high` |
| `movement_type` | `import`, `export`, `adjustment`, `return` |
| `device_type` | `ios`, `android`, `web` |
| `token_type` | `refresh`, `reset_password`, `verify_email` |
