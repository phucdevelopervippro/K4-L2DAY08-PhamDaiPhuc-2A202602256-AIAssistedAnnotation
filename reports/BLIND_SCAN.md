# Quét độc lập trước khi xem pre-label

Frame: frame_0331.jpg

Số xe nhìn thấy bằng mắt: 38

Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe:
1. Xe ở làn ngoài cùng bên trái có đèn pha chiếu xuống mặt đường: AI dễ bị nhầm quầng sáng rọi trên mặt đường thành xe hoặc vẽ khung bao quá rộng ôm cả vệt đèn thay vì ôm sát thân xe.
2. Nhóm xe ở xa phía sau gần biển báo/chân cầu vượt: Các xe có kích thước nhỏ, thân xe tối chìm vào nền đường và chỉ thấy các chấm đèn hậu/đèn pha nhỏ, AI dễ bỏ sót hoàn toàn hoặc gộp hai xe đi gần nhau vào một khung.

Chạy `python3 tools/lock_blind.py` ngay sau khi điền. Sau đó giữ file này nguyên vẹn.