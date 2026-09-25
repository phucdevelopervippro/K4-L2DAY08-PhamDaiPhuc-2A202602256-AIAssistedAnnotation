# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Phạm Đại Phúc

Công cụ gán nhãn đã dùng: CVAT Docker trên máy cá nhân

Mọi con số trong báo cáo được truy xuất chính xác từ `reports/rounds_table.md`, `outputs/selection_round1.csv`, `outputs/metrics_round*.json` và `outputs/round*_diff.md`. Không coi nhãn test do mô hình tạo là chân lý tuyệt đối.

## 1. Dữ liệu và cách chia tập

Tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian (time-based split) kèm vùng đệm an toàn (buffer gap) ở giữa thay vì chia ngẫu nhiên (random split) vì đặc thù của dữ liệu video thu thập từ camera giám sát giao thông góc quay cố định (stationary camera). 

Trong video giám sát góc cố định, các khung hình kế tiếp nhau (cách nhau chỉ vài trăm mili-giây đến vài giây) có độ tương đồng hình ảnh cực kỳ cao (near-duplicate) về góc nhìn, điều kiện ánh sáng, phối cảnh và vị trí của các phương tiện đang dừng đèn đỏ hoặc di chuyển chậm. Nếu chia ngẫu nhiên:
- **Rò rỉ dữ liệu (Data Leakage)**: Các frame gần như giống hệt nhau sẽ rơi đồng thời vào cả tập huấn luyện (pool/train) và tập kiểm thử (test).
- **Độ lệch của số đo (Overoptimistic Metrics)**: Mô hình chỉ cần "học vẹt" (ghi nhớ vị trí pixel và tọa độ tĩnh của xe ở khung cảnh đó) thay vì học các đặc trưng tổng quát để phát hiện xe. Hệ quả là các số đo trên tập kiểm thử (như AP50, Precision, Recall) sẽ bị **thổi phồng giả tạo** (lệch theo hướng quá lạc quan). Khi triển khai vào thực tế ở các đoạn thời gian khác hoặc camera khác, mô hình sẽ sụt giảm hiệu năng nghiêm trọng.

Do đó, việc chia tập theo dòng thời gian độc lập và có khoảng đệm ngăn cách là bắt buộc để đánh giá đúng năng lực tổng quát hóa thực sự của mô hình.

## 2. Mô hình khởi đầu lạnh (cold start)

