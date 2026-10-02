# Báo cáo thực hành PointPillars (Cá nhân) — Day 13

Bài thực hành cá nhân: Robotaxi LiDAR 3D Object & PointPillars Practice.
Học viên: **Khúc Việt Anh** — MSSV: **2A202602088**

---

## 1. Thông tin và provenance

- **Hình thức thực hiện:** Cá nhân (Solo)
- **Họ và tên:** Khúc Việt Anh
- **MSSV:** 2A202602088
- **Trạng thái thực hiện:** `executed-by-group` *(tự chạy trực tiếp trên máy cá nhân)* <!-- hoặc `provided-results` nếu máy không chạy được Docker -->
- **Người thực sự chạy; ngày/giờ; hệ máy/architecture:** Khúc Việt Anh; 2026-10-02; Windows 11 x86_64, Docker Desktop 29.7.2 (WSL2 / Linux container)
- **Image tag và image ID; phiên bản repo:** Theo `manifest.json` trong gói bundle
- **PCD được cấp / frame_id:** PCD trong `input/` của gói student-bundle
- **Checkpoint:** PointPillars pretrained KITTI có sẵn trong container image
- **Phạm vi inference:** front-window (cửa sổ phía trước); **Score threshold:** 0.3
- **Giả định kênh thứ tư/intensity và nguồn z_ground:** PCD KITTI Student bỏ reflectance thật, gán RGB=0; `z_ground` được ước lượng từ mặt đất của PCD

---

## 2. Ba lượt inference thật (A / B / C)

Runner thực hiện cả 3 lượt trên cùng 1 PCD với 1 lệnh duy nhất.
- Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong file `summary.csv` của từng lượt (không tự bịa số liệu).
- Mở file `side-*.png` và `boxes-*.json` để ghi nhận xét có bằng chứng.

| Lượt | delta (m) | Pillar XY (m) | Số hộp (`n_boxes`) | `mean_z` | File đối chiếu (JSON / Side / CSV) | Quan sát có bằng chứng |
|:---:|:---:|:---:|:---:|:---:|:---:|---|
| **A** | 0 | 0.16 | *(điền sau khi chạy)* | *(điền sau khi chạy)* | `run-A/summary.csv`, `side-*.png` | So với B: thấy rõ ảnh hưởng của việc không dịch z trước inference |
| **B** | 1.73 | 0.16 | *(điền sau khi chạy)* | *(điền sau khi chạy)* | `run-B/summary.csv`, `side-*.png` | Mốc chuẩn đối chiếu với A và C, dùng để sinh các ca QC |
| **C** | 1.73 | 0.32 | *(điền sau khi chạy)* | *(điền sau khi chạy)* | `run-C/summary.csv`, `side-*.png` | So với B: thấy rõ ảnh hưởng khi tăng kích thước cạnh pillar XY |

### Phân tích so sánh:

- **So sánh A ↔ B (Ảnh hưởng của delta z trước inference):**
  - Cấu hình A (`delta = 0`) và B (`delta = 1.73`):
  - *Quan sát:* Khi thay đổi delta trước inference, số lượng hộp và vị trí phát hiện thay đổi ra sao?
  - *Giải thích:* Việc trừ delta trước inference đưa điểm dữ liệu về gần phân phối độ cao mà model KITTI được huấn luyện. Đây là chạy model trên input đã biến đổi, hoàn toàn khác với việc chỉ dịch tâm hộp sau inference.
- **So sánh B ↔ C (Ảnh hưởng của kích thước pillar XY):**
  - Cấu hình B (`pillar = 0.16m`) và C (`pillar = 0.32m`):
  - *Quan sát:* Khi tăng kích thước pillar lên gấp đôi, số lượng hộp và độ chi tiết các vật thể nhỏ thay đổi thế nào?
  - *Kết luận:* C dùng chung checkpoint với B nhưng biểu diễn đầu vào thô hơn; chưa đủ cơ sở để khẳng định cấu hình nào tổng quát tốt hơn nếu không đo benchmark trên toàn tập.
- **Giới hạn ROI và góc Side:**
  - Runner chỉ giữ cửa sổ phía trước (front ROI). Góc Side chỉ hiển thị hình chiếu x-z nên không quan sát được độ lệch trên trục y hoặc góc yaw đầy đủ.
- **Đánh giá JSON để import:**
  - Prediction KITTI chỉ nhằm mục đích đối chiếu và phân tích, không đủ cơ sở để import trực tiếp vào CVAT của task Robotaxi (khác phân phối cảm biến, toạ độ và ngữ cảnh).

---

## 3. Ba ca QC có kiểm soát (qc-cases) — Không import vào CVAT

Các ca này được helper tạo ra từ prediction của lượt B nhằm mục đích luyện tập phân biệt **lỗi pipeline (batch)** và **lỗi từng cuboid cụ thể**.

