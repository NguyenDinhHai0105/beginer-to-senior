# Quy tắc bắt buộc khi viết DDL cho một bảng

Tệp này tóm tắt những quy tắc không được bỏ sót khi thiết kế bảng trong cơ sở dữ liệu. Mỗi mục có giải thích ngắn và ví dụ SQL cụ thể để minh họa.

---

## 1. Các cột mặc định (standard audit columns)
Mọi bảng nên có tối thiểu 5–7 cột audit để theo dõi lịch sử và hỗ trợ optimistic locking:

- `created_at` (TIMESTAMP WITH TIME ZONE): thời điểm bản ghi được tạo; sau khi tạo không chỉnh sửa (enforced ở ứng dụng hoặc trigger).
- `updated_at` (TIMESTAMP WITH TIME ZONE): thời điểm bản ghi được cập nhật; cập nhật tự động khi thay đổi.
- `created_by` (BIGINT): id người tạo (nếu có).
- `updated_by` (BIGINT): id người cập nhật (nếu có).
- `version` (INTEGER/BIGINT): phiên bản cho optimistic locking.
- `is_deleted` (BOOLEAN) và `deleted_at` (TIMESTAMP) — nếu áp dụng xóa mềm (soft delete).

Ví dụ (Postgres):

```sql
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  usr_username VARCHAR(50) NOT NULL,
  usr_email VARCHAR(255) NOT NULL,
  usr_full_name VARCHAR(150),

  created_at TIMESTAMPTZ NOT NULL DEFAULT now(), -- thời điểm tạo
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(), -- cập nhật khi thay đổi
  created_by BIGINT NULL,
  updated_by BIGINT NULL,
  version INTEGER NOT NULL DEFAULT 1,

  is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
  deleted_at TIMESTAMPTZ NULL
);
```

Ghi chú: `created_at` thường không bị sửa về phía ứng dụng; `updated_at` được cập nhật tự động bằng trigger hoặc ORM.

---

## 2. Comment cho cột (column comments)
Ghi comment cho các cột quan trọng để người đọc hiểu ý nghĩa, ví dụ:

```sql
COMMENT ON COLUMN users.usr_username IS 'Tên đăng nhập (unique identifier cho user)';
```

Comments giúp team mới nhanh hiểu dữ liệu và tránh nhầm lẫn.

---

## 3. Xóa mềm (Soft delete)
Không xóa hoàn toàn bản ghi trừ khi thực sự cần. Thêm cột `is_deleted` và `deleted_at` để đánh dấu:

```sql
ALTER TABLE users
  ADD COLUMN is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
  ADD COLUMN deleted_at TIMESTAMPTZ NULL;
```

Truy vấn mặc định nên lọc bỏ `is_deleted = true` (ở tầng ứng dụng hoặc view). Nếu cần, tạo view `active_users` chỉ chứa bản ghi chưa xóa.

---

## 4. Tiền tố cột (Column name prefix)
Đặt tiền tố viết tắt của bảng trước cột (ví dụ `usr_` cho bảng `users`) để tránh trùng tên khi join nhiều bảng:

- users: `usr_id`, `usr_email`
- feed: `fd_id`, `fd_title`

Ví dụ:

```sql
-- thay vì `email` dùng `usr_email`
SELECT u.usr_email, p.prd_name
FROM users u
JOIN purchases p ON u.id = p.user_id;
```

---

## 5. Tách bảng khi quá nhiều cột (Vertical partition / separation)
Nếu một bảng có tập cột lớn, tách những phần ít truy vấn chung thành bảng con để giảm I/O và trống cache. Ví dụ `feed` tách `feed_content`:

```sql
CREATE TABLE feed (
  fd_id BIGSERIAL PRIMARY KEY,
  fd_title VARCHAR(200) NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE feed_content (
  fc_id BIGSERIAL PRIMARY KEY,
  fc_feed_id BIGINT NOT NULL REFERENCES feed(fd_id),
  fc_body TEXT NOT NULL,
  fc_media JSONB NULL
);
```

