# Kubernetes Pod: Core Concepts & Architecture

Pod là **đơn vị triển khai nhỏ nhất** trong hạ tầng Kubernetes, đóng vai trò là lớp trừu tượng (abstraction) bao bọc các container.

---

### 1. Bản chất & Mục đích
- **Không phải là Container:** Một Pod có thể chứa một hoặc nhiều container (Sidecar pattern).
- **Chia sẻ tài nguyên:** Các container trong cùng một Pod dùng chung **Network Namespace** (IP, Port) và **Storage Volumes**.
- **Tính tạm thời (Ephemeral):** Pod không tự phục hồi. Việc duy trì số lượng Pod được đảm bảo bởi các **Controllers** (Deployment, ReplicaSet).

### 2. Cấu trúc & Cơ chế vận hành
| Thành phần | Chức năng chính |
| :--- | :--- |
| **App Container** | Chứa logic ứng dụng (Process chính). |
| **Sidecar/Init Container** | Hỗ trợ logging, proxy, cấu hình ban đầu. |
| **Pause Container** | "Container nền" giữ Network Namespace và Cluster IP cho toàn bộ Pod. |
| **Volumes** | Vùng dữ liệu dùng chung giữa các container nội bộ. |

### 3. Quy trình khởi tạo (Execution Flow)
1. **User:** `kubectl apply -f pod.yaml`.
2. **Control Plane:**
    - **API Server:** Tiếp nhận, validate và lưu thông tin vào **etcd**.
    - **Scheduler:** Tính toán và chọn Node tối ưu để chạy Pod.
3. **Worker Node:**
    - **Kubelet:** Phát hiện Pod được gán cho Node, gọi **Container Runtime** (CRI).
    - **Runtime (Containerd/Docker):** Tạo Pause Container -> Thiết lập Network -> Chạy các App Containers.

### 4. Networking nội bộ
- **Single IP:** Pod có một IP duy nhất trong Cluster.
- **Single Port:** Pod có một Port duy nhất trong Cluster.
- **Localhost Communication:** Các container giao tiếp với nhau qua `localhost:[PORT]`, cực nhanh và không cần NAT, ko đi qua Internet

### 5. Xử lý sự cố (Failure Handling)
- **Container Crash:** Kubelet tự động restart container dựa trên `restartPolicy`.
- **Node failure:** Control Plane sẽ lập lịch (re-schedule) tạo Pod mới trên Node khác để đảm bảo tính sẵn sàng cao (HA).

---
> **Key Takeaway:** Pod là đơn vị bao bọc (Wrapper). Cluster IP nằm ở Pod, không nằm ở Container. Hãy luôn dùng **Deployment** để quản lý Pod thay vì tạo thủ công.

# Kubernetes Deployment: Core Concepts & Architecture

Deployment là đối tượng quản lý vòng đời ứng dụng, cho phép bạn cập nhật phiên bản phần mềm mà không gây gián đoạn (Zero Downtime).

### 1. Cơ chế hoạt động
- **Abstraction:** Deployment quản lý **ReplicaSet**, và ReplicaSet quản lý **Pods**.
- **Rolling Update:** Cập nhật từng Pod một để đảm bảo hệ thống luôn sẵn sàng.
- **Rollback:** Dễ dàng quay lại phiên bản cũ nếu phiên bản mới gặp lỗi (`kubectl rollout undo`).
- **Self-healing:** Nếu một Pod hoặc Node chết, Deployment sẽ ra lệnh cho ReplicaSet tạo lại Pod mới.

### 2. Quy trình Cập nhật (Rolling Update)
1. User cập nhật Image trong file YAML.
2. Deployment tạo một **ReplicaSet mới** (V2).
3. ReplicaSet V2 tạo Pod mới.
4. Khi Pod V2 sẵn sàng, Deployment giảm số lượng Pod ở **ReplicaSet cũ** (V1).
5. Quá trình lặp lại cho đến khi toàn bộ Pod là V2.

---

# Kubernetes ReplicaSet: Duy trì trạng thái mong muốn

ReplicaSet đảm bảo số lượng bản sao (Replicas) của Pod luôn đúng với cấu hình.

- **Selector (Bộ chọn):** Đây là "sợi dây" liên kết giữa ReplicaSet và Pod thông qua các **Labels**.
- **Tính linh hoạt:** ReplicaSet có thể quản lý cả các Pod đã tồn tại trước đó nếu chúng có Labels khớp với Selector.
- **Scaling:** Thay đổi số lượng `replicas` trong spec để tăng/giảm quy mô ứng dụng ngay lập tức.

---

# Kubernetes Service: Networking & Load Balancing

Service cung cấp một địa chỉ **IP và DNS ổn định** để các thành phần có thể giao tiếp với nhau, khắc phục tính chất "tạm thời" của Pod.

### 1. Các loại Service chính
| Loại Service | Phạm vi truy cập | Mục đích sử dụng |
| :--- | :--- | :--- |
| **ClusterIP** (Mặc định) | Nội bộ Cluster | Giao tiếp giữa Front-end và Back-end. |
| **NodePort** | Bên ngoài qua IP của Node | Mở cổng trên toàn bộ Worker Nodes (Port 30000-32767). |
| **LoadBalancer** | Bên ngoài qua Cloud IP | Sử dụng bộ cân bằng tải của nhà cung cấp Cloud (AWS, GCP, Azure). |

