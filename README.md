# # THỰC HÀNH TÌM LỖI VÀ SỬA SƠ ĐỒ TUẦN TỰ CHỨC NĂNG THANH TOÁN RIKKEIBANK

## 1. Phân tích các thành phần tham gia

Sơ đồ tuần tự có 4 Lifeline:

| Thành phần      | Loại   | Vai trò                                               |
| --------------- | ------ | ----------------------------------------------------- |
| Khách hàng      | Actor  | Nhập thông tin thẻ và nhận kết quả thanh toán         |
| Cổng Thanh Toán | Object | Tiếp nhận thông tin, kiểm tra và điều phối thanh toán |
| Ngân Hàng Lõi   | Object | Xử lý giao dịch thanh toán                            |
| EmailServer     | Object | Gửi hóa đơn thanh toán qua email                      |

---

# 2. Tìm 2 lỗi trong sơ đồ của thực tập sinh

## Lỗi 1 – Bước 2: Dùng Async thay vì Self

### Mô tả lỗi

Ở bước 2, **Cổng Thanh Toán tự kiểm tra định dạng thẻ trên chính nó**, nhưng thực tập sinh lại vẽ một thông điệp **Async** sang một Lifeline khác và tạo thêm một đối tượng để thực hiện việc kiểm tra.

### Vì sao sai?

Việc kiểm tra định dạng thẻ được xác định rõ là **xử lý nội bộ của Cổng Thanh Toán**, không có đối tượng nào khác tham gia. Việc tạo thêm Lifeline làm sai phạm vi của kịch bản và thể hiện không đúng bản chất của thao tác.

### Loại thông điệp đúng

**Self Message**

```text
Cổng Thanh Toán → Cổng Thanh Toán
checkFormat()
```

Ký hiệu là một mũi tên vòng cung quay lại chính Lifeline **Cổng Thanh Toán**.

### Hậu quả nếu giữ nguyên

* Làm xuất hiện một đối tượng không có trong kịch bản.
* Làm sai kiến trúc và luồng tương tác của hệ thống.
* Không thể hiện đúng việc Cổng Thanh Toán tự xử lý kiểm tra định dạng thẻ.

### Sửa thành

> **Bước 2: Self Message – `checkFormat()`**

---

# 3. Lỗi 2 – Bước 6: Dùng Sync thay vì Async

### Mô tả lỗi

Ở bước 6, Cổng Thanh Toán gửi `sendReceiptEmail()` đến EmailServer. Kịch bản quy định **không được chờ EmailServer phản hồi mới tiếp tục**, nhưng thực tập sinh lại sử dụng **Sync Message** và vẽ thêm Return.

### Vì sao sai?

Thông điệp Sync yêu cầu bên gửi **chờ bên nhận xử lý và trả kết quả**. Trong khi đó, nghiệp vụ yêu cầu Cổng Thanh Toán gửi yêu cầu gửi hóa đơn rồi **tiếp tục xử lý mà không chờ EmailServer**.

### Loại thông điệp đúng

**Asynchronous Message**

```text
Cổng Thanh Toán ───────▷ EmailServer
              sendReceiptEmail()
```

Đây là mũi tên **nét liền đầu hở**, thể hiện gửi yêu cầu và đi tiếp, không chờ phản hồi.

### Hậu quả nếu giữ nguyên

* Cổng Thanh Toán phải chờ EmailServer phản hồi.
* Nếu EmailServer phản hồi chậm, luồng thanh toán có thể bị chậm hoặc bị treo.
* Tăng thời gian phản hồi của hệ thống đối với khách hàng.
* Làm sai yêu cầu nghiệp vụ là **không chờ EmailServer**.

### Sửa thành

> **Bước 6: Async Message – `sendReceiptEmail()`**

**Không vẽ Return ngay sau Async**, vì Async không mặc định có Return đi kèm.

---

# 4. Tổng hợp 2 lỗi

| Lỗi   | Bước | Thực tập sinh vẽ | Đúng phải là | Lý do                                     |
| ----- | ---: | ---------------- | ------------ | ----------------------------------------- |
| Lỗi 1 |    2 | Async            | **Self**     | Cổng Thanh Toán tự kiểm tra trên chính nó |
| Lỗi 2 |    6 | Sync + Return    | **Async**    | Không được chờ EmailServer phản hồi       |

---

# 5. Luồng Sequence Diagram sau khi sửa

Sau khi sửa, sơ đồ chỉ sử dụng đúng 4 Lifeline:

```text
Khách hàng
Cổng Thanh Toán
Ngân Hàng Lõi
EmailServer
```

Luồng hoàn chỉnh:

