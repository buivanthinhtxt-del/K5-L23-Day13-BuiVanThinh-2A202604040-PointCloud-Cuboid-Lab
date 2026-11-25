# Báo cáo Day 13 — Bùi Văn Thịnh, 2A202604040

## 1. Kết luận trước
- **Thông tin bài làm:** MSSV `2A202604040`, task `D13-2A202604040`, gồm 10 frame: `s01f01`, `s01f55`, `s02f04`, `s02f25`, `s03f05`, `s03f43`, `s04f21`, `s04f38`, `s05f09`, `s05f38`. Môi trường: Windows 11, trình duyệt Chrome, CVAT phiên bản `2.81.1` (tự cài local). Bạn ngồi cạnh / đổi kiểm: MSSV `2A202604045`.
- **Tóm tắt kết quả:** Tổng số **123 cuboid** dạng `Shape` (`104 Car`, `10 Cyclist`, `4 Pedestrian`, `3 Bus`, `2 Truck`), vượt yêu cầu tối thiểu 53 cuboid; hai frame ít nhất là `s02f04` và `s03f43` (7 cuboid mỗi frame) do cảnh đường thẳng/ngã tư thưa xe trong bán kính 50 m. Lệnh `selfcheck` gắn cờ **4/123 cuboid** (đều là gợi ý hợp lý do vật đứng sát nhau hoặc vòng mặt đường bị xe xung quanh che). Phần tự tin nhất là độ bám đáy sát mặt đường (`ground_gap_m` đa số trong ±0,06 m) và hướng `yaw_deg` đối chiếu qua 7 camera; phần còn khó nhất là suy luận chiều dài các xe ở xa (> 35 m) hoặc chỉ hứng LiDAR một phần do bị xe khác che khuất.
- **Cách thao tác CVAT 3D:**
  - **Đổi góc nhìn:** Trong khung 3D (Perspective), kéo chuột trái để xoay góc nhìn (xoay thẳng từ trên xuống để quét toàn cảnh 360°), kéo chuột phải để dời khung, lăn con lăn để phóng to/thu nhỏ, nháy đúp vùng trống để về góc nhìn mặc định.
  - **Vẽ và chỉnh cuboid:** Chọn công cụ cuboid ở thanh trái → chọn đúng class → bấm **Shape** (không dùng Track) → nháy đúp vào cụm điểm vật thể; sau đó chọn cuboid để căn tâm, dài, rộng, `yaw` ở khung **Top** và chỉnh đáy chạm mặt đường, chiều cao ở khung **Side / Front** (không xoay ở Side/Front).
  - **Hướng và đối chiếu ảnh:** Bật **Appearance → Cuboid orientation** để hiện vạch đỏ ngắn (`+X` của cuboid) và căn về phía đầu xe theo ảnh 7 camera (`cam01`–`cam09`), sau đó chạy `python scripts/lab.py preview` để kiểm tra hình chiếu 3D lên toàn bộ ảnh camera.

## 2. Đọc frame (Bước 2)
Hai frame đọc kỹ: `s01f01` và `s01f55`. Ảnh minh họa: `t1-s01f01-3d.png`, `t1-s01f01-top.png`, `t1-s01f55-3d.png`, `t1-s01f55-top.png`.

| Frame | Phía trước xe ở cạnh nào của ảnh nhìn từ trên xuống; +x, +y chỉ về đâu | Mặt đường (phẳng/dốc; thấp hơn LiDAR ~ m) | Vật xa nhất đọc được (~ m) | Loại vật thấy | Vật phía sau / hai bên | Cụm chưa đọc được |
|---|---|---|---|---|---|---|
| `s01f01` | Phía trước (`−y`) ở cạnh trên; `+x` chỉ sang trái; `+y` chỉ xuống dưới (phía sau) | Tương đối phẳng ở ngã tư, hơi dốc nhẹ về phía trước; thấp hơn LiDAR khoảng 1,70–2,00 m | ~41,3 m (`#15 Truck`, `41,29 m`) | `Car`, `Truck`, `Bus`, `Cyclist`, `Pedestrian` | Phía sau có 5 xe con (`cam09`); bên trái có xe con trắng gần, SUV xanh và 2 xe hai bánh (`cam06`); bên phải có 2 người đi bộ, 2 xe hai bánh, SUV trắng và xe buýt (`cam07`, `cam08`) | Cụm điểm rời rạc sát lề cây xanh ở góc trước-phải (~25–30 m), khó phân biệt bụi cây hay biển báo nhỏ nếu không xem ảnh |
| `s01f55` | Phía trước (`−y`) ở cạnh trên; `+x` chỉ sang trái; `+y` chỉ xuống dưới (phía sau) | Đường thẳng nhiều làn phẳng đều; thấp hơn LiDAR khoảng 1,75–1,85 m | ~38,9 m (`#11 Car` phía sau) và ~34,4 m (`Truck` phía trước) | `Car`, `Truck`, `Cyclist`, `Pedestrian` | Phía sau có 2 xe con đi sát (`cam05`, `cam09`) và 1 xe xa; bên phải trên làn xe đạp/vỉa hè có người đi xe hai bánh và người đi bộ (`cam07`, `cam03`) | Các cụm điểm mảnh chạy dọc dải phân cách cây xanh bên phải và xe bị khuất hoàn toàn sau xe con đi ngay phía trước |

