# K4 - Day 13: Robotaxi LiDAR 3D Object & PointPillars Practice

Repo bài tập thực hành cá nhân — Day 13: Robotaxi LiDAR 3D Object Annotation & PointPillars Practice.

- **Học viên:** Khúc Việt Anh
- **MSSV:** 2A202602088
- **Hình thức:** Cá nhân (Solo)

## Cấu trúc thư mục nộp bài

```
K4-DAY13-KhucVietAnh-2A202602088/
├── README.md
├── .gitignore
└── report/
    ├── PRE-LABEL-REPORT.md   # Báo cáo thực hành PointPillars và phân tích ca QC
    └── TEAMMATES.md          # Thông tin cá nhân học viên thực hiện
```

## Nội dung bài thực hành

1. **PointPillars Inference (A / B / C):**
   - Lượt A: `delta = 0 m`, `pillar_xy = 0.16 m` (Khảo sát việc không dịch z trước inference)
   - Lượt B: `delta = 1.73 m`, `pillar_xy = 0.16 m` (Mốc đối chiếu chuẩn)
   - Lượt C: `delta = 1.73 m`, `pillar_xy = 0.32 m` (Khảo sát ảnh hưởng khi tăng kích thước pillar)
2. **Phân tích lỗi pipeline vs. lỗi đối tượng (3 ca QC):**
   - Đánh giá `case-correct`, `case-batch-z`, `case-one-box-z`
   - Phân tích công thức chuyển đổi cao độ $z$ thuận và nghịch
   - Ra quyết định dừng batch hay kiểm tra từng cuboid
3. **Nạp Pre-label & Chỉnh sửa Cuboid 3D trên CVAT:**
   - Hoàn thiện các job gán nhãn 3D point cloud Robotaxi trên portal
   - Thực hiện QC chéo và nộp phản hồi snapshot v2
