# Bài 2: Workloads & Scaling (Quản lý và Mở rộng Ứng dụng)

Ở Bài 1, chúng ta biết Pod là đơn vị nhỏ nhất, nhưng Pod lại rất mỏng manh và dễ chết. Do đó, K8s tạo ra các bộ "Quản đốc" (Controllers / Workloads) đứng trên Pod để chăm sóc chúng. Bạn gần như KHÔNG BAO GIỜ tự tay tạo ra một Pod trần trụi, mà bạn sẽ nhờ các "Quản đốc" này tạo giúp.

## 1. ReplicaSet (Bộ Nhân Bản)
**Vấn đề:** Tôi muốn ứng dụng web của mình luôn luôn có chính xác 3 bản sao chạy song song để chịu tải.
**Giải pháp:** ReplicaSet.
- ReplicaSet là một bộ quản lý đảm bảo rằng luôn có MỘT SỐ LƯỢNG CHÍNH XÁC các Pod đang chạy ở mọi thời điểm.
- Nếu bạn bảo ReplicaSet chạy 3 Pod. Đột nhiên 1 Worker Node chết kéo theo 1 Pod chết (còn 2). ReplicaSet sẽ ngay lập tức phát hiện sự chênh lệch (Hiện tại: 2, Mong muốn: 3) và ra lệnh cho API server đẻ thêm 1 Pod mới tinh ở Node khác để bù vào.
- *Thực tế*: Người ta rất hiếm khi tự tạo ReplicaSet trực tiếp. K8s cung cấp một cấp độ cao hơn gọi là **Deployment**.

## 2. Deployment (Bộ Triển Khai) - Thường dùng nhất
**Vấn đề:** Tôi muốn ứng dụng web của tôi có 3 bản sao, nhưng tuần sau tôi muốn update nó lên version 2. Làm sao để update mà không làm sập toàn bộ web?
**Giải pháp:** Deployment.
- Deployment bao bọc lấy ReplicaSet. Khi bạn deploy ứng dụng *Stateless* (ứng dụng Web, API không lưu trạng thái dữ liệu), bạn sẽ dùng Deployment.
- **Tính năng cực đỉnh của Deployment:**
  - **Rolling Update**: Nếu bạn muốn đổi version Image từ `web:v1` lên `web:v2`, Deployment sẽ không tắt hết 3 Pod v1 đi ngay. Nó sẽ tạo từ từ 1 Pod v2, đợi Pod v2 chạy ổn, nó mới tắt bớt 1 Pod v1. Cứ thế lặp lại. Web của bạn được cập nhật mà người dùng KHÔNG HỀ BỊ RỚT MẠNG vài giây nào (Zero-downtime).
  - **Khôi phục (Rollback)**: Nếu lỡ lên version mới mà v2 bị lỗi (code dỏm), bạn chỉ cần gõ 1 lệnh, Deployment sẽ lùi (rollback) về cấu hình ReplicaSet của v1 ngay tắp lự.

## 3. StatefulSet (Bộ Lưu Trạng Thái)
**Vấn đề:** Ứng dụng của tôi là Database (ví dụ MySQL, Elasticsearch). Deployment không hợp vì Deployment đổi tên Pod liên tục và cháo đổi Storage lộn xộn mỗi khi tạo lại Pod mởi. Database cần sự ĐỊNH DANH ỔN ĐỊNH.
**Giải pháp:** StatefulSet.
- Dành riêng cho các ứng dụng có trạng thái (Stateful).
- Các Pod tạo ra từ StatefulSet có tên CỐ ĐỊNH, đếm theo số thứ tự từ 0. Ví dụ: `mysql-0`, `mysql-1`, `mysql-2`.
- Nếu `mysql-1` vô tình bị chết, StatefulSet sẽ tái tạo lại chính xác một Pod mới tên là `mysql-1`.
- Quan trọng nhất: Pod mới `mysql-1` sẽ được tự động GẮN LẠI đúng cái ổ cứng lưu trữ cũ của `mysql-1` vừa chết chứ không bị mất data.
- Các Pod khởi động có quy củ (0 chạy xong mới mang 1 chạy). Dễ dàng thiết lập Master-Slave.

## 4. DaemonSet (Bộ Thường Trực)
**Vấn đề:** Tôi có một phần mềm thu thập log (Filebeat) hoặc một phần mềm chống virus. Tôi muốn HỄ CỨ BẬT PHIÊN BẢN MÁY CHỦ BẤT KỲ LÊN (Bất kì Worker Node nào) là phải có một bản sao ứng dụng đó tự chạy ngầm ở dưỡi.
**Giải pháp:** DaemonSet.
- Đảm bảo rằng CÓ ĐÚNG MỘT (và chỉ 1) Pod của ứng dụng được chạy trên MỌI Node (hoặc một số Node cụ thể) trong Cluster.
- Nếu bạn gắn thêm 1 máy chủ vật lý mới vào cluster, DaemonSet tự động ném 1 Pod log vào máy đó cắm rễ hoạt động.

## 5. Job & CronJob (Công Việc)
- **Job**: Nếu Web Server chạy mãi mãi không ngừng, thì Job chạy xong là phải TẮT đi. Định nghĩa để chạy một tác vụ tính toán (vd: tính xong báo cáo tháng hoặc render xong 1 frame video), lúc xong báo "Completed" và thôi chạy. Nếu lỗi (như crash code), Job khởi động lại Pod đến khi nào success thì thôi.
- **CronJob**: Như Job, nhưng lên lịch thời gian (ví dụ: Job backup Database lúc 2h sáng).

## Tóm gọn
- Dùng cho Web/API: **Deployment**
- Dùng cho Database có data: **StatefulSet**
- Dùng cho Background Process chạy đều mọi nơi: **DaemonSet**
- Chạy task xong tắt: **Job/CronJob**
