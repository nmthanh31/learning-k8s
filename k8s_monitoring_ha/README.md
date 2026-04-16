# Kiến Trúc Giám Sát K8s Monitor High Availability (HA)

Tài liệu này giải thích chi tiết chức năng của từng file cấu hình và kiến trúc của hệ thống giám sát High-Availability đang được triển khai trên cụm Kubernetes của VNPost.
Hệ thống tuân thủ chuẩn Production với đầy đủ tính năng: Thu thập số liệu (Metrics), Báo động định tuyến (Alerting), và Trực quan hoá (Visualizing).

## Cấu Trúc Tổng Quan

Hệ thống được thiết kế theo Tư duy Microservices chia nhỏ thư mục với 5 cụm thành phần chính:
1. `prometheus/` (Não bộ lưu trữ & Xử lý rules cảnh báo)
2. `alert_manager/` (Trạm phân phối cảnh báo)
3. `grafana/` (Bảng điều khiển trực quan UI)
4. `node_exporter/` (Nhân viên thu thập chỉ số máy chủ - Hardware)
5. `kube_state_metrics/` (Nhân viên thu thập chỉ số nội bộ K8s)

---

## Giải Thích Chi Tiết Từng Thành Phần

### 1. Thư mục `prometheus/` (Gom dữ liệu & Đánh giá Cảnh báo)
Prometheus là "trái tim" của hệ thống giám sát. Nó có nhiệm vụ liên tục kéo (scrape) các thông số (metrics) từ các ứng dụng/node về để lưu trữ và phân tích.

- **`prometheus-config.yaml`**: Chứa ConfigMap với file cấu hình lõi. Nơi đây thiết lập các đối tượng giám sát (Scrape Config) như KSM, Node Exporter, kubelet cAdvisor. Quan trọng nhất, nó định nghĩa toàn bộ các **Luật Cảnh Báo** (Alerting Rules) theo chuẩn O11y (NodeDown, OOMKilled, HighCPU...)
- **`prometheus-rbac.yaml`**: Cấp quyền K8s Role / ClusterRoleBinding (Role-Based Access Control) để Prometheus có đặc quyền gọi API Kubernetes nhằm mục đích quét các Node và Pod tự động (Service Discovery).
- **`prometheus-sts.yaml`**: Triển khai Prometheus dưới dạng **StatefulSet**. Đảm bảo tính lưu trữ bền vững (Persistent Storage).
- **`prometheus-pdb.yaml`**: Pod Disruption Budget (Giới hạn gián đoạn). Ngăn chặn tình trạng cả 2 Pod của Prometheus bị tắt ngang cùng lúc khi bảo trì node, giúp cụm chạy High Availability liên tục.
- **`prometheus-svc.yaml`**: Mở cổng mạng (Service ClusterIP) cho phần còn lại (Ví dụ Grafana) đọc dữ liệu từ Prometheus.

### 2. Thư mục `alert_manager/` (Phân phối Cảnh báo)
Khi Prometheus phát hiện lỗi (dựa trên Alerting Rules), nó không tự gửi tin nhắn, mà đẩy tín hiệu lỗi qua cho Alertmanager để tự động gom nhóm lỗi (Grouping), ức chế lỗi trùng (Inhibition) và định tuyến (Routing).

- **`alertmanager-config.yaml`**: Chứa cơ chế Routing. Quản lý việc gửi tín hiệu cảnh báo nào đi đâu (VD: Channel Telegram, Email, Slack).
- **`alertmanager-secret.yaml`**: Nơi mã hóa bảo mật các đoạn mã cực kỳ nhạy cảm như Telegram Bot Token, Webhook Token, không để lộ lên file text.
- **`alertmanager-sts.yaml`**: Triển khai Alertmanager dưới luồng StatefulSet có Storage, đảm bảo cụm luôn nhớ được cảnh báo nào đã Mute (Silence).
- **`alertmanager-headless-svc.yaml`**: Service nội bộ đặc biệt (ClusterIP: None) đóng vai trò cho phép các Pod của Alertmanager nhận diện được nhau tạo thành mạng lưới ngang hàng (Gossip Clustering).
- **`alertmanager-svc.yaml`**: Service chính để các app và Prometheus gọi đẩy lỗi vào.
- **`alertmanager-pdb.yaml`**: Đảm bảo Alertmanager luôn đủ tối thiểu Pod sống chặn đứng rớt mạng cảnh báo.

