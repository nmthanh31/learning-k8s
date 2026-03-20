# KIến trúc của RKE2 Cluster

## Mục lục
- [Tổng quan](#1)
- [Các thành phần chính Server Node](#2)
- [Các thành phần chính Worker Node](#3)
- [Các thành phần cốt lõi (Core Concepts)](#4)
- [Kiến trúc và cơ chế HA của RKE2](#5)

## <a name="1">1. Tổng quan</a>

### 1.1 RKE1
RKE1 (Rancher Kubernetes Engine v1) là một phiên bản phân phối Kubernetes chạy hoàn toàn trong các Docker Container. Nó yêu cầu Docker làm runtime duy nhất. Quá trình triển khai RKE1 dựa trên việc kết nối SSH vào các node và khởi chạy các container hệ thống để hình thành cluster. 

### 1.2 RKE2
RKE2, còn được gọi là RKE Government, là sự kết hợp giữa tính gọn nhẹ của K3s và tính bảo mật của RKE1. Những thay đổi cốt lõi bao gồm:
- Không phụ thuộc Docker: Sử dụng containerd làm Container Runtime Interface (CRI) mặc định.
- Tuân thủ bảo mật: Được thiết kế để đáp ứng các tiêu chuẩn FIPS 140-2 của chính phủ Mỹ.
- Cơ chế Static Pods: Các thành phần Control Plane không chạy như các container Docker rời rạc mà được quản lý bởi Kubelet dưới dạng các Static Pods, giúp nó gần gũi hơn với kiến trúc Kubernetes thuần (upstream)

![RKE2 Architecture](/imgs/RKE2-architecture.png)

## <a name="2">2. Các thành phần chính Server Node</a>
Server Node đóng vai trò là "bộ não" của hệ thống, quản lý trạng thái và điều phối tài nguyên.

- **RKE Supervisor**: Thành phần quản lý cấp cao nhất, chịu trách nhiệm khởi tạo node, quản lý chứng chỉ (certificates) và thiết lập giao tiếp ban đầu giữa các thành phần.

- **Kubelet**: Đại lý quản lý các Pod trên node. Trong RKE2, Kubelet chịu trách nhiệm đọc các file manifest trong thư mục chỉ định để chạy các thành phần Control Plane dưới dạng Static Pods.

- **Hệ thống Static Pods (Control Plane)**:

    - **etcd**: Cơ sở dữ liệu phân tán lưu trữ toàn bộ cấu hình và trạng thái của cluster.

    - **api-server**: Cổng giao tiếp chính. Mọi thành phần khác (kubectl, scheduler, agent) đều tương tác qua đây.

    - **controller-manager**: Giám sát trạng thái cụm và thực hiện các điều chỉnh để đưa trạng thái thực tế về trạng thái mong muốn.

    - **scheduler**: Lập lịch triển khai các Pod lên các Agent Node dựa trên tài nguyên sẵn có.

    - **cloud-controller-manager**: Thành phần tùy chọn để tích hợp với các API của nhà cung cấp hạ tầng đám mây.

## <a name="3">3. Các thành phần chính Worker Node</a>
Worker Node là nơi trực tiếp vận hành các ứng dụng của người dùng.

Managed Processes:

- **Kubelet**: Nhận chỉ thị từ API-server để quản lý vòng đời container trên node.

- **CRI (containerd)**: Runtime thực hiện việc kéo image và chạy container.

- **RKE2 K8s Deployments (System Add-ons)**: RKE2 tự động triển khai các thành phần này thông qua Helm Chart:

    - **kube-proxy**: Duy trì các quy tắc mạng (iptables/IPVS) để cho phép giao tiếp giữa các Pod.

    - **CNI (Canal)**: Kết hợp giữa Flannel (cho mạng overlay) và Calico (cho chính sách bảo mật mạng - Network Policies).

    - **CoreDNS**: Dịch vụ DNS nội bộ cho cluster.

    - **Ingress (Nginx)**: Điều hướng lưu lượng từ bên ngoài vào các Service bên trong.

    - **User-defined workloads**: Các ứng dụng của người dùng, Service Mesh (như Istio), và các ứng dụng bên thứ ba khác.

## <a name="4">4. Các thành phần cốt lõi (Core Concepts)</a>

RKE2 được thiết kế theo triết lý "Batteries Included", nghĩa là nó đóng gói sẵn các thành phần cần thiết nhất nhưng vẫn cho phép thay thế linh hoạt.

### 4.1. CNI (Container Network Interface) - Canal

RKE2 hỗ trợ nhiều tùy chọn CNI khác nhau tùy thuộc vào nhu cầu về hiệu năng và bảo mật của hệ thống:

- **Canal (Mặc định)**: Là giải pháp kết hợp giữa Flannel (quản lý mạng overlay VXLAN đơn giản) và Calico (cung cấp Network Policies mạnh mẽ). Canal mang lại sự cân bằng hoàn hảo cho hầu hết các trường hợp sử dụng phổ thông, đảm bảo các Pod có thể liên lạc xuyên suốt các node trong khi vẫn có thể áp dụng các quy tắc tường lửa lớp 3/4.

- **Cilium**: Là một CNI hiện đại dựa trên công nghệ eBPF (Extended Berkeley Packet Filter). Cilium được ưa chuộng trong các môi trường yêu cầu hiệu suất cực cao và khả năng quan sát (observability) chi tiết.

Hiệu năng: Thay thế iptables truyền thống bằng eBPF giúp giảm độ trễ và tăng tốc độ xử lý gói tin.

Bảo mật lớp 7: Khác với Canal, Cilium có thể thực thi các Network Policy ở cả lớp ứng dụng (HTTP/gRPC/Kafka).

Tính năng nâng cao: Hỗ trợ Service Mesh tích hợp (không cần sidecar), Transparent Encryption (IPsec/WireGuard) và quan sát mạng qua dự án Hubble.

Các tùy chọn khác: RKE2 cũng hỗ trợ Calico thuần túy hoặc Multus cho các trường hợp yêu cầu nhiều giao diện mạng trên một Pod.

### 4.2. CoreDNS

CoreDNS là hệ thống phân giải tên miền (DNS) mặc định trong Kubernetes.

Chức năng: Tự động đăng ký tên miền cho các Service. Khi một Service mới được tạo ra, CoreDNS sẽ gán cho nó một bản ghi (ví dụ: my-svc.my-namespace.svc.cluster.local).

Hoạt động trong RKE2: CoreDNS chạy dưới dạng một Deployment trong namespace kube-system. Nó giúp các ứng dụng tìm thấy nhau thông qua tên gọi thay vì phải quản lý địa chỉ IP biến động của Pod.

### 4.3. Ingress Controller (Nginx)

RKE2 cài đặt sẵn Nginx Ingress Controller để quản lý lưu lượng truy cập từ bên ngoài vào cluster.

Vai trò: Hoạt động như một Reverse Proxy và Load Balancer ở lớp 7 (HTTP/HTTPS). Nó cho phép bạn cấu hình các quy tắc định tuyến (Routing rules) dựa trên Hostname hoặc Path (đường dẫn URL).

Đặc điểm: Hỗ trợ sẵn các tính năng như SSL Termination, Virtual Hosting và Path-based routing.

### 4.4. Helm Controller

Đây là một điểm đặc biệt kế thừa từ K3s. RKE2 sử dụng Helm Controller để quản lý các thành phần hệ thống.

Cơ chế: Các thành phần như CNI, DNS, Ingress được định nghĩa dưới dạng các bản ghi Custom Resource Definition (CRD) gọi là HelmChart.

Lợi ích: Giúp việc nâng cấp hoặc thay đổi cấu hình các thành phần hệ thống trở nên nhất quán và dễ dàng thông qua việc chỉnh sửa file cấu hình YAML thay vì phải chạy các lệnh Helm thủ công.

## <a name="5">5. Kiến trúc và cơ chế HA của RKE2</a>
Để đảm bảo tính sẵn sàng cao và loại bỏ điểm yếu chí tử (Single Point of Failure), RKE2 triển khai mô hình Multi-Server nhằm bảo vệ API Server và dữ liệu etcd.

### 5.1. Kiến trúc HA cơ bản

Một cluster HA chuẩn thường bao gồm:

- **3 hoặc 5 Server Nodes (Control Plane)**: Đảm bảo số lẻ để duy trì Quorum cho etcd.

- **N Agent Nodes (Worker)**: Chạy khối lượng công việc thực tế.

- **1 Endpoint chung (LB hoặc VIP)**: Điểm truy cập duy nhất cho toàn bộ cluster.

Dưới đây là phiên bản **đầy đủ về HA của RKE2**, bao gồm cả **Load Balancer (LB)** và **Virtual IP (VIP)**, kèm gợi ý công cụ và khi nên dùng:

---

## 5.2. Cơ chế HA bằng Load Balancer (LB)

### 5.2.1. Nguyên lý

Load Balancer (LB) là lớp trung gian trước các node Control Plane. Mọi yêu cầu từ Agent hoặc quản trị viên đều đi qua LB, giúp phân phối tải và đảm bảo endpoint ổn định:

* **Thuật toán phân tải:** Round-robin để tối ưu tài nguyên của API Server.
* **Health Check:** Kiểm tra định kỳ trạng thái cổng 6443. Node lỗi sẽ được loại khỏi pool cho đến khi phục hồi.

LB bảo đảm rằng client luôn có endpoint ổn định, ngay cả khi một node control plane gặp sự cố.

---

### 5.2.2. Kiến trúc

```text
       LB (IP: 10.0.0.100 hoặc DNS: api.example.com)
                |
     -------------------------------------
     |                  |                |
   master1           master2           master3
      \                 |                 /
                  etcd cluster
```

* LB có thể là phần mềm (HAProxy, Nginx) hoặc dịch vụ cloud (AWS ELB, Azure ALB).
* LB cần HA (ví dụ Active/Backup với Keepalived) để tránh SPOF.


---

## 5.3. Cơ chế HA bằng Virtual IP (VIP)

### 5.3.1. Nguyên lý

VIP là IP ảo gán cho node Active trong nhóm control plane. Các node Backup giám sát bằng VRRP. Khi node Active lỗi, VIP tự động chuyển sang Backup, giữ endpoint không thay đổi.

---

### 5.3.2. Kiến trúc

```text
        VIP (IP: 10.0.0.100)
                |
           Active master
           /      |      \
     Backup nodes standby
```

---

### 5.3.3. Công cụ triển khai

* **Kube-VIP:** Lightweight, chạy dạng pod/container, tích hợp sẵn RKE2, hỗ trợ cả API Server và LoadBalancer Service.
* **Keepalived + VRRP:** Truyền thống, chạy trên host, phù hợp lab/on-prem nhỏ.

---

