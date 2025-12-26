# Tối ưu Performance cho tính năng Upsert trong Laravel

Bài viết này chia sẻ kinh nghiệm về việc cải thiện hiệu suất khi xử lý dữ liệu lớn với tính năng `upsert` của Laravel (v8+).

## 1. Cơ chế hoạt động của Laravel Upsert
Laravel thực thi lệnh `upsert` thông qua câu lệnh SQL:
```sql
INSERT INTO ... ON DUPLICATE KEY UPDATE ...
```
**Ưu điểm:** Gộp lệnh `INSERT` và `UPDATE` vào một lần gọi duy nhất dựa trên khóa chính hoặc unique key, giúp code gọn gàng hơn.

---

## 2. Vấn đề & Giải pháp tạm thời

### Vấn đề thường gặp
Khi xử lý lượng data lớn (ví dụ: 10,000 records), nếu không kiểm soát tốt cách thực thi, số lượng query có thể tăng vọt, gây ảnh hưởng nghiêm trọng đến hiệu suất hệ thống.

### Giải pháp "Batching" (Tạm thời)
Thay vì dùng `upsert` mặc định nếu cảm thấy nó chậm, chúng ta có thể tách thành 2 câu query lớn:
1. **Batch Insert:** Lọc các ID chưa tồn tại và gộp lại để chèn một lần.
2. **Batch Update:** Sử dụng cấu trúc `UPDATE...CASE...WHEN` để cập nhật đồng loạt các bản ghi cũ.

**So sánh logic Update:**
*   **Cách 1 (Nhiều query đơn):** `UPDATE table SET field = val WHERE id = x`. Hệ thống phải kiểm tra điều kiện $N$ lần cho $N$ bản ghi ($N^2$ checks).
*   **Cách 2 (Một query gộp):** Sử dụng `CASE WHEN`. Hệ thống chỉ cần duyệt bảng một lần ($N$ lần check).

---

## 3. Phân tích Performance & Thực tế (Cập nhật mới)

Qua quá trình test thực tế và đối sánh, chúng ta có những kết quả bất ngờ:

> [!IMPORTANT]
> **Kết quả Benchmark:**
> 1. `upsert` mặc định của Laravel chạy **nhanh gấp 10 lần** so với cách dùng `UPDATE CASE WHEN`.
> 2. `upsert` mặc định chạy **nhanh gấp đôi** so với cách sử dụng "Dynamic Temporary Table" (Join với block VALUES).

### Khi nào nên dùng Laravel Upsert?
*   **Dữ liệu nhỏ (5-10 records):** Hiệu suất không chênh lệch đáng kể, dùng mặc định cho nhanh và tiện.
*   **Dữ liệu lớn (Bulk Import):** Laravel `upsert` xử lý cực tốt vì nó đã hỗ trợ bulk insert/update trong một câu query duy nhất (nếu truyền vào array dữ liệu).
*   **Hỗ trợ đa khóa:** Hoạt động tốt với cả Primary Key và Unique Key phức hợp (Composite keys).

---

## 4. Lưu ý & Tài nguyên bổ sung

### Lưu ý kỹ thuật
*   Hiện tại các Trait hỗ trợ chỉ mới được test ổn định trên **MySQL**.
*   Các giải pháp tùy chỉnh (`SqlBulkUpdatable`, `wantsUpsertQuery`) đang hỗ trợ tối ưu cho 1 field cụ thể.

### Công cụ hỗ trợ
Nếu bạn quan tâm đến việc tối ưu sâu hơn cho Batch Update, mình đã đóng gói một package tại đây:
👉 **[quanggpv/fast-batch-update](https://github.com/quanggpv/fast-batch-update)**

---
*Cảm ơn các bạn đã đọc, hy vọng chia sẻ này giúp ích cho dự án của bạn!*