### 3. Thư mục `grafana/` (Bảng điều khiển & Trực quan Hóa)
Chức năng chính của Grafana là kết nối vào Prometheus, lấy dữ liệu tĩnh học và hiện thực hoá thành các biểu đồ (Dashboard) tối ưu.

- **`grafana-sts.yaml`**: Triển khai Dashboard Grafana dạng StatefulSet. Đi kèm giới hạn phần cứng (Resource Requests/Limits) chống tràn RAM, và Health Probes chống kẹt tiến trình.
- **`grafana-provisioning.yaml`**: Cơ chế Cấu hình tự động (Provisioning) siêu tối ưu. Ép Grafana nhận Prometheus nội bộ làm Datasource ngay lập tức từ khi khởi động (Infrastructure-As-Code - Không click chuột thủ công).
- **`grafana-secret.yaml` & `grafana-db-creds`**: Mã hóa/lưu cấu hình đăng nhập mặc định và mật khẩu kết nối Database Postgres.
- **`grafana-ingress.yaml`**: Chịu trách nhiệm mở Gateway, tạo Ingress Host phơi bày dịch vụ ra web bằng Domain name thật phục vụ người dùng.
- **`grafana-pdb.yaml` & `grafana-svc.yaml`**: Pod Disruption Budget và Tên miền mạng nội bộ LAN của Kubernetes cho Grafana.
- **`postgres-ha.yaml` & `postgres-sts.yaml`**: PostgreSQL được thiết kế theo cấu trúc High Availability Cluster (CloudNativePG) chuyên nghiệp với 3 nodes. Nơi Grafana chọn làm Database kiên cố chống thất thoát thông tin user thay vì SQLite sơ sài.

### 4. Thư mục `node_exporter/` (Công cụ chỉ số phần cứng)
Node Exporter bám sát vào lõi hệ điều hành bằng Kernel.

- **`node_exporter_daemonset.yaml`**: Áp dụng thiết kế DaemonSet, đảm bảo rằng mỗi khi có VM (Node) mới gia nhập K8s, tự động sinh ra Pod đọc phần cứng (CPU, Disk ảo) liên tục bám theo máy đó mà không qua tay người lệnh lại. Tích hợp Tolerations để giám sát được cả luồng Control-Plane Node.
- **`node_exporter_service.yaml`**: Service tĩnh móc các Daemon set phân mảnh gom lại, cung ứng Endpoints gộp cho thiết bị gom dữ liệu (`/metrics`).

### 5. Thư mục `kube_state_metrics/` (Bác sĩ Kubernetes state)
- **`ksm-deployment.yaml`**: Triển khai dạng Deployment Scale out. Khác bên Node-Exporter đọc OS vật lý, KSM móc dữ liệu trực tiếp với K8s API Master Server để truy kích luồng logic: Có bao nhiêu Pod fail? PVC Pending? OOMKilled diễn ra tần suất sao?
- **`ksm-rbac.yaml`**: Danh sách khổng lồ các giấy phép uỷ quyền ClusterRole. Sự sống còn của KSM nằm ở việc nó được cấp phép "đọc thấu" toàn bộ object trong Cluster. Từ Pod, CronJob, HPA cho đến Ingress.
- **`ksm-svc.yaml`**: Service xuất cảng API mạng chờ Prometheus chạy đến đọc `healthz` và `/metrics`.

---
*Lưu ý O11y Standard: Cấu hình Production toàn bộ Stack trên đều tuân thủ độ khắt khe về Health Probes (Liveness/Readiness), Anti-MemoryLeak (Limits CPU/RAM) và Self-Healing PDB (Tránh Downtime bảo trì máy chủ).*