Câu truy vấn hiển thị danh sách feed (không cần body) sẽ chỉ đọc bảng `feed` — ít I/O hơn.

---

## 6. Chọn kiểu dữ liệu và độ dài hợp lý
Chọn kiểu nhỏ nhất vẫn đáp ứng yêu cầu:
- Dùng `INTEGER` hoặc `BIGINT` hợp lý (id của hệ thống nhỏ dùng INT đủ). 64-bit chỉ khi cần.
- Dùng `VARCHAR(n)` có giới hạn nếu biết trước (ví dụ username 50). Dùng `TEXT` cho nội dung không giới hạn.
- Với JSON dữ liệu cấu trúc dùng `JSONB` (Postgres) để query hiệu quả.

Ví dụ: email dùng `VARCHAR(255)`; username `VARCHAR(50)`.

---

## 7. Thêm NOT NULL khi có thể
Đặt `NOT NULL` cho cột bắt buộc để rút gọn kiểm tra ở ứng dụng và giúp optimizer biết dữ liệu chắc chắn không null.

```sql
usr_email VARCHAR(255) NOT NULL
```

Nếu cột có giá trị mặc định hợp lý, kết hợp `NOT NULL DEFAULT ...`.

---

## 8. Đánh index hợp lý
Tạo index cho cột dùng trong WHERE / JOIN / ORDER BY. Tránh index thừa gây chậm ghi.

Ví dụ:

```sql
CREATE UNIQUE INDEX ux_users_email ON users (usr_email);
CREATE INDEX idx_users_created_at ON users (created_at DESC);
```

Sử dụng composite index hoặc partial index khi phù hợp (ví dụ index chỉ trên các bản ghi chưa xóa):

```sql
CREATE INDEX idx_users_email_active ON users (usr_email) WHERE is_deleted = false;
```

---

## 9. Áp dụng quy tắc chuẩn hóa 3NF (3rd Normal Form)
Thiết kế theo 3NF để tránh dư thừa dữ liệu và đảm bảo tính nhất quán. Nếu thấy nhiều cột lặp lại nhóm thông tin (ví dụ địa chỉ), tách thành bảng riêng.

Ví dụ: tách địa chỉ sang `user_address`:

```sql
CREATE TABLE user_address (
  ua_id BIGSERIAL PRIMARY KEY,
  ua_user_id BIGINT NOT NULL REFERENCES users(id),
  ua_address_line VARCHAR(255),
  ua_city VARCHAR(100),
  ua_country VARCHAR(100)
);
```

---

## Ví dụ tổng hợp — Bảng `users` tuân thủ tất cả quy tắc

```sql
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,

  usr_username VARCHAR(50) NOT NULL,
  usr_email VARCHAR(255) NOT NULL,
  usr_full_name VARCHAR(150),

  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_by BIGINT NULL,
  updated_by BIGINT NULL,
  version INTEGER NOT NULL DEFAULT 1,

  is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
  deleted_at TIMESTAMPTZ NULL
);

COMMENT ON COLUMN users.usr_username IS 'Tên đăng nhập (unique)';
COMMENT ON COLUMN users.usr_email IS 'Email người dùng';

CREATE UNIQUE INDEX ux_users_email ON users (usr_email) WHERE is_deleted = false;
CREATE INDEX idx_users_updated_at ON users (updated_at DESC);
```

Ghi chú cuối:
- Tuỳ database (MySQL, Postgres, MSSQL) có cú pháp khác nhau: `TIMESTAMPTZ` là Postgres, MySQL dùng `DATETIME(6)`.
- Một số ràng buộc như cập nhật `updated_at` tự động cần trigger hoặc cơ chế ORM.

Nếu muốn, sẽ tiếp tục bổ sung mẫu trigger để cập nhật `updated_at` và ví dụ về optimistic locking (sử dụng `version`).
