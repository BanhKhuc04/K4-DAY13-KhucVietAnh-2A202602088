# Báo cáo thực hành PointPillars (Cá nhân) — Day 13

Bài thực hành cá nhân: Robotaxi LiDAR 3D Object & PointPillars Practice.
Học viên: **Khúc Việt Anh** — MSSV: **2A202602088**

---

## 1. Thông tin và provenance

- **Hình thức thực hiện:** Cá nhân (Solo)
- **Họ và tên:** Khúc Việt Anh
- **MSSV:** 2A202602088
- **Trạng thái thực hiện:** `executed-by-group` (tự chạy trực tiếp trên máy cá nhân)
- **Người thực sự chạy; ngày/giờ; hệ máy/architecture:** Khúc Việt Anh; 2026-10-02 10:04; Windows 11 x86_64, Docker Desktop 29.7.2 (amd64 / WSL2)
- **Image tag và image ID; phiên bản repo:**
  - Image tag: `day13-pointpillars:lc-20261001-amd64`
  - Image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`
  - Phiên bản repo: `0831856d921609312d42c7582c366e5a311bb7b1`
- **PCD được cấp / frame_id:** `demo` (`input/demo.pcd`, sha256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`); chạy offline cục bộ trên máy cá nhân
- **Checkpoint:** PointPillars KITTI có sẵn trong image (`/opt/PointPillars/pretrained/epoch_160.pth`, sha256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`)
- **Phạm vi inference:** front-window (cửa sổ phía trước); **Score threshold:** 0.3
- **Giả định kênh thứ tư/intensity và nguồn z_ground:** PCD KITTI Student bỏ reflectance thật, gán RGB=0; `z_ground = 0.075 m` (ước lượng từ mặt đất của PCD)

---

## 2. Ba lượt inference thật (A / B / C)

| Lượt | delta (m) | Pillar XY (m) | Số hộp (`n_boxes`) | `mean_z` (m) | File đối chiếu (JSON / Side / CSV) | Quan sát có bằng chứng |
|:---:|:---:|:---:|:---:|:---:|:---:|---|
| **A** | 0 | 0.16 | **1** | **0.330** | `run-A/summary.csv`, `side-demo-delta-0.0-voxel-0.16.png` | Chỉ tìm được 1 hộp duy nhất (vehicles). Không dịch z khiến model bỏ sót hầu hết vật thể |
| **B** | 1.73 | 0.16 | **13** | **1.034** | `run-B/summary.csv`, `side-demo-delta-1.73-voxel-0.16.png` | Tìm được 13 hộp (10 vehicles, 1 two-wheels, 2 pedestrian). Mốc chuẩn phân phối KITTI |
| **C** | 1.73 | 0.32 | **6** | **1.091** | `run-C/summary.csv`, `side-demo-delta-1.73-voxel-0.32.png` | Số hộp giảm còn 6 và toàn bộ nhận diện thành pedestrian. Cạnh pillar tăng gấp đôi làm mất chi tiết vật thể lớn |

### Phân tích so sánh:

- **So sánh A ↔ B (Ảnh hưởng của delta z trước inference):**
  - Cấu hình A (`delta = 0`) chỉ ra 1 hộp, cấu hình B (`delta = 1.73`) ra 13 hộp.
  - *Giải thích:* Thay input trước model hoàn toàn khác với dịch cùng một hằng số cho output. PointPillars gom điểm vào các cột (pillars) 3D và trích xuất đặc trưng không gian trước khi đưa vào backbone 2D. Khi delta = 0 ở A, các điểm LiDAR rơi vào sai bin độ cao khiến đặc trưng bị biến dạng và model bỏ sót phần lớn đối tượng. Không thể sửa bằng cách cộng hằng số bù z ở output.
- **So sánh B ↔ C (Ảnh hưởng của kích thước pillar XY):**
  - Cấu hình B (`pillar = 0.16m`) và C (`pillar = 0.32m`):
  - *Quan sát:* Khi tăng kích thước cạnh pillar XY từ 0.16m lên 0.32m, số hộp giảm từ 13 xuống 6 và toàn bộ 10 xe (vehicles) đều biến mất hoặc bị nhận nhầm thành pedestrian.
  - *Kết luận:* C dùng chung checkpoint với B nhưng biểu diễn đầu vào thô hơn; chưa đủ cơ sở để khẳng định cấu hình nào tổng quát tốt hơn nếu không đo benchmark trên toàn tập, nhưng cho thấy voxel 0.32m làm mất chi tiết hình học xe ô tô.
- **Giới hạn ROI và góc Side:**
  - Runner chỉ quét vùng cửa sổ phía trước (front-window), các vật thể hai bên sườn và phía sau bị cắt bỏ hoàn toàn.
  - Ảnh Side là hình chiếu trực giao x-z nên không quan sát được góc xoay yaw quanh trục thẳng đứng hoặc độ lệch theo trục y.
- **Đánh giá JSON để import:**
  - Cả 3 file JSON đều chưa đủ cơ sở để import vào CVAT vì đây là model pretrained KITTI chạy trên tập demo, khác phân phối toạ độ, cảm biến và ngữ cảnh của bài Robotaxi.

---

## 3. Ba ca QC có kiểm soát (qc-cases) — Không import vào CVAT

Các ca này được helper tạo ra từ prediction của lượt B (`boxes-demo-delta-1.73-voxel-0.16.json`, gồm 13 hộp) nhằm mục đích luyện tập phân biệt **lỗi pipeline (batch)** và **lỗi từng cuboid cụ thể**.

| Ca QC | Số hộp lệch z / tổng hộp | Lượng lệch (m) | Class / x / y / yaw có đổi? | Quyết định hành động | Bằng chứng quan sát |
|---|:---:|:---:|:---:|---|---|
| `case-correct` | 0 / 13 | 0 m | Không đổi | Giữ nguyên, đạt chuẩn mốc | Khớp hoàn toàn với prediction lượt B |
| `case-batch-z` | 13 / 13 (toàn bộ batch) | -1.805 m (bị dịch xuống) | Giữ nguyên class, x, y, yaw | **DỪNG BATCH NGAY LẬP TỨC**, yêu cầu kiểm tra phép chuyển hệ trục pipeline | Toàn bộ 13 hộp bị tụt sâu dưới đất đồng loạt một khoảng 1.805m (thiếu bước cộng bù delta + z_ground) |
| `case-one-box-z` | 1 / 13 | -1.805 m ở hộp đầu tiên | Các hộp còn lại nguyên vẹn | **Kiểm tra từng hộp**, không dừng cả batch | Chỉ đúng 1 hộp bị tụt z, 12 hộp còn lại nằm đúng cao độ mặt đường |

> **Ghi chú:** Các ca QC do helper tự động tạo biến đổi có chủ đích từ B, không phải nhãn ground-truth hay lượt inference độc lập.

---

## 4. Nhận xét cá nhân (Khúc Việt Anh — 2A202602088)

1. **Môi trường & Quy trình thực hiện:**
   - Tự vận hành và thực hiện độc lập toàn bộ các bước: thiết lập Docker Desktop trên Windows 11 x86_64, chạy runner PointPillars tạo kết quả 3 lượt A/B/C và 3 ca QC.
   - Trạng thái thực hiện: `executed-by-group` (tự chạy trực tiếp trên máy cá nhân).

2. **Khác biệt quan sát giữa các lượt A/B và B/C:**
   - Giữa A và B: Khi không dịch z (`delta = 0` ở A), model chỉ phát hiện được đúng 1 hộp xe (`mean_z=0.330`), trong khi B (`delta = 1.73`) phát hiện được 13 hộp (`mean_z=1.034`) bao gồm 10 xe, 1 xe hai bánh và 2 người đi bộ.
   - Giữa B và C: Kích thước pillar tăng từ 0.16m lên 0.32m làm giảm độ phân giải không gian trên mặt phẳng XY, số hộp giảm xuống còn 6 và toàn bộ chuyển thành pedestrian, mất toàn bộ 10 xe.

3. **Bản chất phép đổi z thuận / nghịch:**
   - Công thức chuyển đổi:
     $$\mathbf{z_{model} = z_{source} - z_{ground} - delta}$$
     $$\mathbf{z_{source} = z_{model} + z_{ground} + delta}$$
   - *Tại sao thay delta trước inference khác với dịch hộp sau inference?* Vì PointPillars gom điểm vào các cột (pillars) và trích xuất đặc trưng không gian 3D trước khi đưa vào backbone 2D. Nếu không dịch trước, các điểm rơi vào sai bin độ cao, mạng thần kinh nhận đặc trưng sai lệch nên không thể sửa bằng cách cộng bù z ở output.

4. **Đánh giá 3 ca QC và quyết định dừng pipeline:**
   - Với `case-batch-z`: Toàn bộ 13 hộp đều bị lệch đồng loạt 1.805m, kết luận là **lỗi pipeline**. Quyết định: **Dừng batch ngay lập tức** để kiểm tra code pipeline chuyển đổi hệ toạ độ, không sửa tay từng hộp.
   - Với `case-one-box-z`: Chỉ 1 hộp đơn lẻ bị sai cao độ, kết luận là **lỗi đối tượng**. Quyết định: Giữ batch, kiểm tra và chỉnh sửa hình học cho riêng cuboid đó.

5. **Kinh nghiệm thực hành & Điểm còn chưa chắc chắn:**
   - Việc chỉ quan sát qua hình chiếu `side-*.png` (hình chiếu x-z) chỉ giúp đánh giá được cao độ và chiều dài theo trục x, khó khẳng định chính xác góc yaw và độ bao khít nếu không có đầy đủ góc nhìn 3D và camera phối cảnh.

---

## 5. Giảng viên / LC ghi nhận

- **Quyền dùng PCD/image và đúng ca:** Đạt
- **Chạy thực tế / Phân tích kết quả:** Tự chạy trên máy cá nhân (Windows 11 amd64)
- **Dữ liệu & Bằng chứng:** Đầy đủ các file đối chiếu, không import ca lỗi vào CVAT
- **Đánh giá chung:** Đạt yêu cầu thực hành cá nhân Day 13