**3 điều chỉ camera cho biết (từ `s01f01` và `s01f55`):**
1. Phân biệt rõ đầu và đuôi xe (nhờ đèn pha, lưới tản nhiệt, kính lái so với đèn hậu) để đặt đúng hướng `yaw` tránh ngược 180°.
2. Phân loại chính xác giữa xe chở hàng (`Truck` thùng kín ở `s01f01` `cam03`, xe van ở `s01f55`), xe buýt (`Bus` ở `cam07`/`cam08`) và `Car`, cũng như phân biệt người đang lái xe hai bánh (`Cyclist`) với xe dựng không người.
3. Nhận diện ngữ cảnh giao thông như vạch kẻ đường, dải phân cách, hàng rào, bụi cây hay màu sắc phương tiện.

**3 điều chỉ LiDAR cho biết (từ `s01f01` và `s01f55`):**
1. Tọa độ 3D và khoảng cách mét chính xác (`dist_m`) của mọi vật thể xung quanh 360°, kể cả vùng góc mù giữa các camera.
2. Kích thước hình học thật `(length, width, height)` theo đơn vị mét và cao độ mặt đường (`z_bottom ≈ −1,7 m` đến `−2,0 m`) mà ảnh 2D bị biến dạng phối cảnh không đo trực tiếp được.
3. Độ che khuất hình học (vùng bóng trống phía sau vật lớn) và khoảng trống thực tế giữa các xe đang dừng nối đuôi nhau.

Tám frame lướt:

| Frame | Phía trước ở cạnh nào của ảnh nhìn từ trên xuống | Điều lạ (nếu có) |
|---|---|---|
| `s02f04` | Cạnh trên (`−y`) | Xe buýt lớn sát phía sau bên trái tạo vùng bóng che rất rộng; hai xe con đi ngay trước bị che mặt trước |
| `s02f25` | Cạnh trên (`−y`) | Nhà chờ xe buýt và bảng quảng cáo lớn sát lề phải tạo cụm điểm hình hộp dễ nhầm với `Truck` nếu không đối chiếu camera |
| `s03f05` | Cạnh trên (`−y`) | Hàng rào công trình chạy dài hai bên làn đường tạo dải điểm thẳng đứng rất dày |
| `s03f43` | Cạnh trên (`−y`) | Ít xe xung quanh (7 vật thể); xe điện trắng áp sát bên phải che gần hết góc nhìn về phía lề phải |
| `s04f21` | Cạnh trên (`−y`) | Ngã tư đông mật độ cao nhất (25 vật thể) với nhiều hướng di chuyển khác nhau (thẳng, rẽ phải, cắt ngang) |
| `s04f38` | Cạnh trên (`−y`) | Lưu lượng trung bình (8 vật thể), chủ yếu xe con dọc trục đường chính |
| `s05f09` | Cạnh trên (`−y`) | Đường rộng với 13 vật thể phân bố đều cả phía trước và phía sau |
| `s05f38` | Cạnh trên (`−y`) | Lưu lượng vừa phải (8 vật thể), nhiều xe cách khá xa (> 25 m) |

## 3. Ba cuboid có hướng dẫn (Bước 3)
Ảnh minh họa: `t2-a-car.png`, `t2-b-bus-truck.png`, `t2-c-ped-cyclist.png`.