Bảng số liệu Vòng 0 từ `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Dựa vào số đo từ `outputs/metrics_round0.json` và ảnh trực quan `outputs/compare_round0.jpg`:
- **Các loại xe mô hình không khớp nhãn tham chiếu**: Mô hình khởi đầu lạnh (YOLOv8n pretrained trên COCO) bỏ sót rất nhiều xe di chuyển ban đêm, bao gồm:
  1. Xe ở khoảng cách xa (small objects) chỉ hiện các cụm đèn pha/đèn hậu nhỏ;
  2. Xe có màu tối chìm vào nền đường hoặc bóng tối dưới chân cầu vượt;
  3. Xe bị che khuất một phần (partial occlusion) bởi dải phân cách hoặc xe khác;
  4. Xe có vệt đèn pha rọi mạnh xuống mặt đường bị mô hình vẽ khung bao lệch hoặc bỏ qua ranh giới thân xe thực tế.
- **Độ phủ (Recall) theo kích thước xe**:
  - `R small` chỉ đạt **0.1818** (18.2% - phát hiện được vỏn vẹn 12/66 box tham chiếu, bỏ sót hơn 81% xe nhỏ).
  - `R medium` đạt **0.5473** (54.7% - 162/296 box).
  - `R large` đạt **0.5610** (56.1% - 23/41 box).
  - Điều này cho thấy mô hình pretrained COCO nhận diện tương đối ổn định với các phương tiện lớn, gần camera nhưng hoàn toàn bất lực trước các xe nhỏ, xe ở xa trong điều kiện thiếu sáng.
- **Trường hợp cần người rà lại nhãn tham chiếu**: Nhãn tham chiếu trên tập test vốn được sinh bán tự động (pseudo-labels do mô hình lớn tạo ra), nên có thể tồn tại lỗi gán nhãn như: gán nhãn cho vệt đèn pha rọi mặt đường, nhận nhầm biển báo giao thông phát sáng thành xe (False Positive), hoặc bỏ sót xe thật ở rìa ảnh (False Negative). Trước khi kết luận mô hình dự đoán sai, chuyên gia con người cần trực tiếp đối chiếu hình ảnh thực tế với `GUIDELINE_LABEL.md` để xác thực xem nhãn tham chiếu có bị vẽ sai hoặc vi phạm quy tắc (như ôm vệt đèn rọi, gộp hai xe sát nhau) hay không.

## 3. Chiến lược chọn mẫu

### Giải thích công thức tính điểm và vai trò của `MIN_GAP_S`
Công thức chọn mẫu: `score = W_U * U + W_A * A + W_D * D` (với trọng số mặc định là `0.5 / 0.3 / 0.2`):
- **$W_U \cdot U$ (Trọng số 0.5 - Uncertainty)**: $U$ là trung bình của 5 giá trị bất định $u(c) = 1 - |2c - 1|$ cao nhất trong frame. Khi độ tin cậy $c$ của mô hình tiến sát $0.50$ (mô hình phân vân nhất, không chắc là xe hay nền), $u(c)$ đạt giá trị cực đại là $1.0$. Thành phần này ưu tiên các khung hình chứa các ca biên khó nhất khiến mô hình lưỡng lự.
- **$W_A \cdot A$ (Trọng số 0.3 - Ambiguity Density)**: $A$ là tỷ lệ số box "mập mờ" có confidence nằm trong khoảng $0.15 \le c < 0.50$, chuẩn hóa theo số lượng tối đa trong pool. Thành phần này phản ánh mật độ vùng khó/nhiễu của frame; frame càng có nhiều box mập mờ thì càng cần con người can thiệp chuẩn hóa.
- **$W_D \cdot D$ (Trọng số 0.2 - Temporal Diversity)**: $D$ đo khoảng cách thời gian từ frame đang xét đến frame đã gán gần nhất (chia cho cap 10 giây). Thành phần này đảm bảo việc lấy mẫu được rải đều trên toàn bộ chiều dài video thay vì dồn vào một phân đoạn. (Tại vòng 0 cold start, chưa có ảnh nào được gán nên $D = 1.0$ cho toàn bộ pool).
- **Vai trò của `MIN_GAP_S` (2.0 giây)**: Thuật toán chọn tham lam (`_greedy`) bắt buộc khoảng cách giữa hai frame được chọn trong cùng một lô phải $\ge 2.0$ giây. Với camera giao thông góc quay tĩnh, hai khung hình cách nhau dưới 2 giây hầu như có bối cảnh và vị trí xe trùng khớp hoàn toàn (near-duplicate). `MIN_GAP_S` ngăn chặn việc lãng phí ngân sách gán nhãn vào các frame trùng lặp, ép hệ thống chọn các frame mang lại tri thức mới.

### Cân nhắc từ `reports/SELECTION.md`
- **3 frame model chọn**:
  1. `frame_0182.jpg` (Rank 1, Score 0.9591, $U=0.9182, A=1.0$): Tối đa số box mập mờ (18 box) và độ bất định rất cao trong cảnh đêm phức tạp.
  2. `frame_0369.jpg` (Rank 2, Score 0.9324, $U=0.9315, A=0.8889$): Mật độ 43 box, $U$ vượt trội, đại diện cho cụm ùn tắc mốc 147s.
  3. `frame_0331.jpg` (Rank 5, Score 0.9154, $U=0.8308, A=1.0$): Mật độ cao nhất (47 box), chứa tình huống đèn rọi và xe chìm chân cầu vượt.
- **1 frame bị loại bỏ do trùng lặp**: `frame_0372.jpg` (Rank 6, Score 0.9101, t = 148.8s). Dù có điểm số thuộc top 6, frame này chỉ cách `frame_0369.jpg` (t = 147.6s) đúng 1.2 giây. Việc loại bỏ frame này để giữ khoảng cách $\ge 2.0s$ giúp tiết kiệm chi phí dán nhãn mà không làm mất thông tin.
- **Điểm bất định có chứng minh ảnh đó sẽ cải thiện mô hình không?**: **Không**. Điểm bất định cao chỉ phản ánh sự băn khoăn nội tại của mô hình, không đảm bảo fine-tune sẽ nâng cao chất lượng. Nếu sự bất định xuất phát từ nhiễu không thể khắc phục (aleatoric uncertainty như lóa đèn, nhiễu cảm biến) hoặc nếu tập train quá nhỏ (12 ảnh) dẫn tới hiện tượng trôi phân phối/quá thận trọng, việc dán nhãn các ảnh này có thể không làm tăng điểm kiểm thử tổng thể.

## 4. Các vòng học chủ động (active learning)

Bảng tổng hợp kết quả từ `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | - | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 333 | 0.638 | -0.133 | 1.000 | 0.099 | 0.181 | 0.000 | 0.068 | 0.488 |

