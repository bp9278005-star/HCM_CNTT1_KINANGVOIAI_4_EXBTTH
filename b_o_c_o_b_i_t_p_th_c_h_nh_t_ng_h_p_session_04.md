# BÁO CÁO BÀI TẬP THỰC HÀNH TỔNG HỢP - SESSION 04
**Chủ đề:** Ứng dụng Kỹ thuật Prompting Nâng cao (Chain-of-Thought, Prompt Chaining & Xử lý đứt gãy) xây dựng quy trình xử lý khiếu nại và hoàn tiền tự động.

---

## 1. BỐI CẢNH BÀI TOÁN & THIẾT KẾ GIẢI PHÁP

### 1.1. Bối cảnh thực tế
Trong thực tế vận hành một cửa hàng bán đồ công nghệ/phụ kiện online (ví dụ: **TechZone Store**), bộ phận chăm sóc khách hàng thường xuyên nhận được các tin nhắn yêu cầu bảo hành hoặc hoàn tiền. Dữ liệu khách gửi thường rất lộn xộn: viết tắt, gộp nhiều ý, thiếu mã đơn hàng, hoặc cố tình khai sai thời gian mua hàng.

Nếu chỉ dùng 1 câu lệnh đơn giản, AI rất dễ bị "rối" khi vừa phải đọc hiểu, vừa phải kiểm tra quy định, lại vừa phải viết thư trả lời — dẫn đến tình trạng bỏ sót lỗi hoặc tự bịa thông tin (ảo giác AI).

### 1.2. Kiến trúc luồng xử lý khép kín (Prompt Chaining)
Để giải quyết bài toán lớn này, mình chia nhỏ thành **3 mắt xích (3 Prompts)** chạy nối tiếp nhau, đầu ra của bước trước là đầu vào của bước sau:

1. **Bước 1 (Prompt 1 - Lọc & Chuẩn hóa dữ liệu):** Đọc tin nhắn thô của khách, bóc tách các trường thông tin cốt lõi ra định dạng JSON gọn nhẹ (tối ưu token) và bắt các lỗi dữ liệu dị biệt/trống ngay từ cửa ngõ.
2. **Bước 2 (Prompt 2 - Suy luận Logic / Chain-of-Thought):** Nhận JSON từ Bước 1, đối chiếu với quy định bảo hành/hoàn tiền của shop theo từng bước suy luận logic để ra quyết định (`DUYET`, `TU_CHOI`, hoặc `TAM_DUNG_DO_LOI`).
3. **Bước 3 (Prompt 3 - Phản hồi tự động):** Nhận kết quả quyết định từ Bước 2 để soạn email/tin nhắn phản hồi đúng giọng điệu và đúng kịch bản cho khách hàng.

---

## 2. HỆ THỐNG PROMPT THỰC THI CÁC BƯỚC

### Prompt 1: Trích xuất & Bắt lỗi dữ liệu dị biệt (Tối ưu Token)
*Mục tiêu: Chỉ lấy thông tin cần thiết, loại bỏ từ ngữ thừa để tiết kiệm token cho các bước sau, đồng thời chặn dữ liệu rác.*

```text
Nhiệm vụ: Trích xuất thông tin từ tin nhắn khách hàng sang JSON. Không giải thích thêm.

Quy tắc bắt lỗi:
- Nếu thiếu thông tin hoặc thông tin vô nghĩa/dị biệt (ví dụ: ký tự lạ, ngày tháng không tồn tại), gán giá trị là null.
- Mã đơn hợp lệ phải bắt đầu bằng "TZ" kèm 4 số (VD: TZ1024). Sai định dạng -> gán null.

Cấu trúc JSON đầu ra bắt buộc:
{
  "ma_don": string | null,
  "san_pham": string | null,
  "so_ngay_da_mua": number | null,
  "tinh_trang_loi": string | null,
  "co_dinh_kem_video_unbox": boolean
}

Tin nhắn khách hàng:
"{Input_Tin_Nhan}"
```

---

### Prompt 2: Kiểm tra quy định & Suy luận Logic (Chain-of-Thought)
*Mục tiêu: Ép AI suy nghĩ từng bước trước khi chốt phương án, đảm bảo không làm crash luồng xử lý khi gặp dữ liệu `null`.*

```text
Bạn là bộ phận kiểm duyệt nghiệp vụ của TechZone Store. Dựa vào JSON đầu vào, hãy kiểm tra điều kiện hoàn tiền/đổi trả.

### QUY ĐỊNH CỦA SHOP:
1. Bắt buộc phải có `ma_don` hợp lệ và `so_ngay_da_mua` khác null.
2. Thời gian mua hàng (`so_ngay_da_mua`) phải <= 7 ngày.
3. Nếu lỗi do nhà sản xuất (hỏng hóc kỹ thuật, không lên nguồn) VÀ `co_dinh_kem_video_unbox` là true -> Đủ điều kiện hoàn tiền 100%.
4. Nếu lỗi do khách hàng (rơi vỡ, vào nước) hoặc không có video unbox -> Từ chối hoàn tiền, chỉ hỗ trợ sửa chữa có phí.

### YÊU CẦU SUY LUẬN (CHAIN-OF-THOUGHT):
Phân tích ngắn gọn trong thẻ <logic> theo 3 bước:
- Bước 1 (Kiểm tra đứt gãy): Kiểm tra `ma_don` và `so_ngay_da_mua` có bị null không. Nếu có -> Chốt trạng thái `LOI_DU_LIEU` và dừng kiểm tra các bước sau.
- Bước 2 (Kiểm tra thời hạn): So sánh `so_ngay_da_mua` với 7 ngày.
- Bước 3 (Kiểm tra nguyên nhân & bằng chứng): Đối chiếu `tinh_trang_loi` và `co_dinh_kem_video_unbox`.

### ĐỊNH DẠNG ĐẦU RA (JSON):
{
  "quyet_dinh": "HOAN_TIEN" | "TU_CHOI_HOAN" | "LOI_DU_LIEU",
  "ly_do_chinh": "Nêu rõ 1 câu ngắn gọn lý do quyết định hoặc trường dữ liệu bị thiếu"
}

### DỮ LIỆU ĐẦU VÀO TỪ BƯỚC 1:
{Output_JSON_Buoc_1}
```