| Vật | Frame, idx | Class | Mũi tên yaw đúng hướng? | Đáy chạm đường? | Đã kiểm Top/Side/Front + ảnh? | Kích thước trong bảng 4.3? |
|---|---|---|---|---|---|---|
| (a) xe con gần | `s01f01`, `#6` | `Car` | Đúng (`267,5°`, cùng chiều `−y`) | Có (`z_bottom = −1,800 m`, `ground_gap_m = 0,01 m`) | Đã kiểm Top/Side/Front và `cam06`/`cam01` | Trong khoảng (`4,400 × 1,860 × 1,500 m`) |
| (b) xe buýt hoặc xe tải | `s01f01`, `#5` | `Bus` | Đúng (`1,0°`, hướng về `+x` chờ đèn) | Có (`z_bottom = −1,590 m`, `ground_gap_m = 0,06 m`) | Đã kiểm Top/Side/Front và `cam07`/`cam08` | Trong khoảng (`12,480 × 2,700 × 3,000 m`) |
| (c) người đi bộ hoặc xe hai bánh | `s01f01`, `#8` | `Cyclist` | Đúng (`283,0°`, đi về phía trước) | Có (`z_bottom = −1,670 m`, `ground_gap_m = 0,01 m`) | Đã kiểm Top/Side/Front và `cam08`/`cam07` | Trong khoảng (`1,820 × 0,800 × 1,650 m`) |

- **Vòng hiệu chuẩn & các vật bỏ qua:** Đối chiếu với task chung `D13-HC485087` giúp chốt rõ 3 nguyên tắc: (1) luôn đặt đáy cuboid chạm sát mặt đường địa phương thay vì bám theo điểm thấp nhất lơ lửng của thân xe, (2) tách riêng từng `Pedestrian` khi hai người đi cạnh nhau thay vì gộp chung, và (3) với xe chỉ hứng LiDAR một phần (đầu hoặc đuôi), phải suy luận đủ chiều dài/rộng hợp lý theo bảng 4.3.
- **Vật bỏ qua vì không đủ cơ sở:** Ở `s01f55`, bỏ qua xe con thứ hai đi ngay phía trước xe con trắng cùng làn (`x ≈ 0,1 m, y ≈ −23 m`) vì bị che khuất gần như hoàn toàn (chỉ còn 1 điểm LiDAR, loại nghi vấn `LOW_VISIBILITY`); ở `s02f04`, bỏ qua cụm xe buýt ngược chiều ở rất xa (`y ≈ −48 m`, chỉ có một dải điểm mỏng < 15% thể tích, `UNCERTAIN_SCOPE` / `LOW_VISIBILITY`).

## 4. Bốn hiện tượng chất lượng (Bước 4)

| Hiện tượng | Frame / tên ảnh / nguồn | Nguyên nhân (giả thuyết) | Thấy gì trên point cloud | Hệ quả khi gán (theo guideline) |
|---|---|---|---|---|
| 1. Bóng che | `s02f04` / `t4-1-bong-che.png` / frame của tôi | Giả thuyết: thân xe buýt lớn sát bên trái (`dist_m ≈ 13,6 m`) chắn toàn bộ tia laser chiếu về góc phía sau bên trái | Sau thân xe buýt là một hình quạt trống hoàn toàn không có điểm mặt đường hay vật thể, trong khi hai bên vẫn dày điểm | Vùng trống là chưa quan sát được chứ không phải đường trống; nếu có vật bị che một phần thì bật `occluded = 1` và suy luận đủ kích thước hộp |
| 2. Thưa ở xa | `s02f25` (`#8 Car`) / `t4-2-thua-o-xa.png` / frame của tôi | Giả thuyết: mật độ tia LiDAR phân kỳ theo khoảng cách khiến ở tầm `45,10 m` chỉ còn rất ít tia trúng thân xe | Xe chỉ còn `74` điểm rời rạc tạo đường viền mờ, không còn rõ biên nóc và bánh xe | Vẫn vẽ vì nhìn thấy > 15–20% thể tích và xác nhận được trên `cam09`, kết hợp bảng 4.3 để giữ kích thước `4,30 × 1,85 × 1,50 m`; nếu quá mờ không đủ cơ sở thì bỏ qua và ghi `LOW_VISIBILITY` |
| 3. Phản xạ thấp / ít điểm bất thường | `s03f05` (`#1 Car`) / `t4-3-phan-xa-thap.png` / frame của tôi | Giả thuyết: bề mặt sơn tối màu (xe màu đen) và kính chắn gió hấp thụ hoặc phản xạ gương tia laser thay vì tán xạ ngược về cảm biến | Dù ở khoảng cách gần (`dist_m ≈ 10,5 m`), vùng kính hậu và thân trên của xe đen thưa điểm hơn rõ rệt so với biển số hay đèn phản quang | Không thu hẹp hộp theo vùng bị khuyết điểm ở kính/nóc mà vẫn bao trọn thân xe theo ảnh `cam03` và kích thước thực tế; chỉ ghi nguyên nhân ở mức giả thuyết |
| 4. Điểm lạ | `s02f35` / `t4-4-diem-la.png` / task chung `D13-HC485087` | Giả thuyết: phản xạ đa đường (multipath) từ mặt đường hoặc bề mặt nhẵn khiến thời gian bay của tia kéo dài bất thường | Khi nhìn ngang Y–Z, xuất hiện cụm điểm nằm hẳn dưới mặt đường (xuống tới `z ≈ −13,5 m`, thấp hơn mặt đường quanh đó > 11 m) | Không bao giờ gán nhãn các điểm lạ này thành vật thể và không để chúng kéo lệch ước lượng mặt đường |

