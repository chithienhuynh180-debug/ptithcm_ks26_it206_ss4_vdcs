# Thiết kế giải pháp Prompting ứng dụng thực tế

## Bước 1: Thiết kế giải pháp (I/O)

Bài toán: cửa hàng trà sữa nhận đơn qua tin nhắn của khách, cần AI tự đọc, kiểm tra và tạo đơn hàng chuẩn.

**Input**

- [TIN_NHẮN]: tin nhắn đặt hàng của khách
- [MENU]: danh sách món, size, giá

**Output**

- Phần suy luận ngắn từng bước (Chain-of-Thought)
- Đơn hàng dạng JSON gồm: tên_khách, sđt, địa_chỉ, danh_sách_món (tên, size, số lượng, thành tiền), tổng_tiền, trạng_thái (HỢP LỆ / CẦN HỎI LẠI), lỗi, câu_hỏi_lại

**Cách dùng 3 kỹ thuật**

- Chain-of-Thought: bắt AI suy luận từng bước trước khi ra kết quả.
- Prompt Chaining: chia 3 bước nối tiếp: (1) trích xuất thông tin, (2) kiểm tra với menu, (3) tính tiền và tạo đơn. Kết quả bước trước là đầu vào bước sau.
- Xử lý đứt gãy: có điều kiện khi dữ liệu thiếu hoặc sai thì không đoán bừa mà ghi vào lỗi và hỏi lại khách.

**Bẫy dữ liệu lường trước**

- Thiếu sđt hoặc địa chỉ
- Thiếu size
- Món không có trong menu
- Số lượng không rõ hoặc bằng 0

## Bước 2: Prompt hoàn chỉnh

```
Bạn là nhân viên nhập đơn của cửa hàng trà sữa.

NHIỆM VỤ: Đọc [TIN_NHẮN] của khách, đối chiếu với [MENU], tạo đơn hàng chuẩn.

DỮ LIỆU:
[MENU]: 
- Trà sữa trân châu: M 35.000đ, L 40.000đ
- Trà đào: M 30.000đ, L 35.000đ
- Cà phê sữa: M 25.000đ, L 30.000đ
[TIN_NHẮN]: "cho mình 2 trà sữa trân châu size L, 1 trà đào và 1 matcha latte. giao về 12 Nguyễn Huệ"

LÀM THEO 3 BƯỚC, KẾT QUẢ BƯỚC TRƯỚC LÀ ĐẦU VÀO BƯỚC SAU:
Bước 1: Trích xuất tên khách, sđt, địa chỉ, món, size, số lượng từ [TIN_NHẮN].
Bước 2: Kiểm tra từng món với [MENU].
Bước 3: Tính thành tiền và tổng tiền, chỉ tính các món đã đủ thông tin.

ĐIỀU KIỆN XỬ LÝ LỖI:
- Nếu thiếu tên, sđt hoặc địa chỉ: ghi "chưa có" và thêm vào lỗi.
- Nếu thiếu size: không tự chọn size, ghi vào lỗi.
- Nếu món không có trong [MENU]: không tính tiền món đó, ghi vào lỗi.
- Nếu số lượng không rõ hoặc bằng 0: ghi vào lỗi.
- Nếu có ít nhất 1 lỗi: trạng_thái = "CẦN HỎI LẠI" và viết câu_hỏi_lại cho khách. Nếu không có lỗi: trạng_thái = "HỢP LỆ".
- Tuyệt đối không tự đoán hoặc bịa thông tin còn thiếu.

ĐỊNH DẠNG ĐẦU RA:
1. Phần "Suy luận": mỗi bước 1 gạch đầu dòng, tối đa 15 từ.
2. Phần "Đơn hàng": JSON gồm tên_khách, sđt, địa_chỉ, danh_sách_món, tổng_tiền, trạng_thái, lỗi, câu_hỏi_lại.
3. Cấm lời chào, lời dẫn, lời kết.
```

## Kết quả AI phản hồi

**Suy luận**

- Bước 1: khách đặt trà sữa trân châu L x2, trà đào x1, matcha latte x1. Địa chỉ 12 Nguyễn Huệ. Chưa có tên, sđt.
- Bước 2: trà sữa trân châu L hợp lệ. Trà đào thiếu size. Matcha latte không có trong menu.
- Bước 3: chỉ tính trà sữa trân châu: 2 x 40.000 = 80.000đ.

**Đơn hàng**

```json
{
  "tên_khách": "chưa có",
  "sđt": "chưa có",
  "địa_chỉ": "12 Nguyễn Huệ",
  "danh_sách_món": [
    {"tên": "Trà sữa trân châu", "size": "L", "số_lượng": 2, "thành_tiền": 80000},
    {"tên": "Trà đào", "size": "chưa rõ", "số_lượng": 1, "thành_tiền": null},
    {"tên": "Matcha latte", "size": null, "số_lượng": 1, "thành_tiền": null}
  ],
  "tổng_tiền": 80000,
  "trạng_thái": "CẦN HỎI LẠI",
  "lỗi": [
    "thiếu tên khách",
    "thiếu số điện thoại",
    "trà đào chưa chọn size",
    "matcha latte không có trong menu"
  ],
  "câu_hỏi_lại": "Bạn cho mình xin tên và số điện thoại nhé. Trà đào bạn muốn size M hay L. Quán không có matcha latte, bạn chọn món khác như trà sữa trân châu, trà đào hoặc cà phê sữa được không."
}
```