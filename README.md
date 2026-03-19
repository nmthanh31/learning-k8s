# Learning Kubernetes (VNPOST_K8s)

Chào mừng bạn đến với dự án tìm hiểu và thực hành hệ thống Kubernetes. Đây là nơi lưu trữ các tài liệu lý thuyết, hướng dẫn triển khai và báo cáo phân tích chi tiết về kiến trúc Kubernetes, đặc biệt là trong môi trường RKE/RKE2.

## 📁 Cấu trúc dự án

Dự án được tổ chức thành các phần chính nhằm cung cấp cái nhìn từ tổng quan đến chi tiết về K8s:

### 1. Phân tích Cluster & Traffic
- 📊 **[Báo cáo phân tích Cluster](./docs/cluster_analysis.md)**: Đánh giá chi tiết tình trạng node, tài nguyên và các thành phần cốt lõi của cụm K8s.
- 🚦 **[Phân tích Traffic (North-South & East-West)](./docs/k8s_traffic.md)**: Tìm hiểu luồng di chuyển dữ liệu, các giải pháp Load Balancing và bảo mật mạng (Zero Trust).
- 🛡️ **[Cơ chế High Availability (HA)](./docs/k8s_rke_ha_mechanisms.md)**: Phân tích chuyên sâu về HA trong RKE/RKE2, bao gồm etcd, API Server, kube-vip và CNI (Cilium).

### 2. Các khái niệm cốt lõi (Core Concepts)
Chuỗi bài viết hướng dẫn từ cơ bản đến nâng cao:
- 🏗️ **[Concept 01: Architecture](./docs/k8s_concept_01_architecture.md)**: Tổng quan về kiến trúc Control Plane và Worker Nodes.
- 📦 **[Concept 02: Pods](./docs/k8s_concept_02_pod.md)**: Đơn vị nhỏ nhất trong K8s, cách triển khai và quản lý bằng YAML.
- 🔄 **[Concept 03: ReplicaSet](./docs/k8s_concept_03_ReplicaSet.md)**: Duy trì số lượng bản sao, đảm bảo tính sẵn sàng cao và khả năng mở rộng.

## 🚀 Mục tiêu của dự án

- Tài liệu hóa quá trình tìm hiểu về Kubernetes.
- Xây dựng kho kiến thức chuẩn về triển khai K8s HA cho doanh nghiệp.
- Tìm hiểu cấu hình các thành phần networking (Cilium, CoreDNS) và storage.

---
*Ghi chú: Các tài liệu được viết bằng Tiếng Việt để hỗ trợ quá trình nghiên cứu nội bộ.*
