# Label contract — Phố châu Á — BỔ SUNG v1.1 — 2A202603116 - Nguyễn Quang Thức

> **Chỉ viết phần thay đổi so với v1.0** (không copy cả file). Chốt các câu hỏi mở ở mục 6 của v1.0 (hoặc nói rõ câu nào chưa chốt, vì sao)
> và thêm quy tắc mới rút ra từ lỗi của máy / lỗi của bạn. Mỗi ca khó **có ảnh ví dụ** (`img_0xx`) và khớp một dòng trong `qc/calibration_decisions.csv`.

## 1. Câu hỏi mở đã chốt (từ mục 6 của v1.0)

| # | Câu hỏi | Quyết định | Ảnh ví dụ | `case_id` |
|---|---|---|---|---|
| 1 | Xe đạp điện: bicycle hay motorbike? | Xe có bàn đạp và không thấy động cơ thì gán `bicycle`; có động cơ/đặc điểm xe máy thì gán `motorbike`; không đủ rõ thì tick `can_review`. | img_031 | CAL-03 |

## 2. Ca khó mới (từ lỗi của máy, bẫy lọt, box `can_review`)

| Ảnh | Tình huống | Máy làm gì (model nào) | Quyết định của bạn | Quy tắc mới (vào mục nào của v1.0) |
|---|---|---|---|---|
| img_018 | Mô hình tạo nhiều box rất nhỏ cho các phương tiện ở xa, khó tách thành từng đối tượng riêng. | YOLO26 tạo các box nhỏ, có thể làm tăng số lượng nhãn sai. | Dùng một `ignore_region` bao cụm thay vì giữ từng box nhỏ. | Mục 4.1.6: vật cao dưới 20 px không gán riêng; cụm từ 5 vật nhỏ trở lên dùng một `ignore_region`. |
| img_013 | Mask xe ba bánh có thể lan sang vùng người lái hoặc hành khách. | SAM 2.1 có thể ăn vào vùng người bên trong xe. | Tách `person` thành nhãn riêng và dùng click âm để loại phần mask bị lẫn. | Mục 4.2.1, 4.2.4: mỗi vật một mask, không gộp người vào mask xe. |
| img_031 | Xe đạp chở nhiều hàng hóa khiến mask dễ bao nhầm người bán hàng hoặc bóng đổ. | SAM 2.1 có thể bao cả hàng, người và bóng trong cùng một mask. | Bao hàng hóa gắn liền với xe, nhưng không bao người hoặc bóng đổ. | Mục 4.1.5, 4.2.3: hàng gắn liền thuộc xe; người và bóng là phần ngoài mask. |

## 3. Lịch sử thay đổi

| Phiên bản | Thay đổi | Lý do (ca nào, `case_id`) |
|---|---|---|
| v1.0 | Bản giảng viên | — |
| v1.1 | Bổ sung quy tắc cho vật quá nhỏ, mask tách người lái và hàng hóa trên xe đạp. | CAL-01, CAL-02, CAL-03 |
