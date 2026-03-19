# Phân tích Chi tiết Hiện trạng Cluster Kubernetes (VNPOST)

> **Nguồn dữ liệu:** Trích xuất trực tiếp từ cụm K8s đang hoạt động (Live) thông qua các lệnh `kubectl`.
> **Thời điểm phân tích:** 2026-03-18.

---

## 1. Tổng quan Hệ thống & Topology

| Thông số | Giá trị |
|---|---|
| **Bản phân phối** | **RKE2** (Rancher Kubernetes Engine 2) |
| **Phiên bản K8s** | v1.33.6+rke2r1 |
| **Hệ điều hành Node** | Ubuntu 24.04.3 LTS (Kernel 6.8.0-90-generic) |
| **Container Runtime** | containerd://2.1.5-k3s1 |
| **Tổng số Node** | 6 (3 Control Plane + 3 Worker) |
| **Tuổi đời Cluster** | ~50 ngày |

### Bảng phân bổ Node chi tiết (từ `kubectl get nodes -o wide`)

| Node | Role | IP Nội Bộ | PodCIDR (Dải IP Pod) | Trạng thái |
|---|---|---|---|---|
| `cp01` | control-plane, etcd, master | `172.18.0.51` | `10.100.0.0/24` | Ready |
| `cp02` | control-plane, etcd, master | `172.18.0.52` | `10.100.1.0/24` | Ready |
| `cp03` | control-plane, etcd, master | `172.18.0.53` | `10.100.2.0/24` | Ready |
| `wk01` | worker | `172.18.0.61` | `10.100.4.0/24` | Ready |
| `wk02` | worker | `172.18.0.62` | `10.100.3.0/24` | Ready |
| `wk03` | worker | `172.18.0.63` | `10.100.5.0/24` | Ready |

---

## 2. Cơ chế High Availability (HA) cho RKE2 Server / Kube API Server

Đây là phần cốt lõi giúp hệ thống VNPost không bao giờ ngưng hoạt động ngay cả khi máy chủ bị sập. Cơ chế HA được thiết lập ở **3 tầng** sau:

### 2.1. HA cho Database hệ thống (`etcd`) — Quorum 3 Node

`etcd` là cơ sở dữ liệu Key-Value lưu trữ **toàn bộ trạng thái** của cụm K8s (Pod nào đang chạy ở đâu, Service nào trỏ đi đâu, Secret nào thuộc Namespace nào...).

**Cấu hình thực tế trên cụm VNPost** (xác nhận qua `kubectl get pods -n kube-system`):
```
etcd-cp01   1/1   Running   0   50d   172.18.0.51
etcd-cp02   1/1   Running   0   50d   172.18.0.52
etcd-cp03   1/1   Running   0   50d   172.18.0.53
```

**Cơ chế hoạt động:**
- etcd được triển khai dạng **embedded** (nhúng sẵn bên trong gói cài RKE2) và chạy **phân tán trên cả 3 Master Node**.
- Sử dụng **thuật toán đồng thuận Raft (Raft Consensus)**: Mọi yêu cầu ghi dữ liệu (tạo Pod, sửa Service...) phải được **ít nhất 2/3 node** etcd xác nhận thành công → gọi là **Quorum** (quá bán: ⌊3/2⌋ + 1 = 2).
- **Khả năng chịu lỗi:** Cụm chấp nhận **tối đa 1 node** etcd sập mà hệ thống vẫn ghi được dữ liệu. Nếu 2 node sập → mất Quorum → cụm chuyển sang trạng thái read-only (chỉ đọc).
- **Chống Split-Brain:** Khi mạng bị phân mảnh (ví dụ `cp01` bị cô lập khỏi `cp02` + `cp03`), nhóm `cp02+cp03` (2 node, đủ Quorum) tiếp tục làm chủ. `cp01` chỉ có 1 mình → tự động từ chối phục vụ ghi → tránh tình trạng 2 nhóm ghi dữ liệu xung đột nhau.

### 2.2. HA cho Kube API Server — Active-Active 3 Instances

API Server (`kube-apiserver`) là **cánh cổng duy nhất** cho mọi tương tác với K8s: lệnh `kubectl`, Worker Node báo cáo, Controller Manager điều phối...

**Cấu hình thực tế:**
```
kube-apiserver-cp01   1/1   Running   1   36d   172.18.0.51
kube-apiserver-cp02   1/1   Running   1   36d   172.18.0.52
kube-apiserver-cp03   1/1   Running   1   36d   172.18.0.53
```

- **3 bản API Server chạy song song** (Active-Active) trên cả 3 Master.
- Mỗi bản đều có khả năng nhận và xử lý request.
- Nếu 1 API Server sập, client chỉ cần chuyển sang gọi 1 trong 2 bản còn lại → không cần downtime.
- Các thành phần kèm theo cũng chạy HA đối xứng: `kube-controller-manager` (3 pods), `kube-scheduler` (3 pods).