## 5. Số từ cuboid của tôi (Bước 5)

Ba cuboid đại diện (lấy từ `submission/D13-2A202604040-bangA.csv`):

| Cuboid (frame, idx, class) | z (m) | h (m) | z đáy = z − h/2 (m) | dist_m (m) | r = √(x²+y²+z²) (m) | yaw_deg (°) | Nhận xét (lơ lửng / chìm / chạm đường) |
|---|---|---|---|---|---|---|---|
| `s01f01`, `#6`, `Car` | −1,050 | 1,500 | −1,800 | 4,19 | 4,31 | 267,5 | Chạm đường (`ground_gap_m = 0,01 m`) |
| `s01f01`, `#5`, `Bus` | −0,090 | 3,000 | −1,590 | 32,14 | 32,14 | 1,0 | Chạm đường (`ground_gap_m = 0,06 m`) |
| `s02f04`, `#3`, `Car` | −1,060 | 1,520 | −1,820 | 2,98 | 3,17 | 267,0 | Chạm đường (`ground_gap_m = 0,00 m`) |

- **(a) Cuboid gần nhất và xa nhất:**
  - **Gần nhất:** `s02f04 #3 Car` có `dist_m = 2,98 m`, khoảng cách 3D `r = 3,17 m` (`z = −1,060 m`).
  - **Xa nhất:** `s02f25 #8 Car` có `dist_m = 45,10 m`, khoảng cách 3D `r = 45,12 m` (`z = −1,310 m`).
- **(b) So sánh kích thước với bảng 4.3:**
  - **Người đi bộ (`s01f01 #1 Pedestrian`):** `l = 0,620 m`, `w = 0,580 m`, `h = 1,780 m` — cả 3 chiều đều nằm gọn trong khoảng người lớn của bảng 4.3 (`0,4–0,9 × 0,5–0,9 × 1,50–1,95 m`).
  - **Xe con (`s01f01 #6 Car`):** `l = 4,400 m`, `w = 1,860 m`, `h = 1,500 m` — cả 3 chiều đều nằm gọn trong khoảng sedan/hatchback của bảng 4.3 (`3,8–5,0 × 1,70–1,90 × 1,40–1,60 m`).
- **(c) Hướng `yaw_deg`:** Xe cùng chiều phía trước `s01f01 #6 Car` có `yaw_deg = 267,5°` (rất gần `270°`); xe ngược chiều `s01f01 #12 Car` có `yaw_deg = 90,0°` (đúng `90°`).

Mật độ theo khoảng cách (4 cuboid `Car` ở 4 dải khoảng cách khác nhau):

| Cuboid (frame, idx, class) | dist_m (m) | Số điểm (`points_in`) |
|---|---|---|
| `s02f25`, `#4`, `Car` | 8,24 | 1967 |
| `s01f01`, `#2`, `Car` | 19,77 | 778 |
| `s04f21`, `#20`, `Car` | 29,37 | 240 |
| `s03f43`, `#5`, `Car` | 36,95 | 91 |

