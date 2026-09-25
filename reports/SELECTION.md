# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box:
1. **`frame_0182.jpg`** (Rank 1, Score: 0.9591, U: 0.9182, A: 1.0000, D: 1.0, Thời điểm: 72.8s): Đứng đầu toàn bộ tập pool về điểm tổng hợp; điểm bất định U rất cao (0.9182) và đạt điểm mập mờ tối đa A = 1.0 (với 18 box nằm trong dải phân vân 0.15 <= conf < 0.50). Khung hình ban đêm có mật độ xe phức tạp và nhiều quầng sáng đèn xe gây nhiễu, mang lại giá trị thông tin lớn nhất để hiệu chỉnh model.
2. **`frame_0369.jpg`** (Rank 2, Score: 0.9324, U: 0.9315, A: 0.8889, D: 1.0, Thời điểm: 147.6s): Mật độ xe dày đặc (43 box), điểm U vượt trội (0.9315) với 16 box mập mờ. Chọn frame này làm đại diện cho cụm ùn ứ ở khoảng thời gian t ~ 147s, đồng thời cho phép chủ động bỏ qua các frame gần trùng sát sườn như `frame_0368.jpg` (Rank 9, t = 147.2s, cách 0.4s) và `frame_0372.jpg` (Rank 6, t = 148.8s, cách 1.2s) để tránh lãng phí ngân sách rà nhãn vào cảnh trùng lặp.
3. **`frame_0331.jpg`** (Rank 5, Score: 0.9154, U: 0.8308, A: 1.0000, D: 1.0, Thời điểm: 132.4s): Có mật độ xe cao nhất trong các frame được chọn (47 box), điểm A đạt cực đại 1.0 (18 box mập mờ). Tình huống giao thông nhiều thách thức: xe ở làn ngoài cùng có vệt đèn pha rọi mặt đường và cụm xe nhỏ chìm vào nền tối cầu vượt (đã được ghi nhận độc lập ở `BLIND_SCAN.md`). Ta ưu tiên frame này và bỏ qua `frame_0330.jpg` (Rank 12, t = 132.0s, cách 0.4s) vì hai ảnh gần như trùng góc nhìn và vị trí xe.
4. **`frame_0099.jpg`** (Rank 8, Score: 0.9063, U: 0.9460, A: 0.7778, D: 1.0, Thời điểm: 39.6s): Đại diện cho mốc thời gian sớm (t ~ 40s) giúp phân bổ mẫu đều trên trục thời gian; điểm bất định U cao vượt bậc (0.9460), phản ánh việc 5 box khó nhất có độ tin cậy dao động quanh mức 0.50 khiến mô hình cực kỳ phân vân, rất cần chuyên gia con người can thiệp xác nhận.
5. **`frame_0392.jpg`** (Rank 15, Score: 0.8874, U: 0.9747, A: 0.6667, D: 1.0, Thời điểm: 156.8s): Sở hữu điểm bất định U cao nhất trong toàn bộ 12 frame được model chọn (U = 0.9747, cực kỳ sát mức tuyệt đối 1.0). Mốc thời gian cuối video (t = 156.8s) với điều kiện chiếu sáng và góc xe tạo ra nhiều trường hợp biên (edge cases) mà mô hình hoàn toàn thiếu tự tin, mang lại độ dốc học tập (learning gradient) rất cao sau khi được sửa nhãn.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet:
- **`frame_0182.jpg` (Rank 1, t = 72.8s)**:
  - *Bằng chứng CSV*: Điểm tổng hợp cao nhất 0.9591, A = 1.0 (18 box mập mờ, cao nhất pool), U = 0.9182, tổng số 28 box phát hiện.
  - *Bằng chứng ảnh contact sheet*: Khung cảnh đêm tối với nhiều đèn pha và đèn hậu chiếu thẳng. Hình ảnh thực tế cho thấy AI dự đoán nhiều box có độ tự tin lơ lửng quanh 0.2 - 0.5, vẽ nhầm vào quầng sáng phản chiếu trên mặt đường hoặc vẽ khung bao không dứt khoát ở các xe làn ngược chiều.
