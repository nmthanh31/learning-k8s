# Tìm hiểu cơ chế High Availability (HA) trong RKE / RKE2

Tài liệu này trình bày kiến thức tổng quan về các cơ chế HA được tích hợp sẵn trong bản phân phối RKE và RKE2 của Rancher/SUSE, không gắn với một cụm cụ thể nào.

## 1. RKE / RKE2 là gì?

### 1.1. Kubernetes gốc (Vanilla K8s)

Bản phân phối gốc của Google chỉ bao gồm các thành phần lõi (kube-apiserver, etcd, kubelet...). Việc cài đặt và cấu hình High Availability (HA) thường yêu cầu quy trình thủ công phức tạp (như "Kubernetes the Hard Way").

### 1.2. RKE (Rancher Kubernetes Engine - v1)

Cơ chế: Đóng gói các thành phần K8s vào Docker containers.

Ưu điểm: Đơn giản hóa cài đặt qua file cluster.yml.

Hạn chế: Phụ thuộc hoàn toàn vào Docker daemon. Nếu Docker gặp sự cố, toàn bộ cluster bị ảnh hưởng.

### 1.3. RKE2 (RKE Government - v2)

Ra đời nhằm tối ưu bảo mật và hiệu năng:

- Runtime: Sử dụng containerd thay cho Docker (tuân thủ CRI).

- Security: "Hardened by default" theo tiêu chuẩn CIS Benchmarks.

- Embedded: Tích hợp sẵn etcd, CNI (Cilium/Canal), CoreDNS và Ingress Controller.

- Chứng chỉ: Hỗ trợ môi trường FIPS 140-2.

## 2. Kiến trúc HA trong RKE2

Một cụm RKE2 đạt High Availability (HA) không chỉ dựa vào số lượng node, mà phải đảm bảo đồng thời 3 cơ chế cốt lõi:

- Multi-server (Control Plane HA)

- Datastore HA (etcd quorum)

- Fixed Registration Endpoint (Load Balancer hoặc VIP)

Thiếu một trong ba yếu tố trên → hệ thống không đạt HA thực sự.

### 2.1. Multi-Server (Control Plane HA)

RKE2 triển khai HA bằng cách chạy nhiều node ở chế độ server.

Đặc điểm:

- Mỗi server node chạy đầy đủ:

  - kube-apiserver

  - kube-controller-manager

  - kube-scheduler

  - embedded etcd

Yêu cầu:

- Tối thiểu: 3 server nodes

- Các node dùng chung cluster token để join

Cơ chế HA:

- Các server hoạt động song song (active-active)

- Khi 1 node bị lỗi:
  - Các node còn lại vẫn phục vụ request

- Không gián đoạn control plane

### 2.2. Fixed Registration Endpoint (Load Balancer / VIP)

Đây là thành phần bắt buộc để HA hoạt động đúng.

Vai trò:

- Cung cấp một endpoint duy nhất cho:
  - Agent nodes kết nối vào cluster

  - Server nodes join cluster

  - Client (kubectl, CI/CD) truy cập API

Triển khai:

- Load Balancer (HAProxy, NGINX, Cloud LB)

- hoặc Virtual IP (keepalived)

Nguyên tắc:

- Tất cả node phải dùng chung endpoint:
  - https://<LB_IP>

Cơ chế HA:

- Load Balancer phân phối traffic đến nhiều server nodes

- Khi 1 server node bị lỗi:
  - Traffic tự động chuyển sang node khác

  - Node mới vẫn join cluster bình thường

Lưu ý quan trọng:

- Không sử dụng IP riêng lẻ của từng node

- Không có LB/VIP → không có HA thực sự

### 2.3. Datastore HA (etcd Cluster)

RKE2 sử dụng embedded etcd làm datastore mặc định.

Vai trò:

- Lưu toàn bộ trạng thái cluster (state)

Cơ chế HA:

- Sử dụng thuật toán Raft consensus

- Hoạt động theo nguyên tắc quorum (n/2 + 1)

Cấu hình chuẩn:

- 3 nodes → quorum = 2

- 5 nodes → quorum = 3

Failure handling:

- Mất 1 node → cluster vẫn hoạt động

- Mất quorum → cluster ngừng hoạt động hoàn toàn

Yêu cầu:

