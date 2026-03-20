# 🚀 VNPOST Kubernetes (K8s) Learning Journey

Chào mừng đến với kho tài liệu nghiên cứu và triển khai **Kubernetes** tại **VNPost**. Đây là nơi hệ thống hóa toàn bộ kiến thức từ cơ bản đến nâng cao, tập trung vào kiến trúc thực tế, bảo mật mạng và khả năng sẵn sàng cao (High Availability).

---

## 🏗️ Kiến trúc & Phân tích Chuyên sâu (Advanced Analysis)

Tài liệu chi tiết về cách thức vận hành của hệ thống trong môi trường Production (RKE/RKE2).

- 🏢 **[Kiến trúc RKE/RKE2 HA](./docs/k8s_rke_architecture.md)**: Phân tích cơ chế High Availability, Etcd quorum, và thành phần Control Plane.
- 🚦 **[Phân tích Traffic (South-North & East-West)](./docs/k8s_traffic.md)**: Mô phỏng luồng dữ liệu từ User đến Pod, ứng dụng công nghệ **Cilium/eBPF** để tối ưu hiệu năng.
- 🛡️ **[Báo cáo Phân tích Cluster](./docs/cluster_analysis.md)**: Đánh giá trạng thái thực tế của các Node, tài nguyên (CPU/RAM) và các thành phần cốt lõi của cụm.

---

## 📚 Khái niệm Cốt lõi (Core Concepts)

Chuỗi bài học hướng dẫn từng bước để làm chủ các thành phần của Kubernetes:

1.  **[Concept 01: Architecture](./docs/k8s_basic/k8s_concept_01_architecture.md)** — Hiểu về Control Plane và Worker Nodes.
2.  **[Concept 02: Pods](./docs/k8s_basic/k8s_concept_02_pod.md)** — Đơn vị thực thi nhỏ nhất trong K8s.
3.  **[Concept 03: ReplicaSet](./docs/k8s_basic/k8s_concept_03_ReplicaSet.md)** — Đảm bảo tính sẵn sàng và tự chữa lành.
4.  **[Concept 04: Deployment](./docs/k8s_basic/k8s_concept_04_deployment.md)** — Quản lý vòng đời ứng dụng và Rolling Update.
5.  **[Concept 05: Service](./docs/k8s_basic/k8s_concept_05_service.md)** — Giải pháp Networking và Load Balancing nội bộ.

---
*Ghi chú: Mọi tài liệu được biên soạn bằng Tiếng Việt nhằm hỗ trợ tối đa cho quá trình nghiên cứu và đào tạo nội bộ.*
