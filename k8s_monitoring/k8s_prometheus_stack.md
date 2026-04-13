# BÁO CÁO GIẢI PHÁP GIÁM SÁT CỤM K8S (RKE2 & CILIUM)
## I. VẤN ĐỀ VÀ MỤC TIÊU
### 1. Vấn đề
Hiện nay, việc vận hành các ứng dụng trên nền tảng Kubernetes đòi hỏi sự quan sát liên tục để đảm bảo tính ổn định và hiệu suất. Do đặc tính của K8s là các thành phần có tính chất ngắn hạn, các phương pháp giám sát truyền thống không còn hiệu quả trong việc truy vết sự cố và quản lý tài nguyên động.
### 2. Hạ tầng hiện tại
Hệ thống được triển khai trên cụm Kubernetes sử dụng RKE2 kết hợp với CNI Cilium, cấu hình cụm bao gồm:
•	03 Node Control Plane: Đảm bảo tính sẵn sàng cao (High Availability) cho các thành phần điều phối.
•	03 Node Worker: Nơi triển khai các workload và ứng dụng nghiệp vụ.
### 3. Thành phần cần giám sát
Hệ thống giám sát được thiết kế theo mô hình 4 tầng tiêu chuẩn:
•	Tầng hạ tầng (Infrastructure): Sức khỏe của 06 Node (CPU, RAM, Disk, Network).
•	Tầng Kubernetes (Control Plane): Trạng thái API Server, Scheduler, etcd, và Controller Manager trên 03 node master.
•	Tầng đối tượng (K8s Objects): Trạng thái Deployments, Pods, các lỗi khởi tạo (Pending/CrashLoopBackOff).
## II. MÔ HÌNH GIẢI PHÁP
Chúng ta sẽ cài đặt Prometheus, Grafana, Node Exporter, AlertManager vào cụm RKE2.
### 1. Sơ đồ kiến trúc tổng thể
![moniroting-architect](/imgs/monitoring-architect.png)
 
#### 1.1. Giải thích phân bổ kiến trúc
Hệ thống được thiết kế theo các phân vùng logic (namespaces) và vật lý (nodes) rõ ràng:
•	Khối Control Plane (3 Node): Chứa thành phần cốt lõi của RKE2 là API-Server, nơi cung cấp các metric quản lý trạng thái của toàn bộ cụm.
•	Khối Worker Node (3 Node): Nơi chạy các thành phần đích cần giám sát (Targets) bao gồm: Node Exporter (chạy dạng DaemonSet trên mỗi node), kube-state-metrics, Cilium Agent (quản lý mạng/bảo mật), và các Application nghiệp vụ.
•	Khối Namespace - Monitoring: Nơi triển khai tập trung toàn bộ bộ công cụ giám sát (Prometheus Stack), tách biệt hoàn toàn với các ứng dụng nghiệp vụ để đảm bảo an toàn và dễ quản lý tài nguyên.
#### 1.2. Các thành phần chính và Vai trò giám sát
- Prometheus: Nơi xử lý tập trung
    Mục đích: Thu thập và lưu trữ số liệu (metrics), cung cấp ngôn ngữ truy vấn PromQL để phân tích và kích hoạt cảnh báo.
- Alertmanager: Quản lý cảnh báo thông minh
    Mục đích: Tiếp nhận cảnh báo từ Prometheus, thực hiện loại bỏ trùng lặp, nhóm và định tuyến thông báo.
    Sử dụng: Đảm bảo chỉ các cảnh báo có thể hành động được gửi đến đúng nhóm (qua Slack, Email, Telegram), giảm thiểu nhiễu và hỗ trợ tắt tiếng (silence) khi bảo trì.
- Grafana: Trực quan hóa dữ liệu
    Mục đích: Công cụ tạo bảng điều khiển (Dashboard) tương tác, cung cấp thông tin chi tiết theo thời gian thực.
    Sử dụng: Sử dụng các dashboard mẫu từ cộng đồng cho Kubernetes và Cilium để theo dõi sức khỏe cụm một cách trực quan.
