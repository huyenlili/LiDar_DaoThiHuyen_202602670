# BÁO CÁO THỰC NGHIỆM HIỆU CHUẨN ĐA CẢM BIẾN LIDAR-CAMERA
**Phương pháp:** TLC-Calib (3D Gaussian Splatting) trên Dataset KITTI-360  
**Chuỗi dữ liệu thực nghiệm:** `large_rotation` (156 ảnh huấn luyện, 160 ảnh kiểm thử)

---

## 1. VẤN ĐỀ & ỨNG DỤNG THỰC TẾ (PROBLEM)
* **Ứng dụng thực tế:** Hệ thống xe tự hành (Autonomous Driving) bắt buộc phải tích hợp và đồng bộ hóa chặt chẽ dữ liệu từ cảm biến LiDAR (mây điểm 3D) và hệ thống đa Camera Rig (2D) để định vị, nhận diện vật thể và xây dựng bản đồ không gian ba chiều thời gian thực.
* **Vấn đề phát sinh:** Do rung lắc cơ học liên tục trong quá trình di chuyển hoặc va chạm vật lý, ma trận ngoại tham số (Extrinsic Matrix) giữa các cảm biến bị lệch hướng nghiêm trọng. Ở trạng thái ban đầu, sai số góc quay tích lũy lên tới **3.7955°** và sai số tịnh tiến lên tới **1.0476 m**.
* **Hậu quả hệ thống:** Mây điểm LiDAR bị chiếu lệch pha hoàn toàn trên ảnh camera, phá hủy cấu trúc hình học chồng khớp dẫn đến việc các thuật toán nhận diện vật thể 3D bị sai lệch khoảng cách vật lý nghiêm trọng.

---

## 2. PHƯƠNG PHÁP & THÔNG TIN TRUY VẾT KỸ THUẬT (METHOD)
* **Thuật toán áp dụng:** **TLC-Calib** (Targetless LiDAR-Camera Calibration) - Một giải pháp hiệu chuẩn tự động không cần bia mục tiêu dựa trên hạ tầng trường bức xạ **3D Gaussian Splatting (3DGS)**.
* **Đầu vào (Input):** Ảnh từ hệ thống 4 Camera (CAM 00 đến CAM 03), đám mây điểm LiDAR thô, ma trận ngoại tham số ban đầu bị áp nhiễu mạnh.
* **Đầu ra (Output):** Ma trận ngoại tham số hiệu chỉnh tối ưu cho từng camera rig và mô hình phục dựng trường bức xạ 3D của môi trường.
* **Thông tin truy vết thực nghiệm:**
  * *Repository nguồn:* `SNU-VGILab/TLC-Calib`
  * *Tập lệnh chạy tối ưu hóa:* 
    ```bash
    python train.py -s data/TLC-Calib/KITTI-360/large_rotation -m outputs/kitti-360/large_rotation/eval --eval --from_lidar --use_rig --opt_pose --pose_scheduler --adaptive_voxel --dataset kitti-360
    ```
  * *Tài nguyên & Thời gian huấn luyện:* Tối ưu hóa hoàn toàn 30,000 vòng lặp (Iterations) mất **39 phút 42 giây** trên GPU NVIDIA Tesla T4.

---

## 3. KẾT QUẢ BENCHMARK CÓ ĐỐI CHỨNG (QUANTITATIVE RESULTS)

| Điều kiện thực nghiệm | Tham số nhiễu tác động | Sai số góc quay trung bình ($E_R$) | Sai số tịnh tiến trung bình ($E_T$) | Test PSNR (Chất lượng ảnh phục dựng) | Đánh giá & Kết luận |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **Trạng thái lỗi ban đầu (Initial)** | Áp nhiễu ngẫu nhiên góc quay lớn lên Rig cảm biến | **3.7955°** | **1.0476 m** | *Không thể kết xuất (N/A)* | Ảnh chiếu LiDAR lên Camera bị lệch nghiêm trọng, không khớp hình học. |
| **TLC-Calib Tối ưu (Ours_30000)** | Không chủ động gây lỗi, tối ưu hóa liên tục 30,000 iter | **0.1094°** *(Giảm 97.1%)* | **0.1269 m** *(Giảm 87.9%)* | **22.4868 dB** | Thuật toán tự động căn chỉnh góc xoay đạt độ chính xác cận hoàn hảo; chất lượng ảnh phục dựng sắc nét. |

* **Bằng chứng lưu trữ:** Đã kết xuất thành công **624 tệp ảnh đối chứng** (`compares` / `renders`) và kết quả định lượng lưu trữ tại tệp tin `rig_results.json` nằm trong thư mục `/content/drive/MyDrive/LiDar_DaoThiHuyen_202602670/TLC-Calib_outputs`.

---

## 4. PHÂN TÍCH FAILURE CASE & HẠN CHẾ (FAILURE CASE)
* **Quan sát trực nghiệm:** Mặc dù sai số góc quay được giải quyết xuất sắc về mức cận lý tưởng (~0.1 độ), sai số tịnh tiến trung bình vẫn còn neo ở mức khá cao là **12.69 cm** (chưa đạt tiêu chuẩn công nghiệp đòi hỏi độ chính xác dưới 5 cm).
* **Nguyên nhân vật lý sâu xa:** Chuyển động xoay xe cua gấp liên tục trong chuỗi dữ liệu `large_rotation` tạo ra các vùng mù tạm thời (occlusion) lớn giữa các khung nhìn camera kề nhau. Lực gradient truyền ngược của 3DGS ưu tiên tối ưu hóa xoay hướng trục trước để giảm nhanh hàm loss hình ảnh, khiến việc tinh chỉnh tịnh tiến nhỏ rơi vào cực trị địa phương (local minima).
* **Hạn chế thực nghiệm:** Bài thử nghiệm mới chỉ đánh giá trên 1 chuỗi dữ liệu trong thời tiết lý tưởng, chưa bao quát được các điều kiện thời tiết khắc nghiệt (mưa, sương mù làm suy hao mây điểm LiDAR).

---

## 5. QUYẾT ĐỊNH KỸ THUẬT & ĐỀ XUẤT CẢI TIẾN (ENGINEERING DECISIONS)
* **Đánh giá kiến trúc:** Thuật toán đòi hỏi chi phí tài nguyên tính toán cực kỳ lớn (40 phút chạy trên GPU T4 cho một chuỗi ngắn). Vì thế, **không phù hợp** để triển khai chạy thời gian thực (real-time) liên tục trên thiết bị biên (Edge Devices) có cấu hình thấp như Jetson Nano/Xavier.
* **Quyết định thiết kế hệ thống (Trade-off):**
  1. Triển khai TLC-Calib thành một **tác vụ chạy nền định kỳ (Offline Background Task)**. Thuật toán chỉ tự động kích hoạt khi xe ở trạng thái đỗ tĩnh hoặc di chuyển ổn định trên đường thẳng nhằm cập nhật định kỳ ma trận ngoại tham số.
  2. **Đề xuất cải tiến tiếp theo:** Tích hợp thêm ràng buộc mượt mà thời gian (Temporal Smoothness Constraint) trực tiếp vào hàm mất mát (Loss Function) để kéo giảm sai số tịnh tiến xuống dưới ngưỡng mục tiêu 5 cm.


