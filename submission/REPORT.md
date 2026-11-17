# Báo cáo Day 08 — 2A202603116 - Nguyễn Quang Thức

> Một file duy nhất cho cả bài (đã gộp recipe, QC và ghi chú cá nhân). Mọi con số kèm **mẫu số** và nguồn (file nào). Viết bằng lời của bạn.

## 1. Model và recipe (Part 2–3)

| Lớp | Model + cỡ | Ngưỡng | Vì sao (dẫn `run_id` trong `model_log.csv`) |
|---|---|---|---|
| person | YOLO26m | 0.35 | `threshold_sweep`: recall 92% (11/12), đạt mục tiêu recall 90%; precision 35% (11/31). Nguồn: `y26_c025`, `y26_c050` và `qc/threshold_sweep.csv`. |
| bicycle / motorbike / car / bus / truck | YOLO26m | mặc định 0.3 | Không chọn ngưỡng riêng: bicycle và bus có dưới 5 mẫu gold; truck không có mẫu; motorbike và car không đạt recall 90% ở các ngưỡng <= 0.5. |
| three_wheeler | YOLOE-26s, prompt `tuk-tuk`, `cyclo`, `cargo tricycle` | 0.15 | `threshold_sweep`: recall 100% (7/7), precision 78% (7/9). Nguồn: `yoloe_twheel` và `qc/threshold_sweep.csv`. |

- Lệnh `merge_prelabels.py` sẽ dùng ở Part 4: `python scripts/merge_prelabels.py work/prelabel/yoloe_all/pre.coco.json work/prelabel/y26_all/pre.coco.json --thr "person=0.35,three_wheeler=0.15" --conf 0.3 --iou 0.6 --split "batch,gold" --out work/prelabel/merged_review/pre.coco.json`
- **Phiên bản SAM bạn chọn**: SAM 2.1; đã tạo mask cho 4 ảnh pilot với 57 box, nhưng chất lượng mask phụ thuộc vào độ chính xác của box YOLO.
- Lỗi không sửa được chỉ bằng đổi ngưỡng là lỗi sai lớp, đặc biệt tuk-tuk/xích lô bị gọi là `car` hoặc `truck`. Cách xử lý là dùng prompt YOLOE cho `three_wheeler`, merge theo ngưỡng riêng và review thủ công các box còn nghi ngờ.
- Box điểm cao vẫn có thể sai lớp ở các cảnh xe ba bánh; ngược lại, box `three_wheeler` điểm thấp vẫn có thể đúng khi vật bị che hoặc ở xa. Vì vậy `conf` chỉ dùng để ưu tiên review, không dùng thay cho kiểm tra ảnh. Ví dụ cần chú ý: `img_003.jpg`, `img_013.jpg` và `img_014.jpg`.
- Giới hạn: chọn ngưỡng trên chính gold (4 ảnh, 35 đối tượng) nên precision/recall trên gold là lạc quan; các lớp có ít mẫu chỉ cho kết luận yếu.

## 2. AI error profile (từ `qc/review_stats.md`)

| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted | 479 | 479/502 = 95.4% | 502 box máy |
| edited | 4 | 4/502 = 0.8% | 502 box máy |
| deleted | 19 | 19/502 = 3.8% | 502 box máy |
| added | 6 | 6/489 = 1.2% | 489 đối tượng sau duyệt |

- Lớp máy cần review kỹ nhất là `motorbike` và `three_wheeler`: `motorbike` có 6 box bị xoá, 4 box được thêm và 2 box chỉnh sửa; `three_wheeler` có 8 box bị xoá và 2 box chỉnh sửa. Lỗi tăng ở các cảnh nhiều phương tiện và vật thể nhỏ.
- So với gold: precision **44.3%** và recall **88.6%** của máy, trên 4 ảnh với **35 đối tượng gold** (`qc/review_stats.md`). Đây là kết quả in-sample và chỉ mang tính ước lượng.

## 3. Thiên kiến tự động hóa và ảnh bẫy

- Bắt bẫy: **12/12 = 100%**; loose 3/3, extra 3/3, missing 3/3, wrong_class 3/3; không còn bẫy nào lọt (`qc/review_stats.md`). Kết quả cho thấy việc quét ảnh trước khi tin vào nhãn máy và kiểm tra theo từng loại lỗi đã hiệu quả.
- Accepted cao (95.4%) đồng thời bắt bẫy đạt 100% cho thấy pre-label khá ổn ở nhiều box, nhưng vẫn cần review có hệ thống vì accepted không đồng nghĩa box luôn đúng.

## 4. Máy có đáng dùng không?

- Thời gian (từ `time_log.csv`): gán tay **15/4 = 3.75 phút/ảnh**; duyệt **90/24 = 3.75 phút/ảnh**. Có thể có thiên lệch do đã gán gold trước và đã quen với loại cảnh trong bộ dữ liệu.
- Kết luận: **Có, nhưng chỉ dùng pre-label có review thủ công**. Máy giúp khởi tạo box, nhưng precision trên gold chỉ 44.3%, recall 88.6%; `motorbike` và `three_wheeler` cần kiểm tra kỹ.
- **Ba quy tắc giảm thiên kiến của bạn** (mỗi quy tắc kiểm tra được bằng số hoặc việc làm):
  1. Quét toàn bộ ảnh trước khi nhìn hoặc tin vào box máy, rồi kiểm tra lần lượt bỏ sót → box dư → sai nhãn → sai vị trí.
  2. Với cụm xe dày hoặc vật thể rất nhỏ, đối chiếu guideline và dùng `ignore_region` khi phù hợp thay vì cố gán từng vật.
  3. Dùng ảnh bẫy và thống kê `accepted/edited/deleted/added` sau mỗi lô để kiểm tra chất lượng review, không chỉ dựa vào cảm giác.