- kube-state-metrics: Theo dõi trạng thái đối tượng K8s
    Mục đích: Tạo ra các số liệu về trạng thái của các đối tượng (Deployments, Pods, Nodes) bằng cách truy vấn Kubernetes API.
    Sử dụng: Cung cấp thông tin chi tiết như số lượng bản sao (replicas) khả dụng hoặc trạng thái hiện tại của tài nguyên.
- Node Exporter: Giám sát sức khỏe vật lý của Node
    Mục đích: Thu thập số liệu cấp hệ thống từ mỗi Node (CPU, bộ nhớ, đĩa, mạng).
    Sử dụng: Phát hiện các sự cố phần cứng hoặc cạn kiệt tài nguyên ở cấp độ máy chủ vật lý/ảo hóa.
### 2. Giải thích Workflow (Luồng hoạt động)
![prometheus-flow](/imgs/prometheus-flow.png)
Dựa vào sơ đồ, luồng hoạt động của hệ thống được chia thành 4 bước chính chạy song song:
- Bước 1: Thu thập dữ liệu
Thành phần Retrieval bên trong Prometheus hoạt động theo cơ chế chủ động kéo (Pull metrics). Theo các chu kỳ quét định sẵn (scrape_interval – đang cấu hình mặc định là 15 giây), Prometheus sẽ tự động gọi HTTP GET đến các endpoint (/metrics) của các mục tiêu (targets) để thu thập dữ liệu đo lường thô.
- Bước 2: Lưu trữ và Đánh giá 
Dữ liệu sau khi tiếp nhận sẽ được ghi vào TSDB (Time Series Database – cơ sở dữ liệu chuỗi thời gian nội bộ của Prometheus) nhằm mục đích lưu trữ tối ưu hóa. Cùng lúc đó, thành phần Rule Engine của Prometheus sẽ liên tục đánh giá các điểm dữ liệu mới này dựa trên các bộ quy tắc (Recording Rules và Alerting Rules) đã được định nghĩa từ trước.
- Bước 3: Định tuyến và Cảnh báo 
Nếu dữ liệu đánh giá vi phạm các ngưỡng quy định, Prometheus sẽ đẩy các bản ghi cảnh báo (Push alerts) định kỳ sang Alertmanager. Tại đây, Alertmanager đóng vai trò mấu chốt trong việc loại bỏ cảnh báo trùng lặp (Deduplication), gộp nhóm (Grouping) các cảnh báo liên quan theo logic, và thực hiện việc định tuyến (Routing) để gửi thông báo (Delivery Notifications) ra các kênh giao tiếp như Gmail và Slack cho đội ngũ kĩ thuật theo dõi.
- Bước 4: Truy vấn và Trực quan hóa (Visualization Workflow)
Người quản trị (DevOps) sẽ truy cập vào giao diện web của Grafana. Khi tải các màn hình Dashboard, Grafana sẽ gửi các truy vấn bằng ngôn ngữ PromQL thông qua HTTP API tới Prometheus Server. Prometheus trích xuất dữ liệu lưu trữ từ TSDB và trả kết quả về dưới dạng JSON, giúp Grafana vẽ thành các biểu đồ trực quan, theo thời gian thực. (Ngoài ra, người dùng cũng có thể sử dụng giao diện UI mặc định của Prometheus để truy vấn trực tiếp PromQL phục vụ cho việc debug nhanh).
### V. KẾ HOẠCH TRIỂN KHAI
Hệ thống được triển khai theo lộ trình 5 giai đoạn chính:
1.	Giai đoạn 1: Cài đặt và cấu hình Prometheus Core.
2.	Giai đoạn 2: Triển khai Node Exporter giám sát hạ tầng Node.
3.	Giai đoạn 3: Triển khai Kube-State-Metrics giám sát đối tượng K8s.
4.	Giai đoạn 4: Cài đặt Grafana và thiết lập Dashboard.
5.	Giai đoạn 5: Cài đặt Alertmanager và kết nối thông báo Telegram.
### VI. CHI TIẾT CÁC BƯỚC CÀI ĐẶT VÀ VẬN HÀNH
#### Giai đoạn 1: Cài đặt và cấu hình Prometheus
Prometheus là trung tâm xử lý dữ liệu. Qua cấu hình thực tế, chúng ta có các thông số kỹ thuật sau:

#### 1. Thống số kỹ thuật (Spec)
- Image: `prom/prometheus:v2.51.2`.
- Số lượng bản sao (Replicas): `2` pod (đảm bảo tính sẵn sàng cao).
- Cổng truy cập (Service): NodePort `30090`.
- Chu kỳ thu thập (Scrape Interval): `5s` (rất nhanh, phù hợp cho việc debug sát sao).

#### 2. Cấu hình chi tiết (prometheus.yml)

Dựa trên `prometheus_configmap.yaml`, hệ thống đang thu thập từ 4 nguồn chính:
- Hệ thống Prometheus: `localhost:9090`.
- Node Exporter: Thu thập từ 3 worker node có IP tĩnh: `172.18.0.61`, `172.18.0.62`, `172.18.0.63` tại cổng `9100`.
- Kube-State-Metrics: Kết nối qua Service Discovery của K8s: `kube-state-metrics.monitoring.svc.cluster.local:8080`.
- Cilium: Giám sát mạng qua `cilium-metrics.kube-system.svc.cluster.local:9962`.

#### 3. Cấu hình Cảnh báo (Alerting Rules)
Hiện tại đã thiết lập rule `InstanceDown` trong `alert.rules.yml`:
- Điều kiện: `up == 0` (Target không phản hồi).
- Thời gian chờ: `5s` (Cảnh báo sẽ được kích hoạt ngay lập tức sau 1 chu kỳ lỗi).
- Độ nghiêm trọng: `critical`.

#### 4. Các thành phần chính
- ConfigMap (`prometheus-config`): Chứa file `prometheus.yml` và `alert.rules.yml`.
- Deployment (`prometheus`): Quản lý vòng đời của Pod, mount volume từ ConfigMap vào `/etc/prometheus`.
- Service (`prometheus`): Expose Web UI cổng 9090 ra bên ngoài qua NodePort 30090.

#### 5. Các lỗi thường gặp & Cách Debug
- Lỗi: Pod không chạy hoặc ở trạng thái CrashLoopBackOff.
    - Nguyên nhân: File cấu hình trong ConfigMap sai định dạng YAML (thiếu khoảng trắng, sai tab).
    - Cách Debug: Sử dụng lệnh kubectl logs <pod-name> -n monitoring để xem log lỗi cú pháp cụ thể của Prometheus.
- Lỗi: Không truy cập được Web UI qua NodePort.
    - Nguyên nhân 1: Firewall của Node chưa mở cổng (thường là dải 30000-32767).
    - Nguyên nhân 2: NeuVector Enforcer đang ở chế độ `Protect` và chặn traffic vào namespace `monitoring`.
    - Cách Debug & Khắc phục: 
        1. Kiểm tra bằng `curl localhost:30090` ngay trên Node để xác định Service có hoạt động nội bộ không.
        2. Nếu Service OK nhưng bên ngoài không vào được, kiểm tra NeuVector.
        3. Khắc phục nhanh: Chuyển namespace monitoring sang chế độ `Discover` để NeuVector tự học luồng traffic thay vì chặn:
           `kubectl patch ns monitoring -p '{"metadata":{"annotations":{"neuvector.io/policy-mode":"Discover"}}}'`