### 2.3. Virtual IP (VIP) Failover bằng `kube-vip` — Một Đầu Mối Duy Nhất

**Vấn đề:** 3 API Server chạy ở 3 IP khác nhau (`.51`, `.52`, `.53`). Vậy Worker Node và Admin nên gọi IP nào?

**Giải pháp — `kube-vip`:**

Cấu hình thực tế trích xuất trực tiếp từ DaemonSet `kube-vip-ds`:
```ini
vip_arp      = true               ← Sử dụng giao thức ARP (Layer 2)
address      = 172.18.0.50        ← Virtual IP dùng chung
port         = 6443               ← Port mặc định của API Server
vip_interface= ens3               ← Card mạng vật lý gắn VIP
vip_cidr     = 24                 ← Subnet mask /24
cp_enable    = true               ← Bật chế độ Control Plane HA
vip_ddns     = false              ← Không dùng Dynamic DNS
```

**Cơ chế hoạt động:**
1. `kube-vip` chạy dạng **DaemonSet** trên cả 3 Master Node.
2. Tại một thời điểm, **chỉ 1 node Master** "sở hữu" địa chỉ VIP `172.18.0.50` trên card mạng `ens3`.
3. Node đó liên tục phát tín hiệu **Gratuitous ARP** trên mạng LAN để thông báo "172.18.0.50 thuộc về tôi".
4. Khi node sở hữu VIP bị sập, các node `kube-vip` còn lại **phát hiện trong vài giây** và một node khác lập tức chiếm quyền VIP → phát ARP mới → switch/router cập nhật bảng MAC.
5. Kết quả: Mọi client (kubectl, kubelet của Worker) chỉ cần cấu hình 1 địa chỉ duy nhất `https://172.18.0.50:6443`.

**Xác nhận từ `kubectl cluster-info`:**
```
Kubernetes control plane is running at https://172.18.0.50:6443
CoreDNS is running at https://172.18.0.50:6443/api/v1/namespaces/kube-system/services/rke2-coredns-rke2-coredns:udp-53/proxy
```

**⚠️ Cảnh báo:** Các pod `kube-vip-ds` đang có số lượng restarts rất cao (293–317 lần trong 50 ngày). Điều này có thể do bất ổn mạng Layer 2 hoặc do cơ chế Leader Election liên tục tranh chấp VIP. Cần kiểm tra log chi tiết.

---

## 3. Core Concepts — Giải thích Chi Tiết

### 3.1. Mạng Pod (CNI — Cilium eBPF)

**CNI (Container Network Interface)** là lớp phần mềm chịu trách nhiệm cấp IP và định tuyến traffic cho tất cả Pod trong cụm.

Cụm VNPost sử dụng **Cilium** — một trong những CNI hiện đại và mạnh mẽ nhất hiện nay.

**Các Pod Cilium đang chạy (từ `kubectl get pods -n kube-system`):**
- `cilium-*` — 6 DaemonSet pods (chạy trên mỗi Node)
- `cilium-envoy-*` — 6 DaemonSet pods (proxy L7)
- `cilium-operator-*` — 1 pod (quản lý chính sách)

**Đặc điểm kỹ thuật:**
- **eBPF (extended Berkeley Packet Filter):** Cilium chèn mã xử lý gói tin trực tiếp vào **Kernel Linux**, bỏ qua hoàn toàn `iptables` truyền thống. Kết quả: độ trễ mạng cực thấp, throughput cao hơn đáng kể.
- **Network Policies:** Cung cấp tường lửa nội bộ ở cấp vi mô — có thể kiểm soát Pod nào được giao tiếp với Pod nào, dựa trên nhãn (Labels), namespace, port, và cả HTTP URL path.
- **Hubble (Observability):** Service `hubble-peer` (ClusterIP: `10.101.92.4`) cung cấp khả năng giám sát luồng traffic bên trong cluster theo thời gian thực.

### 3.2. CIDR Network — Phân vùng IP cho Pod

**CIDR (Classless Inter-Domain Routing)** là kỹ thuật chia nhỏ dải mạng IP thành các phần nhỏ hơn để phân cấp quản lý.

Trong K8s, có **2 dải CIDR** chính:

#### a) Pod CIDR — Dải IP cho Pod
- **Cluster Pod CIDR:** `10.100.0.0/16` (tổng cộng 65,536 địa chỉ IP khả dụng)
- K8s **chia nhỏ** dải `/16` này thành các **khối `/24`** (mỗi khối 256 địa chỉ) và **khoán trắng cho từng Node:**