### Phân tích chi tiết Vòng 1:
1. **Mức độ sửa nhãn gợi ý (từ `outputs/round1_diff.md`)**:
   - Trong 12 ảnh của lô 1, mô hình đề xuất ban đầu **169 box**.
   - Sau khi rà soát và hiệu chỉnh bằng CVAT, số box thực tế là **333 box**.
   - Thống kê chi tiết thao tác sửa nhãn:
     - **Accepted (chấp nhận)**: 146 box (tỷ lệ accept rate 86.4%).
     - **Edited (chỉnh sửa ranh giới)**: 8 box (thu hẹp box ôm thừa bóng tối hoặc điều chỉnh viền xe).
     - **Deleted (xóa False Positive của model)**: 15 box (xóa các box AI vẽ nhầm vào quầng sáng đèn rọi mặt đường, biển báo phản quang).
     - **Added (thêm mới False Negative của model)**: **179 box** (bổ sung số lượng cực lớn các xe ở xa, xe tối bị AI bỏ sót hoàn toàn).
2. **Biến động AP50**:
   - AP50 giảm từ **0.771** xuống **0.638** ($\Delta AP50 = -0.133$).
   - Precision tại ngưỡng 0.25 tăng tuyệt đối lên **1.000** (không còn bất kỳ False Positive nào).
   - Tuy nhiên, Recall sụt giảm nghiêm trọng từ **0.489** xuống **0.099** (chỉ còn 40 True Positives so với 197 ở vòng 0, số False Negatives tăng vọt lên 363).
3. **Nhóm xe tốt lên hoặc xấu đi trên cùng tập test**:
   - `R large`: Giảm nhẹ từ 0.561 xuống **0.488** (vẫn giữ được khả năng nhận diện các xe to ở tiền cảnh).
   - `R medium`: Giảm mạnh từ 0.547 xuống **0.068** (bỏ sót hơn 93% xe cỡ trung).
   - `R small`: Rơi thẳng về **0.000** (từ 0.182, không phát hiện được bất kỳ xe nhỏ nào ở xa).
   - Mô hình sau khi fine-tune 50 epochs trên 12 ảnh đã trở nên **cực kỳ thận trọng** (over-conservative): nó triệt tiêu hoàn toàn lỗi dự đoán nhầm (Precision đạt 100%), nhưng lại dìm toàn bộ confidence score của các xe nhỏ và xe cự ly trung bình xuống dưới ngưỡng 0.25.
