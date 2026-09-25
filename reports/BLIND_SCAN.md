# Quét độc lập trước khi xem pre-label

Frame: `frame_0099.jpg`

Số xe nhìn thấy bằng mắt: 22

Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe:

1. Cụm xe nhỏ ở xa, gần đường chân trời phía trên và giữa ảnh: xe tối, kích thước rất nhỏ, đèn xe dễ bị nhầm với ánh sáng nền nên có thể bị bỏ sót hoặc vẽ box lệch.
2. Các xe ở sát mép dưới và mép phải ảnh: một phần thân xe bị cắt khỏi khung hình, đặc biệt xe sáng màu ở góc dưới bên phải, nên box có thể bị thiếu hoặc vượt sai biên ảnh.

Chạy `python3 tools/lock_blind.py` ngay sau khi điền. Sau đó giữ file này nguyên vẹn.