- **`frame_0331.jpg` (Rank 5, t = 132.4s)**:
  - *Bằng chứng CSV*: Phát hiện tới 47 box (nhiều nhất trong lô 12 ảnh), A = 1.0 (18 box mập mờ), U = 0.8308, score = 0.9154.
  - *Bằng chứng ảnh contact sheet*: Dòng xe ùn ứ kéo dài. Khớp với quan sát độc lập trong `BLIND_SCAN.md`: xe ở mép ngoài bên trái có chùm sáng pha rọi xuống lòng đường khiến AI dễ ôm cả vệt đèn vào box, còn ở hậu cảnh xa phía sau gần chân cầu vượt, các xe chỉ hiển thị chấm đèn nhỏ chìm vào nền tối khiến AI sinh ra nhiều box mập mờ hoặc bỏ sót.
- **`frame_0392.jpg` (Rank 15, t = 156.8s)**:
  - *Bằng chứng CSV*: U đạt kỷ lục 0.9747 (cao nhất trong toàn bộ lô 12 ảnh), A = 0.6667 (12 box mập mờ), score = 0.8874, n_boxes = 35.
  - *Bằng chứng ảnh contact sheet*: Nhóm xe ở cự ly trung bình và xa có viền thân xe mờ nhạt do thiếu sáng. Do U tính từ trung bình 5 giá trị `1 - |2c - 1|` cao nhất, mức U = 0.9747 chứng minh độ tin cậy của top 5 box này nằm sát mức 0.50 (tình trạng lưỡng lự 50/50 của mô hình), biến frame này thành mẫu học tập biên cực kỳ đắt giá.

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:
- **Frame điểm cao nhưng không nên chọn: `frame_0372.jpg`** (Rank 6, Score: 0.9101, U: 0.9202, A: 0.8333, D: 1.0, Thời điểm: 148.8s):
  - *Lý do*: Mặc dù `frame_0372.jpg` có điểm số nằm trong top 6 (cao hơn nhiều frame được chọn như Rank 7, 8, 10, 11, 13, 14, 15), nhưng thời điểm của nó (t = 148.8s) chỉ cách `frame_0369.jpg` (Rank 2, t = 147.6s) vẻn vẹn 1.2 giây.
  - Do camera giám sát giao thông là camera góc quay cố định (fixed view), trong khoảng thời gian 1.2 giây các phương tiện chỉ dịch chuyển một quãng rất ngắn, phối cảnh, điều kiện ánh sáng và góc che khuất gần như giống hệt nhau (ảnh gần trùng - near-duplicate).
  - Nếu chọn thêm frame này, người gán nhãn sẽ phải tốn công sức lặp lại trên cùng một khung cảnh mà mô hình không thu nhận thêm tri thức mới. Thuật toán `_greedy` với ngưỡng khoảng cách `min_gap_s = 2.0s` đã loại bỏ frame này để ưu tiên tính đa dạng, giúp tối ưu hóa chi phí và ngân sách gán nhãn.

Điều phép chọn này chưa chứng minh về chất lượng mô hình:
- Phép chọn mẫu bằng Uncertainty Sampling kết hợp tính đa dạng (U, A, D) chỉ đo lường **mức độ phân vân nội tại** của mô hình hiện tại trên các ứng viên chưa gán nhãn trong pool, chứ **không chứng minh được mô hình tổng quát hóa tốt hay kém**:
  1. **Lỗi tự tin sai (Overconfidence)**: Nếu mô hình nhận nhầm biển báo, vệt đèn rọi mặt đường hoặc bóng cây thành xe với độ tin cậy rất cao (ví dụ conf = 0.95), thì theo công thức $u(c) = 1 - |2(0.95) - 1| = 0.10$, điểm bất định U sẽ rất thấp. Khi đó, lỗi sai nghiêm trọng này bị bỏ qua và không được chọn vào lô sửa nhãn.
  2. **Bỏ sót hoàn toàn (False Negatives ở vùng tối)**: Các xe ở quá xa hoặc chìm sâu trong bóng tối khiến mô hình không tạo ra box nào (conf < 0.05) sẽ không đóng góp vào U và A, khiến các trường hợp khó bị lọt lưới nếu frame vẫn có các xe khác dễ nhận diện.
  3. **Đánh giá khách quan độc lập**: Hiệu quả thực sự của mô hình chỉ có thể được chứng minh bằng các thước đo chuẩn (như AP50, mAP) trên tập kiểm thử độc lập (`data/test/`) với nhãn chuẩn đã khóa, chứ không thể suy diễn từ điểm bất định của các ảnh được chọn trong quá trình active learning.