```text
Khách hàng        Cổng Thanh Toán       Ngân Hàng Lõi       EmailServer
    |                    |                    |                  |
    |                    |                    |                  |
    |── nhập thẻ ───────>|                    |                  |
    |                    |                    |                  |
    |                    |↻ checkFormat()     |                  |
    |                    |                    |                  |
    |                    |── processPayment()->                  |
    |                    |                    |                  |
    |                    |<── kết quả giao dịch ─|                |
    |                    |                    |                  |
    |<── hiển thị thành công ─|               |                  |
    |                    |                    |                  |
    |                    |──── sendReceiptEmail() ──────────────>|
    |                    |                    |                  |
    |                    |                    |                  |
```

---

# 6. Phân tích từng thông điệp sau khi sửa

## Bước 1 – Khách hàng nhập thẻ

**Từ:** Khách hàng
**Đến:** Cổng Thanh Toán

**Thông điệp:**

`nhapTheTinDung()`

**Loại:** **Sync**

Khách hàng gửi thông tin thẻ cho Cổng Thanh Toán để bắt đầu quá trình thanh toán.

---

## Bước 2 – Cổng Thanh Toán tự kiểm tra định dạng

**Từ:** Cổng Thanh Toán
**Đến:** Cổng Thanh Toán

**Thông điệp:**

`checkFormat()`

**Loại:** **Self**

Đây là thao tác xử lý nội bộ nên Cổng Thanh Toán tự gọi phương thức của chính nó.

Khi vẽ trên draw.io, **không tạo thêm Lifeline Validator hoặc đối tượng kiểm tra định dạng**.

---

## Bước 3 – Gửi yêu cầu xử lý thanh toán

**Từ:** Cổng Thanh Toán
**Đến:** Ngân Hàng Lõi

**Thông điệp:**

`processPayment()`

**Loại:** **Sync**

Cổng Thanh Toán gửi yêu cầu xử lý giao dịch và **chờ Ngân Hàng Lõi trả kết quả**.

---

## Bước 4 – Ngân Hàng Lõi trả kết quả

**Từ:** Ngân Hàng Lõi
**Đến:** Cổng Thanh Toán

**Thông điệp:**

`Kết quả giao dịch`

**Loại:** **Return**

Đây là thông điệp phản hồi cho lời gọi `processPayment()` ở bước 3.

---

## Bước 5 – Cổng Thanh Toán phản hồi khách hàng

**Từ:** Cổng Thanh Toán
**Đến:** Khách hàng

**Thông điệp:**

`Hiển thị thành công`

**Loại:** **Return**

Đây là phản hồi cho tương tác ban đầu của Khách hàng với Cổng Thanh Toán theo mô tả của đề bài.

---

## Bước 6 – Gửi hóa đơn qua EmailServer

**Từ:** Cổng Thanh Toán
**Đến:** EmailServer

**Thông điệp:**

`sendReceiptEmail()`

**Loại:** **Async**

Cổng Thanh Toán gửi yêu cầu gửi hóa đơn và **không chờ EmailServer phản hồi**.

Vì vậy:

```text
Không vẽ Return ngay sau sendReceiptEmail()
```

---

# 7. Sơ đồ hoàn chỉnh cần vẽ trên draw.io

Sơ đồ cuối cùng nên có cấu trúc:

```text
                         Sequence Diagram – Thanh toán RikkeiBank

   Khách hàng          Cổng Thanh Toán       Ngân Hàng Lõi        EmailServer
       |                      |                     |                  |
       |                      |                     |                  |
       |─── nhapTheTinDung() ─>|                     |                  |
       |                      |                     |                  |
       |                      |↻ checkFormat()       |                  |
       |                      |                     |                  |
       |                      |── processPayment() ->|                  |
       |                      |                     |                  |
       |                      |<─ Kết quả giao dịch ─|                  |
       |                      |                     |                  |
       |<── Hiển thị thành công ─|                   |                  |
       |                      |                     |                  |
       |                      |── sendReceiptEmail() ──────────────────>|
       |                      |                     |                  |
       |                      |                     |                  |
```

### Các loại mũi tên cần sử dụng

```text
Sync:
──────────────▶

Async:
──────────────▷

Return:
- - - - - - - >

Self:
      ┌───────┐
      │       │
──────┘       │
      ↖───────┘
```

---

# 8. Cách bố trí trên draw.io

### Lifeline 1

**Khách hàng**

Đặt ở bên trái.

### Lifeline 2

**Cổng Thanh Toán**

Đặt bên cạnh Khách hàng.

### Lifeline 3

**Ngân Hàng Lõi**

Đặt bên phải Cổng Thanh Toán.

### Lifeline 4

**EmailServer**

Đặt ngoài cùng bên phải.

Thứ tự:

```text
Khách hàng → Cổng Thanh Toán → Ngân Hàng Lõi → EmailServer
```

---

# 9. Thứ tự vẽ trên draw.io

### 1. Tạo 4 Lifeline

Không tạo thêm Validator hoặc đối tượng nào khác.

### 2. Vẽ bước 1

```text
Khách hàng → Cổng Thanh Toán
nhapTheTinDung()
`
```