- Số node etcd phải là số lẻ

- Disk I/O nhanh (ưu tiên SSD)

### 2.4. Phân tách luồng giao tiếp (Ports)

RKE2 sử dụng 2 cổng chính phục vụ HA, mỗi cổng đảm nhiệm một vai trò riêng.

#### a. Kube API Server – Port 6443

Chức năng:

- Nhận request điều khiển từ:
  - kubectl

  - CI/CD

  - Dashboard

Trong HA:

- Traffic đi qua Load Balancer

- Được phân phối đến nhiều server nodes

- Health check:
  - /healthz

#### b. RKE2 Supervisor – Port 9345

Chức năng:

- Dùng cho giao tiếp nội bộ cluster:
  - Node join cluster

  - Phân phối certificate

  - Bootstrap control plane và etcd

Trong HA:

- Node mới kết nối qua endpoint LB

- Nếu 1 server lỗi → join qua server khác

Lưu ý:

- Không expose public như API Server

- Sai cấu hình port này → node không join được

## 3. Quy trình triển khai cụm RKE2 HA

Để thiết lập cụm HA, cần thực hiện theo các bước chiến lược sau:

### 3.1. Cấu hình Fixed Registration Address (Điểm đăng ký cố định)

Các server node (từ node thứ 2 trở đi) và tất cả agent node đều cần một URL cố định để đăng ký. URL này nên nằm sau một Endpoint ổn định để tránh việc thay đổi IP/Hostname của các node theo thời gian.

Các hướng dẫn tiếp cận phổ biến:

- Layer 4 (TCP) Load Balancer: Điều hướng port 9345 (Supervisor) và 6443 (API Server).

- Round-robin DNS: Trỏ một hostname tới danh sách IP của các Server nodes.

- Virtual/Elastic IP (VIP): Sử dụng các giải pháp như Keepalived hoặc kube-vip.

### 3.2. Khởi tạo Server Node đầu tiên (First Server Node)

Node đầu tiên sẽ thiết lập Secret Token dùng chung. Nếu không chỉ định, RKE2 sẽ tự tạo tại /var/lib/rancher/rke2/server/node-token.

Lưu ý quan trọng về TLS: Để tránh lỗi chứng chỉ khi truy cập qua Load Balancer, cần sử dụng tham số tls-san để thêm IP hoặc Hostname của Load Balancer vào danh sách Subject Alternative Name của chứng chỉ TLS.

Ví dụ file /etc/rancher/rke2/config.yaml:

```bash
token: my-shared-secret
tls-san:
  - my-kubernetes-domain.com
  - another-kubernetes-domain.com
# Tùy chọn: Taint để làm Control Plane chuyên dụng
node-taint:
  - "CriticalAddonsOnly=true:NoExecute"
```

### 3.3. Gia nhập các Server Node bổ sung

Các node tiếp theo cần trỏ về node đầu tiên (hoặc qua Fixed Address) và sử dụng cùng token.

Quy tắc khớp cấu hình (Matching Configuration): Các tham số quan trọng như cluster-cidr phải giống hệt nhau trên tất cả các server node. Nếu sai lệch, node sẽ không thể join cụm và báo lỗi critical configuration value mismatch.

Ví dụ config cho các server node sau:

```bash
server: [https://my-kubernetes-domain.com:9345](https://my-kubernetes-domain.com:9345)
token: my-shared-secret
tls-san:
  - my-kubernetes-domain.com
```

## 4. Tầng HA thứ nhất: etcd — Database Phân tán

### 4.1. etcd là gì?

etcd lưu trữ toàn bộ trạng thái của cluster (Pod, Service, ConfigMap, Secret, RBAC...).

### 4.2. Thuật toán Raft Consensus

etcd sử dụng thuật toán Raft để duy trì tính nhất quán. Một thay đổi chỉ được chấp nhận nếu có sự đồng thuận từ đa số các node (Quorum).

Công thức Quorum: n/2 + 1.

3 node: Chịu lỗi được 1 node.

5 node: Chịu lỗi được 2 node.
Lưu ý: Luôn dùng số lượng node lẻ (thường là 3) để đảm bảo khả năng biểu quyết.

### 4.3. RKE vs RKE2: Cách triển khai etcd

