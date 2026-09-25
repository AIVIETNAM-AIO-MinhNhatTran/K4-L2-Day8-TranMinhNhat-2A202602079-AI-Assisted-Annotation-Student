# Vì sao chọn lô này?

## Năm frame ưu tiên nếu chỉ đủ ngân sách rà năm ảnh

Tôi xét 50 dòng đầu của `outputs/selection_round1.csv`, không chỉ lấy năm dòng đầu một cách máy móc. Năm frame tôi ưu tiên là:

| Thứ tự ưu tiên | Frame | Rank trong CSV | Thời điểm (s) | Score | U | A | D | Lý do |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | --- |
| 1 | `frame_0182.jpg` | 1 | 72,8 | 0,9591 | 0,9182 | 1,0000 | 1,0000 | Điểm tổng cao nhất, có 18/28 box mơ hồ. Ảnh có nhiều xe nhỏ và cụm đèn xa nên khả năng pre-label cần sửa cao. |
| 2 | `frame_0369.jpg` | 2 | 147,6 | 0,9324 | 0,9315 | 0,8889 | 1,0000 | Độ bất định cao, 16 box mơ hồ và cảnh đông xe. Ảnh nằm ở đoạn thời gian khác với frame 0182 nên bổ sung độ phủ cho lô. |
| 3 | `frame_0326.jpg` | 4 | 130,4 | 0,9155 | 0,9310 | 0,8333 | 1,0000 | Có cả U và A cao, 39 box dự đoán nên nhiều cơ hội tìm box thiếu/sai. Thời điểm cách đủ xa các lựa chọn trên. |
| 4 | `frame_0099.jpg` | 8 | 39,6 | 0,9063 | 0,9460 | 0,7778 | 1,0000 | U rất cao và đại diện đoạn đầu video. Đây cũng là ảnh quét độc lập: tôi nhìn thấy 22 xe, trong khi pre-label đóng gói chỉ có 13 box và sau rà còn 22 box. |
| 5 | `frame_0392.jpg` | 15 | 156,8 | 0,8874 | 0,9747 | 0,6667 | 1,0000 | Có U cao nhất trong năm lựa chọn dù A thấp hơn. Ảnh giúp kiểm tra các dự đoán rất thiếu chắc chắn ở cuối video mà vẫn cách `frame_0369.jpg` 9,2 giây. |

Chi phí rà không giống nhau: các frame 0182, 0369 và 0326 có nhiều xe/box mơ hồ nên tốn công hơn, nhưng với ngân sách năm ảnh tôi vẫn ưu tiên chúng vì khả năng phát hiện lỗi nhãn cao. `frame_0099.jpg` và `frame_0392.jpg` bổ sung hai vùng thời gian khác, tránh tập trung toàn bộ công sức vào một chuỗi cảnh gần trùng.

## Ba frame thuộc lô model chọn và bằng chứng

- `frame_0182.jpg` được chọn, rank 1, score 0,9591. CSV cho thấy A = 1,0 với 18 box mơ hồ; contact sheet cũng cho thấy nhiều xe nhỏ, tối và các cụm đèn ở xa. Sau rà, `round1_diff.md` ghi 13 box gợi ý ở ngưỡng đóng gói và 26 box cuối, gồm 13 box thêm mới.
- `frame_0369.jpg` được chọn, rank 2, score 0,9324. Đây là cảnh đông xe với 43 box dự đoán trong CSV và 16 box mơ hồ. Sau rà, ảnh còn 37 box cuối; 24 box được thêm mới, mức thêm cao nhất trong lô.
- `frame_0099.jpg` được chọn, rank 8, score 0,9063. U = 0,9460 cho thấy nhiều dự đoán gần vùng confidence khó quyết định. Quan sát độc lập ghi 22 xe; kết quả rà cũng có 22 box, gồm 13 box giữ nguyên và 9 box thêm.

Các con số `n_boxes` trong CSV được dùng để xếp hạng ứng viên, còn số pre-label trong `round1_diff.md` là các box đi vào bước rà ở ngưỡng cấu hình; vì vậy tôi không coi hai cột này là cùng một đại lượng.

## Một frame điểm cao nhưng không chọn

`frame_0372.jpg` đứng rank 6, score 0,9101, tại 148,8 giây nhưng không được model đưa vào lô. Frame này chỉ cách `frame_0369.jpg` tại 147,6 giây đúng 1,2 giây, nhỏ hơn `MIN_GAP_S = 2,0`. Với camera cố định, hai ảnh gần nhau có nền và nhiều xe gần trùng; rà cả hai làm tăng chi phí nhưng mang thêm ít thông tin. Vì vậy tôi giữ `frame_0369.jpg` có score cao hơn (0,9324) và loại `frame_0372.jpg` khỏi ngân sách năm ảnh. Đây là ví dụ cho thấy rank cao chưa đủ: phải xét khoảng cách thời gian và trùng lặp cảnh.

## Điều phép chọn này chưa chứng minh

Score cao chỉ cho biết mô hình đang bất định, có nhiều box mơ hồ hoặc frame đa dạng về thời gian theo các đại lượng đã thiết kế. Nó không chứng minh nhãn gợi ý chắc chắn sai, ảnh chắc chắn chứa thông tin mới, hay fine-tune trên ảnh đó sẽ làm AP50 tăng. Thực tế vòng 1 của bài này dùng các frame điểm cao nhưng AP50 vẫn giảm từ 0,7714 xuống 0,4286. Muốn kết luận về chất lượng chiến lược chọn mẫu cần nhiều vòng/lần chạy, đối chứng với chọn ngẫu nhiên và một tập kiểm thử lớn hơn, đã được người rà nhãn.