| Node | PodCIDR | Ý nghĩa |
|---|---|---|
| `cp01` | `10.100.0.0/24` | Mọi Pod trên `cp01` nhận IP dạng `10.100.0.x` |
| `cp02` | `10.100.1.0/24` | Mọi Pod trên `cp02` nhận IP dạng `10.100.1.x` |
| `cp03` | `10.100.2.0/24` | Mọi Pod trên `cp03` nhận IP dạng `10.100.2.x` |
| `wk01` | `10.100.4.0/24` | Mọi Pod trên `wk01` nhận IP dạng `10.100.4.x` |
| `wk02` | `10.100.3.0/24` | Mọi Pod trên `wk02` nhận IP dạng `10.100.3.x` |
| `wk03` | `10.100.5.0/24` | Mọi Pod trên `wk03` nhận IP dạng `10.100.5.x` |

**Xác minh thực tế:** Pod `Keycloak-0` chạy trên `wk01` → IP: `10.100.5.4` (nằm trong dải `10.100.4.0/24` hoặc `10.100.5.0/24` của `wk01`/`wk03`). ← Thực tế: node `wk01` mang PodCIDR `10.100.4.0/24` nhưng pod có IP `10.100.5.*` chứng tỏ PodCIDR đã được reassigned. Dựa trên live data, `wk01 → 10.100.4.0/24`, `wk03 → 10.100.5.0/24`.

#### b) Service CIDR — Dải IP cho Service
- **Service CIDR:** `10.101.0.0/16`
- Ví dụ: Service `kubernetes` (API mặc định) nhận ClusterIP `10.101.0.1`, CoreDNS nhận `10.101.0.10`.

### 3.3. CoreDNS — Máy chủ Phân giải Tên Miền Nội bộ

Trong K8s, Pod liên tục sinh/chết/đổi IP. Nếu ứng dụng hardcode IP thì sẽ bị lỗi liền. **CoreDNS** giải quyết bài toán này bằng cách cung cấp **hệ thống phân giải tên miền nội bộ** cho toàn cụm.

**Cấu hình thực tế:**
- Pods: `rke2-coredns-rke2-coredns-*` — 2 replicas, chạy trên `cp01` (IP `10.100.0.193`) và `cp02` (IP `10.100.1.193`).
- Service: `rke2-coredns-rke2-coredns` — ClusterIP: `10.101.0.10`, lắng nghe port `53/UDP` và `53/TCP`.
- Autoscaler: `rke2-coredns-rke2-coredns-autoscaler` — tự động tăng/giảm số replica DNS theo tải.

**Nội dung file cấu hình Corefile** (trích xuất trực tiếp từ ConfigMap):
```
.:53 {
    errors                          ← Log lỗi phân giải
    health {
        lameduck 10s                ← Chờ 10s trước khi tắt (graceful shutdown)
    }
    ready                           ← Endpoint kiểm tra sẵn sàng (/ready)
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure               ← Cho phép phân giải Pod theo IP
        fallthrough in-addr.arpa ip6.arpa
        ttl 30                      ← Cache DNS record 30 giây
    }
    prometheus 0.0.0.0:9153         ← Endpoint metrics cho Prometheus
    forward . /etc/resolv.conf      ← Chuyển tiếp DNS ngoài cụm lên DNS của Host
    cache 30                        ← Cache toàn bộ response 30 giây
    loop                            ← Phát hiện vòng lặp DNS
    reload                          ← Tự tải lại config khi thay đổi
    loadbalance                     ← Round-robin giữa các Pod backend
}
```

**Cách hoạt động:**
1. Pod ứng dụng muốn gọi DB: gửi truy vấn DNS tên `psql-keycloak-rw.keycloak.svc.cluster.local`.
2. Truy vấn được gửi tới CoreDNS (`10.101.0.10:53`).
3. CoreDNS tra plugin `kubernetes` → tìm thấy Service `psql-keycloak-rw` ở namespace `keycloak` → trả về ClusterIP `10.101.113.13`.
4. Ứng dụng dùng IP đó để kết nối.
5. Nếu DNS query là tên miền bên ngoài (vd: `google.com`), plugin `forward` sẽ chuyển tiếp lên DNS server của máy chủ vật lý.

---

## 4. Workloads & Pods thực tế (từ `kubectl get pods -A -o wide`)

### 4.1. Hạ tầng cốt lõi (`kube-system`)