---

### Prompt 3: Soạn phản hồi khách hàng (Đóng chuỗi)
*Mục tiêu: Biến dữ liệu khô khan thành tin nhắn CSKH tự nhiên, lịch sự.*

```text
Đóng vai nhân viên CSKH của TechZone Store. Dựa vào kết quả kiểm duyệt dưới đây, hãy viết tin nhắn phản hồi ngắn gọn (dưới 100 từ), lịch sự cho khách:
- Nếu `quyet_dinh` là "LOI_DU_LIEU": Xin lỗi và nhờ khách cung cấp bổ sung thông tin ghi ở `ly_do_chinh`.
- Nếu `quyet_dinh` là "HOAN_TIEN": Thông báo đơn đủ điều kiện hoàn tiền và hướng dẫn khách gửi hàng về kho.
- Nếu `quyet_dinh` là "TU_CHOI_HOAN": Khéo léo từ chối dựa trên `ly_do_chinh` và mời khách mang qua trung tâm bảo hành hỗ trợ sửa chữa.

Dữ liệu kiểm duyệt:
{Output_JSON_Buoc_2}
```

---

## 3. KẾT QUẢ CHẠY THỬ HỆ THỐNG (TESTING)

Để kiểm tra độ ổn định và cơ chế bắt lỗi dị biệt, mình chạy thử chuỗi Prompt trên với một tin nhắn chứa **dữ liệu nhiễu và thiếu thông tin quan trọng** (khách gõ sai mã đơn, không nói rõ mua bao nhiêu ngày, nhưng có gửi video).

* **Đầu vào thực tế (Tin nhắn khách):**
  > *"Alo shop ơi cái tai nghe bluetooth hôm bữa mình đặt bị rè 1 bên không nghe được. Mình có quay video lúc mở hộp rồi nhé. Mã đơn hình như là 8899 hay gì đó không nhớ rõ, giải quyết gấp cho mình!"*

### Kết quả chạy Bước 1 (Trích xuất & Lọc nhiễu):
```json
{
  "ma_don": null,
  "san_pham": "tai nghe bluetooth",
  "so_ngay_da_mua": null,
  "tinh_trang_loi": "bị rè 1 bên không nghe được",
  "co_dinh_kem_video_unbox": true
}
```
*(Nhận xét: Prompt 1 đã bắt chuẩn xác bẫy dữ liệu — mã đơn "8899" thiếu tiền tố "TZ" nên bị chuyển thành `null`, cụm từ "hôm bữa" không xác định được số ngày cụ thể nên cũng chuyển thành `null`, không làm crash JSON).*

### Kết quả chạy Bước 2 (Suy luận Logic CoT):
```xml
<logic>
- Bước 1 (Kiểm tra đứt gãy): Phát hiện `ma_don` là null và `so_ngay_da_mua` là null. Thiếu dữ liệu định danh và thời gian bắt buộc -> Chốt trạng thái `LOI_DU_LIEU`, dừng kiểm tra điều kiện hoàn tiền để tránh duyệt sai quy định.
</logic>
```
```json
{
  "quyet_dinh": "LOI_DU_LIEU",
  "ly_do_chinh": "Thiếu mã đơn hàng hợp lệ (định dạng TZxxxx) và thời gian mua hàng cụ thể."
}
```

### Kết quả chạy Bước 3 (Tin nhắn phản hồi cuối cùng gửi khách):
> "Chào bạn, TechZone Store rất tiếc về sự cố tai nghe bluetooth bị rè mà bạn đang gặp phải ạ. Shop đã ghi nhận bạn có quay video mở hộp, tuy nhiên để hệ thống kiểm tra và hỗ trợ đổi trả nhanh nhất, bạn vui lòng kiểm tra lại giúp shop **mã đơn hàng chính xác (bắt đầu bằng chữ TZ)** và **ngày bạn nhận được hàng** nhé. Shop chờ phản hồi từ bạn để xử lý ngay ạ!"

---

## 4. TỔNG KẾT & ĐÁNH GIÁ KỸ THUẬT
* **Tính tối ưu Token:** Việc ép Prompt 1 và Prompt 2 trả về JSON ngắn gọn giúp giảm khoảng 50% lượng token truyền qua lại giữa các bước so với việc để AI viết văn dài dòng.
* **Chống đứt gãy hệ thống:** Cơ chế gán giá trị `null` khi gặp dữ liệu mập mờ ("hôm bữa", "8899") kết hợp với nhánh `LOI_DU_LIEU` giúp hệ thống vận hành trơn tru, tuyệt đối không xảy ra tình trạng AI tự ý duyệt hoàn tiền khi chưa đủ căn cứ.