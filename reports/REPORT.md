# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Trần Minh Nhật

Công cụ gán nhãn đã dùng: CVAT Docker local, xuất/nhập định dạng Ultralytics YOLO Detection 1.0.

## 1. Dữ liệu và cách chia tập

Dữ liệu gồm 400 frame lấy từ một video liên tục của camera cố định, với khoảng cách 0,4 giây giữa hai frame liên tiếp. Vì cùng một xe tồn tại trong khung hình vài giây, chia ngẫu nhiên có thể đưa các ảnh gần như giống nhau, thậm chí cùng một chiếc xe, vào cả tập train và test. Khi đó mô hình được đánh giá trên đối tượng/cảnh đã thấy và số đo có xu hướng cao giả tạo do rò rỉ dữ liệu.

Vì vậy bài lab dùng 20 ảnh test lấy từ bốn đoạn thời gian, loại 112 ảnh làm vùng đệm và để 268 ảnh còn lại trong pool. Ảnh pool gần ảnh test nhất vẫn cách 4,4 giây. Cách chia theo thời gian và vùng đệm giảm tương quan trực tiếp giữa train với test, nên phép đánh giá khó hơn nhưng phản ánh khả năng tổng quát tốt hơn so với chia ngẫu nhiên.

Tập test có 417 box tham chiếu, trong đó 14 box cao dưới 16 px bị bỏ qua khi chấm; bảng kết quả vì thế đánh giá trên 403 box. Nhãn tham chiếu do mô hình khác sinh ra và chưa được người rà từng box, nên các số đo dưới đây là mức khớp với bộ tham chiếu, không phải chân lý tuyệt đối.

## 2. Mô hình khởi đầu lạnh (cold start)

Dòng vòng 0 từ `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0,771 | — | 0,925 | 0,489 | 0,640 | 0,182 | 0,547 | 0,561 |

Cold start có precision cao (0,9249) nhưng recall chỉ 0,4888: khi mô hình phát hiện thì phần lớn box khớp tham chiếu, nhưng nó bỏ sót 206/403 xe. `compare_round0.jpg` cho thấy lỗi chủ yếu nằm ở xe nhỏ/xa chỉ hiện thành cụm đèn, xe tối và xe bị cắt ở mép ảnh. Một số xe vừa ở vùng đông xe cũng bị bỏ sót hoặc box không đạt IoU 0,5. Recall theo kích thước xác nhận điều này: small chỉ 0,1818, thấp hơn nhiều so với medium 0,5473 và large 0,5610.

Một trường hợp cần người rà tham chiếu là các cụm hai chấm đèn rất xa gần đường chân trời, ví dụ vùng xe nhỏ trong `frame_0350`. Chỉ từ ảnh so sánh khó khẳng định mỗi cụm là một xe đủ rõ, phản chiếu ánh sáng, hay box cao dưới 16 px vốn phải bỏ qua. Cần xem ảnh gốc ở độ phân giải đầy đủ và áp dụng cùng guideline trước khi gọi dự đoán đó là FN/FP thật; không được sửa trực tiếp nhãn test.

## 3. Chiến lược chọn mẫu

Notebook dùng:

`score = 0,5·U + 0,3·A + 0,2·D`

Trong đó U đo độ bất định của confidence (cao nhất khi confidence gần 0,5), A biểu diễn tỷ lệ/số box mơ hồ đã chuẩn hóa, còn D khuyến khích khoảng cách thời gian với dữ liệu đã gán. Các frame được xét từ score cao xuống thấp. `MIN_GAP_S = 2,0` ngăn hai frame quá gần nhau cùng vào một lô, vì camera cố định khiến chúng gần trùng và không đáng với công rà nhãn lặp lại.

Ba ví dụ trong `SELECTION.md` là:

- `frame_0182.jpg`: rank 1, score 0,9591, U = 0,9182, A = 1,0; 18 box mơ hồ nên có xác suất cần nhiều thao tác rà.
- `frame_0369.jpg`: rank 2, score 0,9324, có 43 box dự đoán và 16 box mơ hồ; sau rà phải thêm 24 box.
- `frame_0099.jpg`: rank 8, score 0,9063, U = 0,9460; quét độc lập và nhãn cuối đều đếm 22 xe, trong khi pre-label đóng gói chỉ có 13 box.

Một quyết định khác là không chọn `frame_0372.jpg` dù frame này đứng rank 6 với score 0,9101. Nó ở giây 148,8, chỉ cách `frame_0369.jpg` 1,2 giây, nên bị ràng buộc khoảng cách loại ra. Việc này tiết kiệm công rà một cảnh gần trùng và dành ngân sách cho đoạn thời gian khác.

Điểm bất định không chứng minh một ảnh chắc chắn cải thiện mô hình. Confidence có thể thấp do bóng tối, vật rất nhỏ, mờ chuyển động hoặc do mô hình chưa được hiệu chỉnh; một ảnh còn có thể gần trùng với dữ liệu đã chọn. Cần nhãn người rà, kết quả trên test độc lập và đối chứng qua nhiều vòng mới đánh giá được lợi ích thật.

## 4. Các vòng học chủ động (active learning)

Bảng kết quả từ `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0,771 | — | 0,925 | 0,489 | 0,640 | 0,182 | 0,547 | 0,561 |
| 1 | yolov8n fine-tune vòng 1..1 | 12 | 338 | 0,429 | -0,343 | 1,000 | 0,132 | 0,233 | 0,000 | 0,085 | 0,683 |

Ở vòng 1, model đề xuất 169 box trên 12 ảnh. Sau khi rà còn 338 box: 143 box được giữ nguyên, 9 box được chỉnh, 17 box sai bị xóa và 186 box bị thiếu được thêm. Accept rate là 84,62% nếu tính trên các box gợi ý được đối chiếu. Số box thêm mới rất lớn cho thấy pre-label có precision tương đối tốt nhưng thiếu nhiều xe, nhất là các xe nhỏ/tối.