| Đặc tính   | RKE1                        | RKE2                          |
| ---------- | --------------------------- | ----------------------------- |
| Triển khai | Docker container            | Static Pod (embedded)         |
| Backup     | `rke etcd snapshot-save`    | `rke2 etcd-snapshot save`     |
| Khôi phục  | `rke etcd snapshot-restore` | `rke2 server --cluster-reset` |

## 5. Tầng HA thứ hai: Controller & Scheduler

Khác với API Server (Active-Active), kube-controller-manager và kube-scheduler chạy theo mô hình Active-Standby:

Tại một thời điểm, chỉ 1 instance là Leader.

Cơ chế Lease: Leader giữ một "hợp đồng thuê" trong etcd. Nếu Leader sập và không gia hạn Lease -> Instance Standby sẽ giành quyền và trở thành Leader mới.

## 6. Core Concepts: Hạ tầng mạng và Điều phối

### 6.1. CIDR Network (Phân vùng IP)

Dùng để làm gì: Xác định dải IP nội bộ cho Pod và Service, tránh xung đột với mạng vật lý.

Các thành phần:

- Cluster CIDR (Pod Network): Mặc định 10.42.0.0/16. Cấp IP cho từng Pod.
- Service CIDR (Cluster IP): Mặc định 10.43.0.0/16. IP ảo ổn định cho Service.

Cách dùng: RKE2 chia Cluster CIDR thành các khối /24 (256 IP) cho mỗi Node. Khi Pod sinh ra trên Node, CNI tự cấp IP từ khối đó mà không cần hỏi các Node khác.

### 6.2. CNI - Cilium (Mạng Overlay & eBPF)

Dùng để làm gì: Kết nối Pod trên các Node khác nhau. RKE2 mặc định dùng Cilium.

Công nghệ eBPF: Thay thế iptables truyền thống, cho phép xử lý gói tin trực tiếp trong Linux Kernel.

Ưu điểm HA: Tự động cập nhật bảng định tuyến cực nhanh khi Pod di chuyển hoặc Node lỗi.

### 6.3. CoreDNS (Phân giải tên miền)

Dùng để làm gì: Phân giải Service Name (ví dụ: my-db.namespace.svc.cluster.local) thành IP.

Cơ chế HA:

- Chạy tối thiểu 2 replica (mặc định).
- Autoscaler: RKE2 tích hợp coredns-autoscaler, tự động tăng số lượng Pod CoreDNS khi cluster mở rộng (thêm Node/CPU).

### 6.4. Ingress Controller (Nginx)

Dùng để làm gì: Cổng đón traffic từ Internet/Intranet vào cụm dựa trên Domain/Path.

Cơ chế HA: Chạy dưới dạng DaemonSet (mỗi node 1 pod) hoặc Deployment có nhiều replica, thường kết hợp với External LB để có IP tĩnh.

7. Tổng kết mô hình HA chuẩn

| Tầng     | Thành phần         | Cơ chế HA               | Chịu lỗi (3 nodes)                         |
| -------- | ------------------ | ----------------------- | ------------------------------------------ |
| Database | etcd               | Raft Consensus / Quorum | 1 node sập                                 |
| API      | kube-apiserver     | Active-Active           | Phụ thuộc etcd (mất quorum → toàn bộ chết) |
| Control  | controller-manager | Leader Election (Lease) | 2 node sập (nếu etcd còn quorum)           |
| Control  | kube-scheduler     | Leader Election (Lease) | 2 node sập (nếu etcd còn quorum)           |
| DNS      | CoreDNS            | Replicas + Autoscaling  | Phụ thuộc số replica                       |
| Network  | CNI (Cilium)       | Distributed (eBPF)      | Không phụ thuộc control-plane              |

### 8. Kịch bản khôi phục khi lỗi (Failover)

Nếu 1 Server sập: Load Balancer nhận biết lỗi health check -> Chuyển hướng traffic API/Supervisor sang các node còn lại.

Nếu Pod DNS sập: Kubernetes Scheduler tự động tạo Pod mới. Các replica còn lại vẫn duy trì phân giải tên miền.

Agent Node mất kết nối: rke2 agent đóng vai trò là client-side load balancer, sẽ tự động retry kết nối tới các server node thông qua địa chỉ đăng ký cố định. Workload (Pod) đang chạy vẫn tiếp tục hoạt động bình thường.
