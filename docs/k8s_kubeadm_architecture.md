## Kiến trúc của Kubeadm Cluster

---

### Mục lục

1. [Tổng quan về Kubeadm](#1)
2. [Các thành phần lớp Control Plane (Chi tiết)](#2)
3. [Các thành phần lớp Worker Node (Chi tiết)](#3)
4. [Quy trình khởi tạo chuyên sâu (Bootstrapping Phases)](#4)
5. [Cơ chế Quản lý Chứng chỉ (Certificate Management)](#5)
6. [So sánh chi tiết Kubeadm và RKE2](#6)

---

### <a name="1">1. Tổng quan về Kubeadm</a>

Kubeadm là công cụ tiêu chuẩn (standardized tool) được duy trì bởi nhóm SIG Cluster Lifecycle của Kubernetes. Mục tiêu của nó là cung cấp một con đường "best-practice" để khởi tạo một cụm Kubernetes mà không làm mất đi tính linh hoạt của người dùng.

Khác với các giải pháp "Managed" hoặc "All-in-one" như RKE2 hay K3s, Kubeadm đóng vai trò như một bộ khung (framework). Nó đảm bảo các thành phần cốt lõi được cấu hình đúng chuẩn bảo mật, đồng thời để ngỏ các lớp mạng (Networking) và lưu trữ (Storage) cho các giải pháp bên thứ ba.

**Triết lý thiết kế:**

* **Tính tối giản (Simplicity):** Tập trung vào việc đưa cluster về trạng thái Ready với số lượng câu lệnh ít nhất.
* **Tính tuân thủ (Conformity):** Đảm bảo mọi cụm được tạo ra đều vượt qua các bài kiểm tra chuẩn của CNCF.
* **Hỗ trợ vòng đời (Lifecycle):** Cung cấp các công cụ để nâng cấp phiên bản (upgrade), xoay vòng chứng chỉ (cert rotation) và mở rộng cụm.

---

### <a name="2">2. Các thành phần lớp Control Plane (Chi tiết)</a>

Trong mô hình Kubeadm, hầu hết các thành phần Control Plane chạy dưới dạng Static Pods. Kubelet sẽ quan sát thư mục `/etc/kubernetes/manifests` và tự động khởi chạy các Pod này.

#### 2.1 kube-apiserver

* **Vai trò:** Trái tim của cluster, là thành phần duy nhất tương tác trực tiếp với etcd.
* **Cơ chế:** Mọi thao tác thay đổi trạng thái đều được thực hiện qua REST API. Nó thực hiện xác thực (Authentication), phân quyền (Authorization) và kiểm soát nhập nạp (Admission Control).
* **Kết nối:** Thường lắng nghe trên cổng 6443.

#### 2.2 etcd

* **Vai trò:** Cơ sở dữ liệu key-value, lưu giữ "nguồn sự thật" (Source of Truth) của toàn bộ hệ thống.
* **Cấu hình:** Kubeadm mặc định thiết lập etcd đơn lẻ trên master node đầu tiên. Trong mô hình HA, etcd được cấu hình thành một cluster (thường là 3 hoặc 5 node) sử dụng thuật toán đồng thuận Raft.
* **Lưu ý:** etcd cực kỳ nhạy cảm với hiệu năng I/O của đĩa cứng.

#### 2.3 kube-scheduler

* **Vai trò:** Bộ não lập lịch. Nó theo dõi các Pod chưa gán Node và quyết định vị trí dựa trên tài nguyên (CPU/RAM), các ràng buộc (node affinity, taints/tolerations) và chính sách ưu tiên.

#### 2.4 kube-controller-manager

* **Vai trò:** Tập hợp các vòng lặp điều khiển (control loops).
* **Thành phần:** Bao gồm Node Controller (quản lý trạng thái node), Job Controller (quản lý tác vụ chạy một lần), và EndpointSlice Controller (kết nối Pod với Service).

---

### <a name="3">3. Các thành phần lớp Worker Node (Chi tiết)</a>

Worker Node tập trung vào việc thực thi khối lượng công việc (Workloads).

#### 3.1 Kubelet (Thành phần quan trọng nhất)

* **Hoạt động:** Chạy dưới dạng một dịch vụ hệ thống (systemd service), không phải container.
* **Trách nhiệm:** Giao tiếp với Container Runtime qua giao diện CRI để chạy container, báo cáo trạng thái tài nguyên của node về API Server và thực hiện các bài kiểm tra sức khỏe (Liveness/Readiness probes).

#### 3.2 Kube-proxy

* **Chế độ hoạt động:** Thường chạy ở chế độ iptables (mặc định) hoặc IPVS (hiệu năng cao hơn cho cluster lớn).
* **Chức năng:** Ánh xạ các IP ảo của Service thành IP thực của các Pod trong backend.

#### 3.3 Container Runtime (CRI)

Kubeadm yêu cầu người dùng tự cài đặt CRI. Xu hướng hiện tại là sử dụng containerd vì tính nhẹ nhàng và ổn định. Các runtime khác bao gồm CRI-O hoặc Mirantis Container Runtime.

---

### <a name="4">4. Quy trình khởi tạo chuyên sâu (Bootstrapping Phases)</a>

Lệnh `kubeadm init` thực tế là một chuỗi các giai đoạn (phases) mà bạn có thể chạy riêng lẻ:

* **preflight:** Kiểm tra các yêu cầu hệ thống (swap phải tắt, port phải trống, kernel modules như `br_netfilter` phải được load).
* **certs:** Tạo bộ chứng chỉ tự ký trong `/etc/kubernetes/pki` (API server, etcd, front-proxy).
* **kubeconfig:** Tạo các tệp cấu hình cho admin, kubelet, controller-manager và scheduler.
* **kubelet-start:** Tạo tệp môi trường cho kubelet và khởi động dịch vụ.
* **control-plane:** Tạo các file manifest cho Static Pods.
* **etcd:** Tạo file manifest cho etcd cục bộ.
* **upload-config:** Lưu cấu hình kubeadm vào một ConfigMap trong cụm để sử dụng cho việc nâng cấp sau này.
* **mark-control-plane:** Gán nhãn (label) và taints cho master node để ngăn các ứng dụng người dùng chạy trên đó (mặc định).
* **bootstrap-token:** Tạo token để các node khác gia nhập.
* **addon:** Cài đặt Proxy và CoreDNS.

---

### <a name="5">5. Cơ chế Quản lý Chứng chỉ (Certificate Management)</a>

Một trong những thách thức lớn của Kubeadm là quản lý chứng chỉ:

* **Thời hạn:** Mặc định các chứng chỉ do Kubeadm tạo ra có thời hạn 1 năm.
* **Gia hạn:** Người dùng phải chủ động chạy `kubeadm certs renew all` hoặc thực hiện nâng cấp phiên bản (`kubeadm upgrade`) để tự động gia hạn chứng chỉ.
* **CA:** Root Certificate Authority (CA) mặc định có thời hạn 10 năm.

---

### Lời kết

Kubeadm mang lại sự hiểu biết thấu đáo về từng "con ốc vít" trong hệ thống Kubernetes. Đây là công cụ không thể thiếu cho các kỹ sư muốn làm chủ hoàn toàn hạ tầng, trong khi RKE2 là sự lựa chọn tối ưu cho tốc độ và sự an toàn trong doanh nghiệp.
