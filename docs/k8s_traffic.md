# BÁO CÁO PHÂN TÍCH KIẾN TRÚC LƯU LƯỢNG MẠNG (NORTH–SOUTH & EAST–WEST)

## 1. Tổng quan

Trong kiến trúc hạ tầng hiện đại, đặc biệt là với Kubernetes (K8s) và Microservices, việc phân loại lưu lượng mạng (Traffic) thành hai trục North–South (Bắc–Nam) và East–West (Đông–Tây) là yếu tố tiên quyết để thiết kế hệ thống bảo mật, hiệu năng cao và có khả năng mở rộng.

### Sơ đồ luồng lưu lượng (Traffic Flow)



## 2. Chi tiết về North–South Traffic (Lưu lượng Bắc–Nam)

### 2.1. Định nghĩa

Lưu lượng North–South đại diện cho các kết nối đi vào hoặc đi ra khỏi cụm (Cluster). Đây là cửa ngõ giao tiếp giữa người dùng cuối (End-users) hoặc các hệ thống bên ngoài với các dịch vụ bên trong nội bộ.

### 2.2. Luồng di chuyển (Data Flow)

Request: Người dùng → Internet → Cloud Load Balancer → Ingress Controller/API Gateway → Service → Pod.

Response: Pod → Service → Ingress/LB → Internet → Người dùng.

### 2.3. Các thành phần kỹ thuật chủ chốt

API Gateway: Thực hiện Authentication, Rate Limiting, và Request Transformation.

> [!NOTE]
> **Phân biệt Load Balancer (L4 vs L7):**
> - **Layer 4 (TCP/UDP):** Điều hướng dựa trên IP và Port. Hiệu suất cực cao, thường dùng cho External LB.
> - **Layer 7 (HTTP/HTTPS):** Điều hướng dựa trên Content (Domain, Path, Header). Ingress Controller hoạt động ở lớp này.


### 2.4. Trọng tâm quản lý

Bảo mật: Triển khai WAF (Web Application Firewall), SSL/TLS Termination.

Hiệu năng: Cấu hình Auto-scaling cho Ingress để tránh hiện tượng "nghẽn cổ chai" (Bottleneck).

## 3. Chi tiết về East–West Traffic (Lưu lượng Đông–Tây)

### 3.1. Định nghĩa

Lưu lượng East–West là các kết nối diễn ra hoàn toàn bên trong cụm (Internal Cluster). Đây là sự giao tiếp giữa các Microservices với nhau để hoàn tất một nghiệp vụ phức tạp.

### 3.2. Đặc điểm vận hành

Tỷ trọng: Chiếm khoảng 80%–90% tổng lưu lượng trong các hệ thống Microservices hiện đại.

Độ phức tạp: Khó quan sát hơn North-South do số lượng kết nối cực lớn và diễn ra liên tục.

### 3.3. Các thành phần kỹ thuật chủ chốt

Kube-Proxy & CoreDNS: Chịu trách nhiệm Service Discovery (tìm kiếm dịch vụ theo tên).

Service Mesh (Istio, Linkerd): Quản lý traffic thông qua các Sidecar Proxy (Envoy).

Network Policies: Định nghĩa các quy tắc firewall nội bộ giữa các Namespace/Pod.

### 3.4. Trọng tâm quản lý

Độ trễ (Latency): Tối ưu hóa giao tiếp (sử dụng gRPC thay vì REST).

Khả năng quan sát (Observability): Triển khai Distributed Tracing (Jaeger, Zipkin) để theo dõi luồng request qua nhiều service.

Bảo mật nội bộ: Sử dụng mTLS (Mutual TLS) để đảm bảo các service tin tưởng lẫn nhau.

## 4. So sánh và Đối chiếu

| Tiêu chí | North–South (Bắc–Nam) | East–West (Đông–Tây) |
| :--- | :--- | :--- |
| **Hướng đi** | Ngoài cụm ↔ Trong cụm | Nội bộ giữa các Pod/Node |
| **Đối tượng** | Khách hàng, API bên ngoài | Microservices, Databases |
| **Giao thức** | HTTPS, HTTP, WebSockets | gRPC, REST, Message Queues |
| **Công cụ chính** | Ingress, Load Balancer, Gateway | Service Mesh, ClusterIP, DNS |
| **Rủi ro chính** | Tấn công xâm nhập (External Attack) | Tấn công leo thang (Lateral Movement) |

Kết luận: North-South là điều kiện cần để hệ thống hoạt động, nhưng East-West là điều kiện đủ để hệ thống bền vững và an toàn. Việc hiểu rõ cả hai là bước đi quan trọng nhất để làm chủ kiến trúc Cloud Native.