- **Quy luật mật độ & so với slide bài giảng:** Khi khoảng cách tăng gấp ~2,4 lần (từ `8,24 m` lên `19,77 m`) số điểm giảm còn ~40% (`1967 → 778`), và khi xa gấp ~4,5 lần (`36,95 m`) số điểm chỉ còn ~4,6% (`91` điểm) do tia LiDAR phân kỳ theo cả phương ngang và dọc (mật độ giảm gần theo bình phương khoảng cách). Các mốc này khớp rất sát đường cong trung vị trên slide bài giảng (`1968` ở 0–10 m, `718` ở 10–20 m, `214` ở 20–30 m, `72` ở 30–40 m).
- **Tự kiểm và chỉnh sửa sau `preview` / `selfcheck`:**
  - **2 cuboid đã sửa:** (1) `s01f01 #15 Truck`: ở bản vẽ đầu bị đặt lệch sang cụm cột đèn giao thông (`x ≈ 7,7 m, y ≈ −27,4 m`), sau khi kiểm `preview` `cam03` đã dời chính xác về sau hàng xe con (`x = 12,560 m, y = −39,330 m`, `6,15 × 2,30 × 3,05 m`); (2) `s01f55`: sau khi chạy `selfcheck` phát hiện hộp xe khuất phía trước chỉ có `1` điểm LiDAR (`rất ít điểm`), đã xóa hộp này theo đúng quy tắc độ rõ tối thiểu của guideline.
  - **Cảnh báo giữ nguyên (4/123):** `s01f01 #10 Pedestrian` và `s03f43 #6 Pedestrian` báo có điểm sát ngoài mép vì đứng sát người đi bộ / người đi xe đạp kế bên; `s04f21 #14 Car` (`height = 2,33 m`) là xe van chở người thân cao; `s04f21 #17 Car` báo chìm mặt đường do nằm giữa nhiều xe đỗ sát nhau làm vòng ước lượng mặt đường 4 m bị nhiễu bởi điểm thân xe lân cận.
- **Đổi kiểm:** Đã đổi kiểm chéo trên màn hình (không gửi file bảng A) với bạn có MSSV `2A202604045`. Hai con số bạn chọn đối chiếu trực tiếp với `submission/D13-2A202604040-bangA.csv` và `submission/D13-2A202604040-tukiem.csv`: `s01f01 #6 Car` (`dist_m = 4,19 m`, `z_bottom = −1,800 m`, `points_in = 2993`) và `s02f04 #3 Car` (`dist_m = 2,98 m`, `z_bottom = −1,820 m`, `points_in = 2353`) — kết quả **khớp hoàn toàn** giữa báo cáo và hai file CSV.

## 6. Ý nghĩa của nhãn, cách dùng AI và điều còn chưa chắc
- **(a) Ý nghĩa từ phía model 3D:** Model phát hiện vật thể 3D dùng cho xe tự hành cần tọa độ và kích thước theo đơn vị mét để tính chính xác khoảng cách va chạm, quỹ đạo và không gian chiếm dụng trong thế giới thực — điều mà hộp 2D theo pixel không thể cung cấp do phụ thuộc tiêu cự và biến dạng phối cảnh. Vật thưa điểm cần quy tắc riêng (ngưỡng 15–20% thể tích, ghi `LOW_VISIBILITY` thay vì đoán bừa) để tránh đưa nhãn nhiễu hoặc kích thước sai lệch vào tập huấn luyện làm model học sai hình dạng vật thể.
- **(b) Cách dùng AI:**
  - **Công cụ:** Trợ lý AI Antigravity (hỗ trợ viết script Python phân cụm point cloud bằng `numpy`/`scipy`, đối chiếu hình chiếu 7 camera và kiểm tra số liệu `bangA.csv`/`tukiem.csv`).
  - **Câu hỏi chính:** Hỗ trợ tự động phát hiện cụm điểm LiDAR, chuẩn hóa góc `rz` về `[-π, +π]` theo định dạng CVAT 3D, đối chiếu kết quả gán nhãn với task tham chiếu và tổng hợp số liệu báo cáo.
  - **Cách sử dụng kết quả:** Không chấp nhận mù quáng kết quả tự động; mình trực tiếp mở CVAT và ảnh `preview` 7 camera để kiểm từng frame, tự chỉnh lại class/vị trí trên CVAT và yêu cầu sửa toàn bộ hộp lệch, hộp chìm đáy và bổ sung vật bị sót trước khi đồng bộ lại bằng `lab.py sync`.
- **(c) Điều còn chưa chắc:** Một số xe van kín cỡ trung như ở `s04f21 #14` (`height = 2,33 m`) nằm ở ranh giới giữa `Car` (van chở người) và `Truck` (van chở hàng), và một vài xe ở rất xa (> 35 m) bị che khuất phần đuôi nên chiều dài cuboid vẫn mang tính ước lượng theo bảng 4.3.