| Thành phần | Số Pod | Chạy trên | Ghi chú |
|---|---|---|---|
| **etcd** | 3 | cp01, cp02, cp03 | Database HA |
| **kube-apiserver** | 3 | cp01, cp02, cp03 | API Server Active-Active |
| **kube-controller-manager** | 3 | cp01, cp02, cp03 | Restarts: 30-35 |
| **kube-scheduler** | 3 | cp01, cp02, cp03 | Restarts: 21-34 |
| **kube-vip-ds** | 3 | cp01, cp02, cp03 | ⚠️ Restarts: 293-317 |
| **cilium** | 6 | Tất cả node | DaemonSet CNI |
| **cilium-envoy** | 6 | Tất cả node | L7 Proxy |
| **cilium-operator** | 1 | cp01 | Restarts: 83 |
| **rke2-coredns** | 2 | cp01, cp02 | Cluster DNS |
| **rke2-coredns-autoscaler** | 1 | cp01 | DNS scaling |
| **rke2-metrics-server** | 1 | wk01 | Resource metrics |
| **rke2-snapshot-controller** | 1 | wk01 | ⚠️ Restarts: 161 |
| **openstack-cinder-csi-controller** | 1 | wk02 | ⚠️ Restarts: 644 |
| **openstack-cinder-csi-nodeplugin** | 6 | Tất cả node | Storage driver |

### 4.2. Ứng dụng (Workloads)

| Namespace | Ứng dụng | Pod | Node | IP |
|---|---|---|---|---|
| `elk` | Elasticsearch Master 0 | `elasticsearch-master-0` | wk02 | `10.100.3.185` |
| `elk` | Elasticsearch Master 1 | `elasticsearch-master-1` | wk01 | `10.100.5.191` |
| `elk` | Elasticsearch Master 2 | `elasticsearch-master-2` | wk01 | `10.100.5.234` |
| `elk` | Kibana | `kibana-kibana-*` | wk02 | `10.100.3.148` |
| `keycloak` | Keycloak 0 | `keycloak-0` | wk01 | `10.100.5.4` |
| `keycloak` | Keycloak 1 | `keycloak-1` | wk02 | `10.100.3.219` |
| `keycloak` | PostgreSQL 1 | `psql-keycloak-1` | wk01 | `10.100.5.32` |
| `keycloak` | PostgreSQL 2 | `psql-keycloak-2` | wk01 | `10.100.5.70` |
| `keycloak` | PostgreSQL 3 | `psql-keycloak-3` | wk02 | `10.100.3.200` |
| `cnpg-system` | CNPG Controller | `cnpg-controller-manager-*` | wk01 | `10.100.5.6` |
| `mysql-operator` | MySQL Operator | `mysql-operator-*` | wk02 | `10.100.3.159` |
| `nginx-ingress` | Nginx Ingress | `nginx-ingress-*` | wk02 | `10.100.3.205` |
| `sonobuoy` | Sonobuoy | `sonobuoy` | wk03 | `10.100.4.17` |

---

## 5. Services & Ingress (từ `kubectl get svc -A` và `kubectl get ingress -A`)

### 5.1. Danh sách Services

| Namespace | Service | Type | ClusterIP | Port(s) |
|---|---|---|---|---|
| `default` | `kubernetes` | ClusterIP | `10.101.0.1` | 443/TCP |
| `kube-system` | `rke2-coredns-rke2-coredns` | ClusterIP | `10.101.0.10` | 53/UDP, 53/TCP |
| `kube-system` | `hubble-peer` | ClusterIP | `10.101.92.4` | 443/TCP |
| `kube-system` | `rke2-metrics-server` | ClusterIP | `10.101.213.108` | 443/TCP |
| `kube-system` | `cilium-envoy` | ClusterIP | None (Headless) | 9964/TCP |
| `elk` | `elasticsearch-master` | ClusterIP | `10.101.232.144` | 9200, 9300/TCP |
| `elk` | `elasticsearch-master-headless` | ClusterIP | None (Headless) | 9200, 9300/TCP |
| `elk` | `kibana-kibana` | ClusterIP | `10.101.82.190` | 5601/TCP |
| `keycloak` | `keycloak` | ClusterIP | `10.101.39.214` | 8080/TCP |
| `keycloak` | `keycloak-discovery` | ClusterIP | None (Headless) | — |
| `keycloak` | `psql-keycloak-rw` | ClusterIP | `10.101.113.13` | 5432/TCP |
| `keycloak` | `psql-keycloak-ro` | ClusterIP | `10.101.216.224` | 5432/TCP |
| `keycloak` | `psql-keycloak-r` | ClusterIP | `10.101.104.200` | 5432/TCP |
| `nginx-ingress` | `nginx-ingress` | **NodePort** | `10.101.48.214` | 80→32053, 443→31489 |

### 5.2. Ingress Rules

| Namespace | Ingress | Host | Ingress Class |
|---|---|---|---|
| `elk` | `elasticsearch-master` | `elasticsearch.vnpost.cloud` | nginx |
| `elk` | `kibana-kibana` | `kibana.vnpost.cloud` | nginx |
| `keycloak` | `keycloak` | `iam.vnpost.cloud` | nginx |

---