AP50 vòng 1 giảm 0,3428 so với cold start (0,7714 xuống 0,4286); vì chỉ có một vòng train nên đây cũng là mức giảm so với vòng trước. Precision tại confidence 0,25 tăng từ 0,9249 lên 1,0000 và FP giảm từ 16 xuống 0, nhưng recall giảm từ 0,4888 xuống 0,1315, F1 giảm từ 0,6396 xuống 0,2325. Mô hình sau fine-tune trở nên quá dè dặt: nó chỉ tạo 53 TP và bỏ sót 350 xe.

Theo kích thước, recall small giảm từ 0,1818 xuống 0; medium giảm từ 0,5473 xuống 0,0845; riêng large tăng từ 0,5610 lên 0,6829. Điều này cho thấy vòng train nhỏ đã thiên về các xe lớn/rõ và làm mất khả năng phát hiện phần lớn xe nhỏ/vừa. Với chỉ 12 ảnh train, chưa thể kết luận đây là xu hướng ổn định; cần kiểm tra phân bố kích thước box, cấu hình train và confidence trước khi train thêm.

`compare_round1.jpg` cho một ca thay đổi rõ ở `frame_0050`: cold start có TP 11, FP 2, FN 7; vòng 1 chỉ còn TP 3, FP 0, FN 16. Việc loại được hai FP là mặt tốt, nhưng đánh đổi bằng tám TP bị mất và chín FN tăng thêm, chủ yếu ở xe nhỏ/vừa và xe xa. Các frame 0150, 0250 và 0350 cũng cùng kiểu: FP về 0 nhưng FN tăng. Đây là bằng chứng trực quan phù hợp với precision 1,0 và recall 0,1315.

Cần phân biệt ba lớp bằng chứng:

- `BLIND_SCAN.md` là quan sát trước pre-label trên `frame_0099.jpg`: tôi đếm 22 xe và dự đoán vùng xe xa cùng mép ảnh dễ sai.
- `REVIEW_LOG.csv` và `round1_diff.md` mô tả lỗi của nhãn AI ban đầu đã được sửa. Riêng `frame_0099.jpg` có 13 box giữ nguyên, 9 box thêm, tổng 22; cả lô có 186 box thêm và 17 box xóa.
- `metrics_round1.json` và `compare_round1.jpg` đánh giá model mới sau train trên tập test, không phải đánh giá lại nhãn pre-label. Model sau train bỏ sót nhiều hơn dù nhãn train đã được bổ sung.

Ca khó điển hình là xe tối chỉ thấy đèn hoặc xe bị cắt ở mép ảnh. Theo guideline, nếu vẫn suy ra được thân xe thì box phải ôm phần thân nhìn thấy/ước lượng được, không chỉ khoanh hai chấm đèn hay vệt sáng; xe bị cắt chỉ vẽ phần nằm trong ảnh. Hai xe sát nhau phải có hai box riêng. Đây cũng là lý do vùng mép dưới/phải và cụm đèn xa trong `frame_0099.jpg` cần rà thủ công.

## 5. Kết luận và giới hạn

Vòng 1 không cải thiện cold start: AP50 giảm 0,3428 và recall giảm mạnh, dù precision đạt 1,0 và recall xe lớn tăng. Tôi dừng sau vòng 1 để kiểm tra nguyên nhân thay vì tiếp tục train ngay trên một mô hình đang quá dè dặt. Trước hết cần xem lại phân bố box theo kích thước trong 12 ảnh, tính nhất quán của box xe tối/xe xa, tham số fine-tune và ngưỡng confidence; đồng thời xem các dự đoán dưới 0,25 có tồn tại nhưng bị ngưỡng loại hay không.

Nếu tiếp tục vòng 2, hai ca tôi ưu tiên từ `selection_round2.csv` là:

- `frame_0076.jpg` (rank 1, 30,4 s, score 0,7618, U = 0,7271, A = 0,7143, D = 0,92): có 5/7 box mơ hồ, khá khác về thời gian so với dữ liệu đã gán và số box ít nên chi phí rà vừa phải.
- `frame_0030.jpg` (rank 2, 12,0 s, score 0,7228, U = 0,6170, A = 0,7143, D = 1,0): bổ sung đoạn đầu video chưa được phủ, cũng chỉ có 7 box dự đoán nên có thể rà nhanh và kiểm tra hiện tượng model bỏ sót hàng loạt.

Tôi sẽ chưa ưu tiên `frame_0388.jpg` dù A = 1,0, vì D chỉ 0,16: frame này gần dữ liệu đã gán ở cuối video và có nguy cơ mang thông tin gần trùng. Nếu chọn nó, lợi ích phải được cân bằng với chi phí rà và so sánh trực tiếp với các frame 0380/0392 đã có.

Kết luận còn bị giới hạn bởi test chỉ có 20 ảnh, 14 box quá nhỏ bị bỏ qua và nhãn tham chiếu do mô hình tạo chưa được người rà. Vì vậy một số FN/FP có thể phản ánh bất đồng với nhãn tham chiếu hơn là lỗi tuyệt đối; kết quả cũng có phương sai lớn và không đại diện cho mọi cảnh giao thông ban đêm. Nếu AP50 giảm, tôi sẽ kiểm tra nhãn train và test bằng ảnh, phân bố kích thước, lỗi duplicate/missing, ngưỡng confidence, loss/đường học và cấu hình fine-tune trước khi thêm dữ liệu hoặc tăng epoch. Chỉ sau bước QC đó mới chạy vòng tiếp theo và so trên đúng cùng tập test.
