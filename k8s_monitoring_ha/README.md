# ☸️ Kubernetes Monitoring High-Availability (HA) Stack - SRE Standard

Tài liệu này tổng hợp kiến trúc, giải pháp và hướng dẫn vận hành hệ thống giám sát cụm Kubernetes (RKE2/Cilium) theo tiêu chuẩn SRE (Site Reliability Engineering), phục vụ cho mục tiêu đảm bảo tính ổn định cao và khả năng quan sát (Observability) toàn diện.

---

## 🏗️ 1. Tổng quan & Mục tiêu (Overview & Objectives)

### 1.1. Vấn đề (The Problem)

Trong môi trường Kubernetes, các workload thường có tính chất ngắn hạn (ephemeral). Việc giám sát truyền thống không thể đáp ứng khả năng truy vết sự cố và quản lý tài nguyên động. Hệ thống này được thiết kế để giải quyết:

- **Tính sẵn sàng cao (HA):** Giám sát không được phép "chết" trước khi hệ thống chính gặp sự cố.
- **Khả năng quan sát tầng sâu:** Từ hạ tầng (Node) đến logic K8s (Pods) và mạng (Cilium eBPF).
- **Cảnh báo thông minh:** Giảm nhiễu (Alert Fatigue) và định tuyến thông báo hiệu quả.

### 1.2. Hạ tầng mục tiêu

- **Cụm K8s:** RKE2 với 03 Node Control Plane và 03 Node Worker.
- **Network:** Cilium CNI với Hubble hỗ trợ giám sát luồng mạng mức nhân Linux.

---

## 🖼️ 2. Kiến trúc giải pháp (Architecture)

Hệ thống sử dụng mô hình **Pull-based Monitoring** với sự phối hợp của các thành phần hàng đầu:

![picture1](/imgs/Picture1.png)

### Điểm nhấn kỹ thuật (Key Features):

1. **StatefulSet & PVC:** Đảm bảo tính danh tính và dữ liệu bền vững cho Prometheus/Alertmanager.
2. **Alertmanager Clustering:** Sử dụng giao thức **Gossip** để đồng bộ trạng thái "Silence" giữa các node trong cụm.
3. **External DB for Grafana:** Sử dụng **PostgreSQL HA** thay vì SQLite để Grafana có thể scale ngang mà không mất session hay dashboard.
4. **Anti-Affinity:** Ép buộc các Pod HA phải nằm trên các Node vật lý khác nhau.

---

## ⚙️ 3. Chi tiết thành phần (Component Breakdown)

### 3.1. Prometheus - "Não bộ" của hệ thống

- **Cơ chế:** Active Pull (15s interval).
- **Storage:** TSDB (Time Series Database) tối ưu cho dữ liệu chuỗi thời gian.
- **HA Logic:** Chạy 2 bản sao song song, scraping cùng một target để đảm bảo không mất dữ liệu khi 1 pod restart.

### 3.2. Alertmanager - Quản lý cảnh báo

- **Deduplication:** Loại bỏ các cảnh báo trùng lặp từ nhiều thực thể Prometheus.
- **Grouping:** Nhóm các lỗi liên quan (ví dụ: 10 pods chết cùng lúc sẽ chỉ gửi 1 thông báo tổng hợp).
- **Inhibitions:** Ngừng bắn cảnh báo Pod chết nếu Node chứa Pod đó cũng đang chết.

### 3.3. Grafana - Trực quan hóa

- **Version:** 13.0 (Latest).
- **Provisioning:** Dashboards và DataSources được quản lý bằng Code (GitOps ready).
- **HA:** Kết nối PostgreSQL bên ngoài để đảm bảo tính sẵn sàng.

---

## 📈 4. SRE Golden Signals (Chỉ số vàng)

Hệ thống được cấu hình sẵn các **Recording Rules** để theo dõi:

1. **Latency:** Độ trễ của API Server và các ứng dụng.
2. **Traffic:** Lưu lượng mạng qua Cilium/Hubble.
3. **Errors:** Tỷ lệ lỗi HTTP 5xx, Pod CrashLoopBackOff.
4. **Saturation:** CPU Throttling, Disk Full, Memory Pressure.

---

## 🛠️ 5. Hướng dẫn triển khai (Deployment Guide)

Sử dụng **Kustomize** để quản lý các thành phần:

```powershell
# 1. Triển khai các Exporters trước
kubectl apply -k ./node_exporter
kubectl apply -k ./kube_state_metrics

# 2. Triển khai Core Monitoring
kubectl apply -k ./prometheus
kubectl apply -k ./alert_manager

# 3. Triển khai Visualization
kubectl apply -k ./grafana
```

### Kiểm tra trạng thái:

- `kubectl get pods -n monitoring`: Đảm bảo tất cả Pod ở trạng thái `Running`.
- `kubectl get pvc -n monitoring`: Đảm bảo các Volume đã được `Bound`.

---

## 🆘 6. Xử lý sự cố (Troubleshooting)

| Sự cố           | Nguyên nhân                            | Cách xử lý                                      |
| :-------------- | :------------------------------------- | :---------------------------------------------- |
| Pod Pending     | Thiếu tài nguyên hoặc StorageClass lỗi | `kubectl describe pod <name>`                   |
| Metric trống    | Target chưa được discovery             | Kiểm tra `Status -> Targets` trên Prometheus UI |
| Alert không bắn | Sai Token/Config hoặc lỗi Network      | Kiểm tra logs của Alertmanager Pod              |
