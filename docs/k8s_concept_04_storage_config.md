# Bài 4: Storage và Configuration (Trí nhờ và Bí mật)

Chúng ta có App chạy, chúng kết nối với nhau êm ái, giờ là lúc nói chuyện dữ liệu trên Kubernetes.

## 1. Lưu Trữ 
Tại sao Storage quan trọng? Nếu bạn chạy ứng dụng MySQL dạng Pod, tạo database trong đó. Rồi ngày nọ Pod bị khởi động lại. BÙM. Tất cả Database sẽ bốc hơi. Bởi vì không gian đĩa nội bộ trong Pod gắn liền vòng đời của Pod. Pod chết = Mất trắng.

Do đó sinh ra bộ 3 khái niệm PV, PVC, Storage Class.

### A. PersistentVolume (PV) - Mảnh Đất Trống
PV là một khối dung lượng lưu trữ TỒN TẠI ĐỘC LẬP khỏi vòng đời bất kì Pod nào. Hiểu đơn giản, nó là một phân vùng ổ cứng vật lý được mount vào hệ thống.
PV có các loại cắm khác nhau như: Gắn ổ đĩa AWS (EBS), gắn vào NFS Server nhà trồng 1 mạng LAN, Gắn vào thư mục trần trên local máy chủ (`hostPath`), ...

### B. PersistentVolumeClaim (PVC) - Lá đơn xin cấp đất
Lập trình viên viết ứng dụng không quan tâm công ty dùng mạng NFS rườm rà hay AWS đắt tiền. Họ chỉ muốn nói "Hey, tui cần cái ổ cứng 10GB xài chung cho Pod của mình (ReadWriteOnce)". Đó là lúc họ nộp một Lá Đơn gọi là PVC.
K8s sẽ đi tìm một Mảnh đất PV nhàn rỗi có tối thiểu 10GB để tự động Nối (Binding) vào PVC này. Người viết ứng dụng sau đó chỉ cần điền tên cái PVC vào cấu hình Deployment/StatefulSet thì dữ liệu sẽ lưu qua ổ PV. Lỡ Pod chết -> Dữ liệu vào PV vĩnh viễn. Tạo Pod mới trỏ vào cái r PVC cũ -> Đọc lại từ PV cũ.

### C. StorageClass - Cò mồi xây dựng tự động phát cạn
Admin hệ thống chả rảnh mà ngồi tự tay tạo 1000 cái PV (mảnh đất không) chỉ để chờ đống claim PVC của Dev. Nên admin sinh ra StorageClass (SC). Cứ có SC thì Dev tung PVC hệ thống tự động gọi API xuông đám mây (AWS/GCP/vSphere...) chia ổ 10GB về gắn luôn. Cực đỉnh. (Dynamic Provisioning).

---

## 2. ConfigMap và Secret
1 ứng dụng Frontend khi dev ở máy bàn khác (gọi API local), lúc lên Product lại trỏ IP khác. Nếu Hardcode (đóng rắn link đó vào source code để build Image), lúc đổi môi trường thì ta vỡ mặt ngậm ngùi Build lại hình ảnh khác tốn thời gian.
1 Database có Password. Nếu ai push file source code lộ Password lên mạng Github mở là hỏng việc.

Kế hoạch K8s tách code ra Code mà Cấu Hình ra Cấu Hình.

### A. ConfigMap - Bảng Tin Cấu Hình
Là tờ giấy dán tường lưu dưới dạng Key-Value (Chìa khóa-Giá trị). (VD: `COLOR = blue`, `API_URL = google.com`).

Thay vì đổi mã Nguồn, bạn tạo 1 ConfigMap, sao đó gắn dính `ConfigMap` này vào Pod.
- Cách 1: Bắn thành các **Biến Môi Trường (Environment Variables)** vào trong Container. Image đọc tự do.
- Cách 2: Sinh ra 1 **File gắn (Volume mount)** như tờ giấy găm vào thư mục `/etc/config` bên trong Container. Container đọc file đó chạy.

Lúc đổi cấu hình, chỉ sửa ConfigMap và Restart lại Pod. KHÔNG CẦN BUILD LẠI IMAGE.

### B. Secret - Két an toàn 
Giống hệt ConfigMap, cũng là Key-Value, nhưng sinh ra để cất giấu các thứ nhạy CẢM như Token, Mật Khẩu, SSH keys, SSL/TLS Certificates...
Điểm khác biệt:
- K8s mặc định cố gắng giữ Secret riêng tư trong bộ nhớ đệm ẩn ẩn, hạn chế viết rõ xuống đĩa cứng vật lý trừ Etcd (khuyến khích bạn lấy plugin cấu hình để etcd Mã hóa file nó luôn).
- Tất cả các giá trị trong tệp bạn tải lên khi viết Secret phải được Encode bằng cơ chế mã hóa **Base64** răn rối (Tuy dễ giải, nhưng chỉ tránh các ánh mắt xem trộm tọc mạch bằng tay).

---
*(Kết thúc chuỗi kiến thức K8s. Nắm vững Pod, Deployment, Service vòng qua Ingress và Volume, bạn đã nắm được kiến trúc chạy trơn tru của ngàn công ty toàn cầu).*