4. **Đối chiếu thực tế và ca khó theo guideline**:
   - Đối chiếu giữa `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `outputs/round1_diff.md` trên `frame_0331.jpg`:
     - *Quan sát độc lập (Blind Scan)*: Ghi nhận 38 xe, chỉ rõ 2 vị trí AI dễ sai là vệt đèn pha rọi sàn làn ngoài cùng bên trái và cụm xe nhỏ chìm trong nền tối chân cầu vượt.
     - *Rà sửa nhãn (Review Log & Diff)*: Đã xóa 5 box (loại bỏ box ôm vệt đèn rọi mặt đường), sửa 2 box và thêm mới 19 box (bổ sung các xe nhỏ ở xa chân cầu vượt chìm trong nền tối).
     - *Kết quả mô hình sau train (`compare_round1.jpg`)*: Mô hình đã học được việc không vẽ nhầm vào vệt đèn rọi sàn (khắc phục triệt để lỗi FP), nhưng lại không tự tin dự đoán các xe nhỏ ở hậu cảnh xa.
   - *Mô tả ca khó theo guideline*: Các xe ở hậu cảnh xa chỉ hiển thị hai chấm đèn hậu màu đỏ hoặc đèn pha mờ nhạt, chiều cao box xấp xỉ ngưỡng 16 pixel và ranh giới thân xe chìm hoàn toàn vào bóng đêm. Việc phân định ranh giới thân xe quanh cụm đèn đòi hỏi annotator phải phóng to tối đa và phán đoán theo ngữ cảnh làn đường.

## 5. Kết luận và giới hạn

### Đánh giá kết quả và quyết định vòng tiếp theo
- **So sánh với khởi đầu lạnh**: Mặc dù Precision đạt mức hoàn hảo 1.000 và mô hình loại bỏ được các lỗi nhận diện nhầm quầng sáng đèn xe, AP50 tổng thể bị giảm 0.133 do Recall sụt giảm mạnh trên nhóm xe nhỏ và xe vừa.
- **Quyết định**: **Tạm dừng chưa train tiếp Vòng 2 ngay lập tức** vì kết quả chưa đạt ngưỡng cải thiện tối thiểu kỳ vọng ($\Delta AP50 \ge 0.01$). 
- **Nguyên nhân và việc cần làm trước khi train thêm**:
  1. *Kiểm tra phân phối confidence score*: Rất có thể mô hình vẫn định vị được xe nhưng confidence bị nén xuống mức 0.10 - 0.20 (thấp hơn ngưỡng cắt 0.25).
  2. *Điều chỉnh siêu tham số fine-tuning*: Cần giảm learning rate, đóng băng một phần backbone (freeze layers) để giữ tri thức tổng quát về vật thể từ COCO, hoặc bổ sung kỹ thuật data augmentation ban đêm thay vì để mô hình overfit vào 12 ảnh train.
  3. *Cân nhắc hàm mất mát*: Cần điều chỉnh trọng số focal loss / objectness loss để khuyến khích mô hình mạnh dạn dự đoán các xe nhỏ.

### Đề xuất 2 ca còn yếu hoặc bất định cho vòng sau (nếu tiếp tục)
1. **Ca xe nhỏ chìm trong bóng tối ở hậu cảnh xa**: Cần ưu tiên các frame có nhiều xe nhỏ dưới chân cầu vượt ở các mốc thời gian khác nhau để cung cấp thêm mẫu xe nhỏ cho mô hình (tuy nhiên chi phí rà nhãn rất cao, mất từ 7–10 phút mỗi ảnh để vẽ chính xác 30–40 box nhỏ).
2. **Ca xe bị che khuất ở dải phân cách giữa**: Các xe di chuyển ở làn ngược chiều bị rào chắn che một phần thân xe.
- *Lưu ý nguy cơ*: Cần giữ vững luật `MIN_GAP_S >= 2.0s` để tránh chọn các frame kế cận gây lãng phí ngân sách và trùng lặp cảnh quan.

### Giới hạn của tập kiểm thử ảnh hưởng đến kết luận
1. **Quy mô tập test nhỏ (20 ảnh)**: Với chỉ 20 khung hình (403 box tham chiếu), phương sai thống kê là rất lớn. Việc mô hình bỏ sót một vài cụm xe ở xa trong một vài ảnh có thể kéo tụt tỷ lệ Recall và AP50 một cách đáng kể, chưa phản ánh đầy đủ năng lực trên toàn bộ video dài.
2. **Quy tắc bỏ qua xe dưới 16 pixel**: 14 box xe quá nhỏ bị lọc bỏ khi chấm điểm giúp loại trừ nhiễu kích thước, nhưng cũng khiến việc đánh giá năng lực phát hiện xe ở cự ly cực xa chưa được đo lường trọn vẹn.
3. **Nhãn tham chiếu test chưa được rà thủ công**: Nhãn test do mô hình tạo tự động (pseudo-labels) có thể chứa lỗi hệ thống vốn có của mô hình lớn ban đầu. Nếu nhãn tham chiếu chứa các box ảo mà mô hình sau fine-tune đã khôn ngoan bỏ qua, mô hình lại bị phạt oan điểm False Negative theo cách tính IoU cứng nhắc.