## 5. Mask SAM cho 3 vật (Part 5)

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
|---|---|---|---|---|---|
| `img_003.jpg` | SAM 2.1 | 2 dương (trái), 1 âm (phải) | Mask bám phần lớn xe ba bánh nhưng có lẫn người ngồi và hàng hóa ở vùng tiếp giáp | Vùng giữa khung xe, người và hàng hóa khó tách sạch; cần điểm âm để loại vùng thừa | `masks/sam2.1_img003_three_wheeler.jpg` |
| `img_013.jpg` | SAM 2.1 | 1 dương (trái), 0 âm (phải) | Mask bao phủ xe ba bánh và lẫn một phần người lái/nội thất | Người lái và chi tiết phía trong xe làm biên mask bị lẫn | `masks/sam2.1_img013_three_wheeler.jpg` |
| `img_031.jpg` | SAM 2.1 | 3 dương (trái), 0 âm (phải) | Mask bao gồm xe đạp và hàng hóa chở trên xe; không nên bao gồm người bán | Hàng hóa che khuất khung xe, biên bánh và giỏ chưa hoàn toàn sạch | `masks/sam2.1_img031_bicycle.jpg` |

- Đối chiếu guideline mục 4.2: cần bám biên phần nhìn thấy, không tính bóng đổ, xử lý lỗ lớn trong khung xe và tách riêng hai vật chạm nhau. Vùng người lái/hàng hóa tiếp xúc với xe là phần khó, cần thêm điểm âm hoặc chỉnh polygon thủ công.
- Interactor SAM trong CVAT: đã thực hành trên 3 vật bằng thao tác semi-auto; kết quả nhanh nhưng vẫn cần kiểm tra biên mask, đặc biệt ở người lái, hàng hóa và khoảng trống trong khung xe.

## 6. Calibration (Part 6)

- Số quyết định: **3** (`calibration_decisions.csv`); quan trọng nhất: `CAL-01` về vật nhỏ và cụm xe, `CAL-02` về tách người khỏi mask xe, `CAL-03` về hàng hóa trên xe đạp.
- Quy tắc mới đã vào `guideline_v1.1_calibrated.md`: vật cao dưới 20 px dùng `ignore_region` khi phù hợp; mask xe không gộp người; hàng hóa gắn liền thuộc xe nhưng không bao người hoặc bóng.
- Nguyên nhân gốc: guideline v1.0 còn câu hỏi mở, mô hình có lỗi phân lớp/biên mask và vật nhỏ ở xa khó tách.

## 7. QC và quyết định (Part 6)

- Mẫu: **5 ảnh** (0 ảnh `can_review` + 5 ảnh ngẫu nhiên, seed 42); người kiểm: bạn (**hạn chế**: bạn vừa là người duyệt).
- Lỗi theo **đối tượng**: **0/105 = 0.0%** · theo **ảnh**: **0/5 = 0.0%** (ngưỡng Release ≤ 2%).
- Top lỗi: không ghi nhận lỗi nào trong mẫu QC; vì vậy không có ảnh lỗi đại diện trong mẫu.
- **Giới hạn mẫu nhỏ** (cận trên 95% trong `qc_report_auto.md`): với 0/5 ảnh lỗi, cận trên Wilson là **43%**; mẫu nhỏ không thể chứng minh chắc chắn tỉ lệ lỗi thật ≤2%.
- Script kiểm được tên/lớp/box, số lượng ảnh, box nhỏ và các số đo QC; không kiểm được box có ôm sát vật thật, vật mà cả người và máy cùng bỏ sót, hoặc việc tuân thủ checklist bằng mắt.

**Quyết định: Release**, ngày 2026-11-17. Lý do: không có lỗi trong 5/5 ảnh QC, bẫy đạt 12/12 và không phát hiện lỗi hệ thống; tuy nhiên kết luận có giới hạn do mẫu nhỏ.
Nếu Rework / Hold, việc cần làm tiếp: không áp dụng; khi dùng thực tế vẫn nên tăng kích thước mẫu QC.

## 8. Ghi chú cá nhân

- **Dùng AI để làm bài**: dùng trợ lý AI để đọc hướng dẫn, giải thích quy trình, hỗ trợ chạy script, kiểm tra file và tổng hợp nhận xét; các quyết định gán nhãn, review CVAT và click SAM do học viên kiểm tra.
- Gold có bị nhiễm không: **không**; 4 ảnh gold được gán thủ công trước khi dùng pre-label.
- Máy / GPU / CVAT dùng; vướng mắc cài đặt: Windows, CPU không có GPU, CVAT Docker v2.82.0; không ghi nhận vướng mắc cài đặt nghiêm trọng.
- Dùng cache `data/cache/` thay vì tự chạy? **không**; các model và script được chạy trong repo.
- Lỗi / bug script đã gặp: phát hiện một nhãn `bus` sai lớp trong `img_013.jpg` khi review và sửa thành `three_wheeler`; không phải lỗi của script xuất.
- Một điều bạn làm khác lần sau: tăng mẫu QC và đánh dấu `can_review` rõ hơn cho các ca mơ hồ trước khi chạy lấy mẫu.