### 2. Cơ chế Service IP
- **Virtual IP:** Service IP không gắn vào card mạng thật, nó là một IP ảo được quản lý bởi **kube-proxy**.
- **Load Balancing:** Tự động chia tải đến các Pod khỏe mạnh (Healthy Pods) dựa trên Endpoint list.

---

# Kubernetes Architecture: Control Plane & Worker Nodes

Để vận hành các đối tượng trên, Kubernetes chia hệ thống thành hai phần chính:

### 1. Control Plane (Bộ não)
- **API Server:** Cổng giao tiếp duy nhất của hệ thống.
- **etcd:** Cơ sở dữ liệu lưu giữ toàn bộ trạng thái của Cluster.
- **Scheduler:** Quyết định Pod sẽ chạy trên Node nào.
- **Controller Manager:** Duy trì các trạng thái mong muốn (Node, Deployment, v.v.).

### 2. Worker Nodes (Nơi thực thi)
- **Kubelet:** "Đại sứ" trên mỗi Node, đảm bảo các container chạy đúng như chỉ thị.
- **Kube-proxy:** Quản lý quy tắc mạng (Networking rules) cho Services.
- **Container Runtime:** (Containerd/Docker) Trực tiếp chạy các container.

---

# Kubernetes DaemonSet: Đảm bảo phủ sóng toàn cụm
DaemonSet đảm bảo rằng mọi Worker Node (hoặc một số Node chỉ định) giới hạn đều chạy đúng **một bản sao** của một Pod cụ thể.

### 1. Ứng dụng thực tế
- **Logging & Monitoring:** Chạy Fluentd, Logstash, Prometheus Node Exporter, hoặc Datadog agent trên từng Node để thu thập log và metrics.
- **Networking:** Triển khai các thành phần mạng CNI (như Cilium, Calico) tới mọi Node.
- **Storage:** Chạy các daemon lưu trữ như Ceph, GlusterFS trên các Node được chỉ định.

### 2. Sự khác biệt so với Deployment
- **Deployment** điều hướng số lượng replica một cách linh hoạt, các Pod có thể dồn vào vài Node.
- **DaemonSet** gắn chặt mật thiết với Node: Một Node mới gia nhập cluster sẽ ngay lập tức được tự động cài đặt Pod từ DaemonSet.

---

# Kubernetes StatefulSet: Quản lý ứng dụng có trạng thái
Dùng riêng cho các ứng dụng có lưu trữ dữ liệu bền vững (Stateful) và yều cầu đồng bộ chặt chẽ như Databases (MySQL, MongoDB, Cassandra, Kafka, Zookeeper).

### Đặc điểm cốt lõi
- **Định danh duy nhất (Stable Network ID):** Thay vì có tên ngẫu nhiên (như `pod-1a2b3c`), các Pod trong StatefulSet được đánh số thứ tự (`db-0`, `db-1`, `db-2`).
- **Mạng nhận dạng cố định:** Nếu `db-0` chết và tái tạo lại, nó vẫn giữ nguyên danh tính là `db-0` với cùng một định danh mạng (DNS nội bộ), giúp các tiến trình Master-Slave dễ nhận diện.
- **Lưu trữ gắn chặt (Stable Storage):** Mỗi Pod có một Persistent Volume Claim (PVC) gắn liền với nó không bao giờ bị tráo đổi, dữ liệu của `db-0` sẽ luôn là của `db-0`.

---

# Kubernetes Namespace: Phân chia không gian ảo
Namespace là một ranh giới ảo giúp nhóm tài nguyên lại để phân tách và quản lý (Multi-tenancy) ngay trên cùng một Cluster vật lý.

### Đặc điểm cốt lõi
- **Tránh xung đột tên (Isolation):** Bạn có thể có Pod tên `web-server` ở Namespace `dev` và một Pod khác tên y hệt `web-server` ở Namespace `prod`.
- **Giới hạn tài nguyên (ResourceQuota):** Ngăn chặn một Namespace chiếm dụng toàn bộ tài nguyên (CPU, RAM) phần cứng của Cluster.
- **Phân tách phạm vi:** 
  - *Namespaced objects:* Pod, Service, Deployment, Secret, ConfigMap.
  - *Cluster-wide objects:* Node, PersistentVolume, Khai báo phân quyền ClusterRole (Ảnh hưởng lên toàn cục).

---

# Kubernetes ConfigMap & Secret: Quản lý cấu hình & Bảo mật
Một trong mười hai nguyên tắc ứng dụng (12-Factor App) là: Tách rời phần cấu hình ra khỏi mã nguồn và Image. 
- **ConfigMap:** Lưu trữ cấu hình dạng plain-text (Ví dụ: file config Nginx, các tham số port, thông điệp môi trường).
- **Secret:** Lưu trữ dữ liệu nhạy cảm (Password, Token, SSH Keys). Dữ liệu này được mã hóa dưới dạng base64 (Trên production bắt buộc cần mã hóa bổ sung tại tầng etcd - Encryption at rest).
> **Cách hoạt động:** Khi Pod khởi chạy, nó có thể kéo dữ liệu từ ConfigMap/Secret rồi mount vào bên trong Container dưới dạng Biến môi trường (Environment Variables) hoặc thành hẳn một tập tin tĩnh (File).