#### Giai đoạn 2: Triển khai Node Exporter (Infrastructure Monitoring)
Node Exporter thu thập các chỉ số phần cứng (CPU, RAM, Disk) từ các Node.
#### 1. Các bước thực hiện
- Triển khai DaemonSet để đảm bảo mỗi Node đều có 1 Pod thu thập.
- Sử dụng hostNetwork: true để lấy thông tin mạng trực tiếp từ host.
- Mount /proc và /sys từ host vào container.
#### 2. Cách Research & Giải quyết vấn đề
- Research: Khi cần giám sát thêm các thông số đặc biệt (như GPU, nhiệt độ), hãy tra cứu "Node Exporter collectors" trên GitHub chính thức của Prometheus.
- Lỗi thường gặp: Node Exporter không đọc được dữ liệu disk.
    - Cách xử lý: Kiểm tra quyền hạn (Security Context) và đảm bảo đã mount đúng đường dẫn rootfs của host.
#### Giai đoạn 3: Triển khai Kube-State-Metrics (K8s Objects Monitoring)
KSM không theo dõi hiệu năng mà theo dõi trạng thái các thực thể (Pod có bao nhiêu bản sao, Deployment có đủ không).
#### 1. Các bước thực hiện
- Triển khai RBAC (Role-Based Access Control): Đây là bước quan trọng nhất để KSM có quyền "đọc" dữ liệu từ API Server.
- Triển khai Deployment và Service (cổng 8080).
#### 2. Lỗi và Debug
- Lỗi: Log báo Forbidden khi truy cập API.
    - Cách Debug: Kiểm tra lại ClusterRoleBinding xem đã gắn đúng ServiceAccount vào ClusterRole chưa.
    - Research: Tra cứu tài liệu "Kube-state-metrics RBAC requirements" để cập nhật danh sách các apiGroups mới nhất.
#### Giai đoạn 4: Cài đặt Dashboard với Grafana
Grafana kết nối với Prometheus để vẽ biểu đồ trực quan. Cấu hình triển khai như sau:

#### 1. Thống số kỹ thuật (Spec)
- Image: `grafana/grafana:10.0.0`.
- Thông tin đăng nhập mặc định: User: `admin` / Password: `12345678`.
- Service (NodePort): `32001` (Truy cập qua `<Node-IP>:32001`).

#### 2. Cách thiết lập & Research
- Cách làm: Sau khi đăng nhập, chọn Data Sources -> Thêm Prometheus -> Điền URL là `http://prometheus:9090` (sử dụng tên Service trong namespace).
- Research Dashboard: Thay vì tự vẽ, hãy truy cập Grafana Dashboards và tìm mã ID (ví dụ: `1860` cho Node Exporter, `14711` cho Cilium) để Import trực tiếp.
#### Giai đoạn 5: Cài đặt Alertmanager và Telegram
Giai đoạn cuối cùng là thiết lập phản hồi tự động khi có sự cố qua Telegram.

#### 1. Cấu hình Telegram (alertmanager.yml)
Dựa trên `alertmanager-config.yaml`, hệ thống đã được cấu hình với thông số:
- Receiver Name: `telegram`.
- Bot Token: `8715216441:AAFd_wEAn8reKnSEB3tJ_yc_4lqFskVcHaI`.
- Chat ID: `7524643345`.
- Resolve Timeout: `5m` (Tự động gửi thông báo khi sự cố đã được khắc phục).

#### 2. Các thành phần chính
- ConfigMap (`alertmanager-config`): Chứa file `alertmanager.yml` định nghĩa bot_token và chat_id.
- Deployment (`alertmanager`): Đảm bảo Alertmanager luôn chạy để nhận cảnh báo từ Prometheus.
- Service (`alertmanager`): Phục vụ việc nhận request từ Prometheus tại cổng `9093`.

#### 3. Lỗi thường gặp & Cách Research
- Lỗi: Alertmanager gửi tin nhắn chậm 


