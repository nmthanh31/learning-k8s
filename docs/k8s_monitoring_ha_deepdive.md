# Giải Thích Chuyên Sâu Kiến Trúc High Availability (HA) Monitoring

Tài liệu này đi sâu vào cơ chế hoạt động của từng thành phần trong giải pháp HA Monitoring mà chúng ta đã triển khai trong thư mục `k8s_monitoring_ha`.

---

## 1. Prometheus HA: StatefulSet & VolumeClaimTemplates

### Tại sao cần HA cho Prometheus?
Nếu chỉ chạy 1 Pod Prometheus, khi Node đó chết hoặc Pod restart, việc thu thập dữ liệu (scraping) sẽ bị gián đoạn, để lại "khoảng trắng" trên biểu đồ.

### Cơ chế hoạt động trong bản HA:
*   **StatefulSet (STS) thay vì Deployment:** 
    *   Deployment coi Pod là "vô danh" (cattle). STS coi Pod là "danh tính cố định" (pet) với số thứ tự: `prometheus-0`, `prometheus-1`.
    *   Mỗi Pod giữ nguyên tên và ổ đĩa gắn kèm ngay cả khi bị xóa và tạo lại.
*   **VolumeClaimTemplates (VCT):** 
    *   Thay vì tất cả Pod dùng chung 1 PVC (gây lỗi Multi-Attach), VCT yêu cầu Kubernetes tạo ra **nhiều PVC riêng biệt**: `data-prometheus-0` và `data-prometheus-1`.
    *   **Tác dụng:** Cả 2 bản sao cùng ghi dữ liệu độc lập vào 2 ổ đĩa khác nhau. Khi truy vấn, Grafana sẽ lấy dữ liệu từ một trong hai bản sao. Nếu 1 bên hỏng, ổ đĩa của nó vẫn còn nguyên đó, chờ Pod khởi động lại để gắn lại.
*   **Deduplication (Phụ thuộc vào Grafana/Thanos):** Hiện tại chúng ta chạy song song. Điểm yếu nhỏ là dữ liệu 2 bên có thể lệch nhau vài milliseconds, nhưng đổi lại là sự an toàn tuyệt đối về dữ liệu.

---

## 2. Grafana HA: PostgreSQL làm "Não Bộ" Dùng Chung

### Vấn đề của bản cũ:
Mặc định Grafana dùng **SQLite** (file local bên trong container). Khi anh chạy 2 bản sao Grafana, nếu anh tạo Dashboard trên Pod A, Pod B sẽ **không thấy** Dashboard đó vì nó nằm ở ổ đĩa của Pod A.

### Cơ chế hoạt động trong bản HA:
*   **External Database (PostgreSQL):** Chúng ta tách toàn bộ "trạng thái" (Dashboards, Users, Sessions, Alert rules của Grafana) ra một database PostgreSQL riêng.
*   **Hoạt động:** Khi anh đăng nhập vào bất kỳ Pod Grafana nào (`grafana-0` hoặc `grafana-1`), Grafana sẽ truy vấn Postgres để lấy cấu hình. Khi anh ấn "Save" Dashboard, nó ghi vào Postgres. Nhờ đó, cả 2 Pod Grafana luôn nhìn thấy dữ liệu đồng nhất 100%.
*   **Sizing:** Chỉ cần 1 bản sao Postgres có Persistence là đủ để nuôi cụm Grafana HA chạy ổn định.

---

## 3. Alertmanager HA: Mesh Network & Gossip Protocol

### Tại sao 2 bản sao Prometheus lại không gây ra 2 tin nhắn cảnh báo trùng lặp?
Vì Prometheus đẩy cảnh báo tới **Alertmanager Mesh**.

### Cơ chế hoạt động trong bản HA:
*   **Cụm 3 Replicas:** Chúng ta chạy 3 Pod để đảm bảo số lẻ (tránh lỗi split-brain).
*   **Gossip Protocol:** Các Pod liên lạc với nhau qua cổng `9094`. Chúng chia sẻ thông tin: "Tôi đang chuẩn bị gửi tin nhắn cảnh báo InstanceDown này tới Telegram". 
*   **Deduplication (Khử trùng lặp):** Khi Pod 1 nhận thấy Pod 0 đã gửi tin nhắn rồi, nó sẽ đánh dấu là "đã xử lý" và không gửi nữa. Kết quả là anh chỉ nhận được 1 tin nhắn duy nhất dù cả cụm đang giám sát.
*   **High Availability:** Nếu anh tắt 2 trong 3 Pod Alertmanager, hệ thống vẫn gửi được tin nhắn bình thường.

---

## 4. Kube-State-Metrics (KSM): Leader Election

### Vấn đề:
KSM không lưu trữ dữ liệu nhưng nó tạo ra metrics từ API Server. Nếu anh chạy 2 Pod KSM bình thường, Prometheus sẽ lấy được 2 bộ metrics giống hệt nhau, làm các con số (như tổng số Pod) bị nhân đôi (ví dụ Cluster có 10 Pod nhưng Grafana hiện 20 Pod).

### Cơ chế hoạt động trong bản HA:
*   **Leader Election:** Các bản sao KSM tranh giành một cái "khóa" (Lease) trong Kubernetes. 
*   **Active-Standby:** Chỉ Pod nào giữ được khóa (Leader) mới xuất dữ liệu tại cổng `8080`. Pod còn lại sẽ báo lỗi hoặc không trả về dữ liệu metrics khi Prometheus scrape.
*   **Failover:** Nếu Pod Leader chết, Pod Standby sẽ nhận ra khóa đã hết hạn và tự lên làm Leader mới trong vòng vài giây.

---

## 5. Anti-Affinity: Sự Phân Tán Vật Lý

Đây là cấu hình "vô hình" nhưng quan trọng nhất trong file YAML:
*   **Tác dụng:** Ép Kubernetes phải đặt `prometheus-0` ở Node 1 và `prometheus-1` ở Node 2.
*   **Tại sao cần?** Nếu không có cái này, Kubernetes có thể tiện tay đặt cả 2 bản sao lên cùng 1 server vật lý. Khi server đó cháy nguồn, cả "hệ thống HA" của anh sẽ sập cùng lúc. Anti-Affinity đảm bảo hạ tầng giám sát "không bao giờ bỏ tất cả trứng vào một giỏ".

---

## 6. Security Context & fsGroup: Chìa Khóa Quyền Hạn

### Vấn đề:
Nhiều bản cài đặt bị lỗi "Permission Denied" khi ghi vào Volume (như Postgres hay Prometheus). Đó là vì Image chạy bằng user không có quyền truy cập vào ổ đĩa được gắn vào.

### Cách HA xử lý:
*   **fsGroup:** Chúng ta ép Kubernetes đổi quyền sở hữu của toàn bộ ổ đĩa (Volume) cho Group ID của ứng dụng (ví dụ 472 cho Grafana, 999 cho Postgres).
*   **Tác dụng:** Đảm bảo dù anh dùng Storage của bất kỳ hãng nào (NFS, Longhorn, Cinder), ứng dụng vẫn có quyền đọc ghi dữ liệu mà không cần anh phải vào chmod tay.

---

Tài liệu này giúp anh hiểu rõ tại sao bộ file trong `k8s_monitoring_ha` lại phức tạp và nhiều thành phần hơn bản cũ, nhưng đổi lại là sự tin cậy tuyệt đối cho hệ thống vận hành thực tế.