---

# Kubernetes Ingress: Cổng điều hướng quốc tế (L7 Routing)
Trong khi Service NodePort và LoadBalancer chỉ định tuyến sơ cấp ở Layer 4 (IP và Port), thì **Ingress** định tuyến thông minh ở **Layer 7** (giao thức HTTP/HTTPS).

### Tại sao cần Ingress?
- **Domain Routing (Host-based):** Định tuyến từ `api.domain.com` sang Backend Service và `web.domain.com` sang Frontend Service chỉ với 1 địa chỉ IP chung (tiết kiệm tiền thuê IP public).
- **Path Routing (Path-based):** `domain.com/auth` trỏ vào Service Authentication, `domain.com/cart` trỏ vào Service Giỏ hàng.
- **SSL/TLS Termination:** Gắn chứng chỉ SSL để mã hóa HTTPS một cách tập trung, giúp giảm tải công việc giải mã cho hàng loạt Pod nội bộ ở phía trong.
- **Điều kiện cần:** Cluster bắt buộc cần một phần mềm **Ingress Controller** (như Nginx Ingress Controller, Traefik, HAProxy) chạy ngầm để đọc và thực thi các luật Ingress này.

---

# 🔄 BỨC TRANH TỔNG THỂ: Giao tiếp liên thành phần trong hệ thống
Giờ chúng ta hãy lắp ghép các concept đơn lẻ trên thành một luồng kiến trúc thực tế.

### Luồng 1: Lưu lượng truy cập vào hệ thống (South-North Traffic)
1. **Phân giải:** User truy cập `https://k8s.local/api`. DNS chuyển hướng request về địa chỉ Public IP của LoadBalancer tại Edge.
2. **Ingress:** LoadBalancer dẫn luồng đi vào thông qua cổng của **Ingress Controller**.
3. **Layer 7 Routing:** Ingress đọc HTTP header, thấy đường dẫn là `/api` và dựa trên cấu hình Ingress Rule đã định nghĩa, nó điều phối lưu lượng đến một **Service (ClusterIP)** chuyên xử lý API.
4. **Load Balancing nội bộ:** Service tra cứu `Endpoints` list (danh sách các Pod được bao bọc bởi Deployment có nhãn Label khớp) và chia đều tải cho một trong những **Pod** đang khoẻ mạnh (`Running` & `Ready`).
5. Pod đó sẽ xử lý tác vụ và trả kết quả ngược lại.

### Luồng 2: Sự giao tiếp nội bộ sau hậu trường (East-West Traffic)
1. App Pod (API xử lý giao dịch) cần lưu trữ dữ liệu xuống Database.
2. App Pod **không bao giờ sử dụng IP của DB Pod** vì nó tạm thời. Thay vào đó, API Pod sẽ gọi domain nội bộ mà CoreDNS duy trì cho Service, ví dụ: `mysql-svc.db-namespace.svc.cluster.local`.
3. Yêu cầu từ API Pod được định tuyến đến Service ClusterIP của Database, từ đó dẫn chính xác đến Pod **StatefulSet** (`mysql-0`).
4. Trước đó, Pod `mysql-0` đã khởi động bằng cách lấy Database URL / Password từ **ConfigMap/Secret** và được gán trước một hệ thống **Persistent Volume (PVC)** chuyên biệt, giúp cho dữ liệu DB ghi xuống đĩa cứng (Storage Class) vĩnh viễn không bị xóa.

### Luồng 3: Theo dõi và thu thập (Monitoring Flow)
Trong lúc các Application Pod và Database Pod cặm cụi chạy, **Kubelet** ghi nhận log của mọi Container xuất ra thư mục `/var/log` trên Worker Node. Ngay lập tức, một Pod thuộc **DaemonSet** nằm sẵn trên Node đó (như Fluent-bit/Prometheus-agent) sẽ âm thầm đọc các file Log/Metrics này và đẩy về máy chủ theo dõi tổng tập trung để quản trị viên (DevOps) giám sát. Mọi thứ hoạt động hoàn toàn độc lập, tách biệt vòng đời và tự động hoá.

---
> **Tổng kết toàn cục:** 
> - **Pod** để chạy Container.
> - **Deployment / StatefulSet / DaemonSet** dùng để khởi tạo và quyết định tính chất tồn tại của Pod.
> - **Service** là danh bạ nội bộ, **Ingress** là bốt gác tiếp khách bên ngoài.
> - **Namespace** xây dựng rào cản ngăn cách, **ConfigMap/Secret** truyền linh hồn cấu hình.
> - **Control Plane** (API Server/Etcd) là kiến trúc sư vĩ đại giám sát toàn bộ.