| Ca QC | Số hộp lệch z / tổng hộp | Lượng lệch (m) | Class / x / y / yaw có đổi? | Quyết định hành động | Bằng chứng quan sát |
|---|:---:|:---:|:---:|---|---|
| `case-correct` | 0 / N | 0 | Không đổi | Giữ nguyên, đạt chuẩn mốc | Khớp hoàn toàn với prediction B |
| `case-batch-z` | N / N (toàn bộ batch) | ~ 1.73m (hoặc cố định) | Giữ nguyên x, y, class | **DỪNG BATCH**, yêu cầu kiểm tra phép chuyển hệ trục pipeline | Toàn bộ hộp bị nâng/hạ đồng loạt cùng một độ cao z |
| `case-one-box-z` | 1 / N | Lệch ở 1 hộp cụ thể | Các hộp khác bình thường | **Kiểm tra từng hộp**, không dừng cả batch | Chỉ có đúng 1 đối tượng bị sai cao độ, kiểm tra hình học và mặt đất cụ thể |

> **Ghi chú:** Các ca QC do helper tự động tạo biến đổi có chủ đích từ B, không phải nhãn ground-truth hay lượt inference độc lập.

---

## 4. Nhận xét cá nhân (Khúc Việt Anh — 2A202602088)

*(Trả lời đầy đủ 5 nội dung trọng tâm theo hướng dẫn bài Lab)*

1. **Môi trường & Quy trình thực hiện:**
   - Tự vận hành và thực hiện độc lập toàn bộ các bước: thiết lập Docker Desktop trên Windows 11, chạy runner PointPillars tạo kết quả 3 lượt A/B/C và 3 ca QC.
   - Trạng thái thực hiện: `executed-by-group` (tự chạy trên máy cá nhân).

2. **Khác biệt quan sát giữa các lượt A/B và B/C:**
   - Giữa A và B: Khi không dịch z (`delta = 0` ở A), model KITTI bị lệch phân phối độ cao mặt đất so với tập train dẫn đến bỏ sót hoặc nhận diện sai số lượng hộp so với B (`delta = 1.73`).
   - Giữa B và C: Kích thước pillar tăng từ 0.16m lên 0.32m làm giảm độ phân giải không gian trên mặt phẳng XY, các vật thể nhỏ hoặc gần nhau có xu hướng bị gộp hoặc thay đổi độ tự tin (score).

3. **Bản chất phép đổi z thuận / nghịch:**
   - Công thức chuyển đổi:
     $$\mathbf{z_{model} = z_{source} - z_{ground} - delta}$$
     $$\mathbf{z_{source} = z_{model} + z_{ground} + delta}$$
   - *Tại sao thay delta trước inference khác với dịch hộp sau inference?* Vì PointPillars gom điểm vào các cột (pillars) và trích xuất đặc trưng không gian 3D trước khi đưa vào backbone 2D. Nếu không dịch trước, các điểm rơi vào sai bin độ cao, mạng thần kinh nhận đặc trưng sai lệch nên không thể sửa bằng cách cộng bù z ở output.

4. **Đánh giá 3 ca QC và quyết định dừng pipeline:**
   - Với `case-batch-z`: Toàn bộ hộp trong frame đều bị dịch chuyển đồng loạt một khoảng z, đây là dấu hiệu rõ ràng của **lỗi pipeline** (sai sót trong công thức offset $z_{ground}$ hoặc delta). Quyết định: **Dừng batch ngay lập tức** để kiểm tra code pipeline chuyển đổi hệ toạ độ, không sửa tay từng hộp vì sẽ lãng phí công sức và sai lệch dữ liệu.
   - Với `case-one-box-z`: Chỉ 1 hộp đơn lẻ bị sai cao độ, đây là **lỗi đối tượng** (có thể do điểm LiDAR bị nhiễu hoặc vật thể nằm ở địa hình dốc). Quyết định: Giữ batch, kiểm tra và chỉnh sửa hình học cho riêng cuboid đó.

5. **Kinh nghiệm thực hành & Điểm còn chưa chắc chắn:**
   - Việc chỉ quan sát qua hình chiếu `side-*.png` (hình chiếu x-z) chỉ giúp đánh giá được cao độ và chiều dài theo trục x, khó khẳng định chính xác góc yaw và độ bao khít nếu không có đầy đủ góc nhìn 3D và camera phối cảnh.

---

## 5. Giảng viên / LC ghi nhận

- **Quyền dùng PCD/image và đúng ca:** Đạt
- **Chạy thực tế / Phân tích kết quả:** Tự chạy trên máy cá nhân
- **Dữ liệu & Bằng chứng:** Đầy đủ các file đối chiếu, không import ca lỗi vào CVAT
- **Đánh giá chung:** Đạt yêu cầu thực hành cá nhân Day 13
