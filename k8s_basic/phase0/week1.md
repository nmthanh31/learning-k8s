# Phase 0 - Week 1: Tổng quan về K8s (Kiến trúc & Pod | Mọi thứ bắt đầu từ đâu?)

<a name="top"></a>
## 📑 Mục lục
- [1. Kiến trúc của K8s](#kiến-trúc-của-k8s)
    - [1.1 Control Plane](#1-control-plane)
    - [1.2 Worker Node](#2-worker-node)
- [2. Pod Lifecycle (Vòng đời của Pod)](#pod-lifecycle-vòng-đời-của-pod)
    - [2.1 Trạng thái chi tiết (Detailed Status & Reasons)](#1-trạng-thái-chi-tiết-detailed-status--reasons)
    - [2.2 Các chỉ số sức khỏe (Pod Conditions)](#2-các-chỉ-số-sức-khỏe-pod-conditions)
    - [2.3 Các Exit Codes phổ biến](#2-các-exit-codes-phổ-biến-tại-sao-container-dừng)
- [3. Câu lệnh kiểm tra và xử lý lỗi](#câu-lệnh-kiểm-tra-và-xử-lý-lỗi)
- [4. Hành trình của lệnh kubectl apply](#điều-gì-xảy-ra-khi-bạn-chạy-kubectl-apply--f-podyaml)

---

Tài liệu này dùng để học căn bản những thứ nên biết để bắt đầu học K8s cơ bản.

## Kiến trúc của K8s

Kiến trúc của K8s gồm 2 phần chính:
1. Control Plane
2. Worker Node

### 1. Control Plane
Control Plane là một thành phần đóng vai trò điều khiển cluster K8s. Gồm các thành phần sau:
1. kube-apiserver: Trung tâm giao tiếp giữa các thành phần trong control plane, worker nodes và với người dùng (CLI). Cung cấp API cho các thao tác quản trị cluster
2. etcd: Đây là một nơi lưu trữ dạng key-value dùng để lưu trữ toàn bộ thông tin trạng thái của cluster, bao gồm cả thông tin vể cấu hình, trạng thái thực tế của các tài nguyên
3. kube-scheduler: Thành phần này chịu trách nhiệm quan sát các pod mới tạo chưa được gán Node và chọn Node phù hợp cho Pod đó (Chỉ phân bố vào Node chứ ko làm gì khác)
4. kube-controller-manager: Thành phần chính thực hiện việc điều khiển cluster. Nó bao gồm một số controller khác nhau để quản lý các tài nguyên khác nhau trong cluster.
5. cloud-controller-manager: Thành phần này giúp tách biệt logic của cluster và logic của nhà cung cấp Cloud. Nó chịu trách nhiệm giao tiếp với các API của nhà cung cấp đám mây (AWS, Azure, GCP...) để quản lý các tài nguyên như Load Balancer, Storage Volumes hoặc Nodes trên đám mây.

### 2. Worker Node
Worker Node là nơi chứa các Pod và thực hiện các tác vụ của cluster (Nơi thực sự sẽ chạy các workload). Thành phần bên trong Worker Node:
1. kubelet: Nơi đảm bảo các pod hoạt động bao gồm cả các container và các tài nguyên liên quan
2. kube-proxy: Chạy trên mỗi node và quản lý các quy tắc mạng (iptables/IPVS). Nó giúp điều hướng traffic đến đúng các Pod và thực hiện cân bằng tải cơ bản cho Service.
3. Container Runtime: Nơi thực sự khởi chạy các container trên node (Docker, containerd)

**Control plane giao tiếp chủ yếu qua kube-apiserver. kube-apiserver giao tiếp với worker nodes qua kubelet.**

[⬆ Quay lại đầu trang](#top)

## Pod Lifecycle (Vòng đời của Pod)

Pod không "sống" mãi mãi. Nó được tạo ra, gán định danh (UID) và tồn tại cho đến khi kết thúc hoặc bị xóa.

### 1. Trạng thái chi tiết (Detailed Status & Reasons):
Đây là những gì bạn thực sự nhìn thấy ở cột **STATUS** khi chạy `kubectl get pods`. Nó cho biết chính xác Pod đang kẹt ở bước nào:

| Trạng thái hiển thị | Ý nghĩa thực tế | Thành phần xử lý |
| :--- | :--- | :--- |
| **Pending** | Pod đã được chấp nhận bởi hệ thống nhưng chưa được gán Node hoặc đang chờ kéo image/khởi tạo. | **Control Plane** |
| **PodScheduled** | Scheduler đã chọn được Node cho Pod nhưng Kubelet chưa bắt đầu làm việc. | **kube-scheduler** |
| **ImagePullBackOff** | Không thể kéo Image (sai tên, sai tag, hoặc thiếu quyền access). K8s đang tạm dừng để thử lại sau. | **Kubelet** |
| **ContainerCreating** | Đang tạo container và thiết lập mạng/volume. | **Kubelet** & **Runtime** |
| **Initializing** | Đang chạy các **Init Containers** (các container chạy mồi trước khi container chính khởi động). | **Kubelet** |
| **Running** | Ít nhất một container chính đang chạy. | **Kubelet** |
| **CrashLoopBackOff** | Container khởi động lên rồi bị chết ngay lập tức. K8s đang tự động khởi động lại nó theo chu kỳ tăng dần. | **Kubelet** & **Ứng dụng** |
| **Terminating** | Pod đang bị xóa, đang chờ các container dừng hẳn. | **Kubelet** |
| **OOMKilled** | Container bị hệ thống "trảm" vì dùng quá lượng RAM cho phép (Limit). | **Kubelet** & **OS** |
| **Completed** | Ứng dụng đã hoàn thành công việc và thoát thành công (Exit 0). | **Ứng dụng** |

### 2. Các chỉ số sức khỏe (Pod Conditions):
Khi bạn chạy `kubectl describe pod`, hãy nhìn vào phần **Conditions**. Đây là "nguồn sự thật" để K8s quyết định trạng thái của Pod:
*   **PodScheduled**: Đã tìm được Node phù hợp chưa?
*   **Initialized**: Các Init Containers đã chạy xong chưa?
*   **ContainersReady**: Tất cả các container trong Pod đã sẵn sàng chưa?
*   **Ready**: Pod đã sẵn sàng để nhận traffic (đã vượt qua Readiness Probe)?

### 3. Các Exit Codes phổ biến (Tại sao container dừng?):
K8s dựa vào Exit Code của Linux để xác định nguyên nhân:
*   **Exit 0**: Hoàn thành thành công (thường gặp ở Jobs).
*   **Exit 1**: Lỗi ứng dụng (Application crash).
*   **Exit 127**: Sai lệnh thực thi (Command not found) trong image.
*   **Exit 137 (128 + 9)**: Bị ép dừng ngay lập tức. Thường là **OOMKilled** (Vượt giới hạn RAM).
*   **Exit 143 (128 + 15)**: Dừng an toàn (Graceful Termination/SIGTERM).

[⬆ Quay lại đầu trang](#top)

---

## Câu lệnh kiểm tra và xử lý lỗi

Khi hệ thống gặp lỗi, hãy kiểm tra theo thứ tự "từ ngoài vào trong":

1. **Kiểm tra trạng thái tổng quát:**
   `kubectl get pods -n <namespace> -o wide`
   *(Nhìn STATUS và số lần RESTARTS)*

2. **Kiểm tra nhật ký sự kiện (Cực kỳ quan trọng):**
   `kubectl describe pod <tên-pod>`
   *(Kéo xuống phần **Events** để xem K8s đã làm gì: Pull image lỗi? Thiếu RAM? Hay Scheduler không tìm được Node?)*

3. **Kiểm tra Log ứng dụng:**
   `kubectl logs <tên-pod>`
   `kubectl logs -f <tên-pod>` *(Theo dõi log thời gian thực)*
   `kubectl logs <tên-pod> --previous` *(Xem log của lần crash ngay trước đó)*

4. **Kiểm tra trực tiếp:**
   `kubectl exec -it <tên-pod> -- /bin/sh`

[⬆ Quay lại đầu trang](#top)

---

## Điều gì xảy ra khi bạn chạy `kubectl apply -f pod.yaml`?
    
Đây là hành trình "khai báo trạng thái" của K8s:

1.  **Giai đoạn CLI:** `kubectl` kiểm tra cú pháp và gửi yêu cầu tới **kube-apiserver**.
2.  **Giai đoạn Lưu trữ:** **API Server** xác thực quyền truy cập, sau đó ghi cấu hình Pod vào **etcd**.
3.  **Giai đoạn Lập lịch:** **kube-scheduler** thấy Pod mới chưa có Node, nó tính toán và chọn Node tối ưu nhất, rồi báo lại cho API Server.
4.  **Giai đoạn Thực thi:** **Kubelet** trên Node được chọn nhận lệnh, gọi **Container Runtime** để pull image và start container.
5.  **Giai đoạn Phản hồi:** Kubelet cập nhật trạng thái Pod về API Server để lưu vào etcd.

**Nguyên lý cốt lõi:** K8s hoạt động theo cơ chế **Desired State** (Trạng thái mong muốn). Bạn không ra lệnh "Làm đi", bạn chỉ nói "Tôi muốn như thế này", và Control Plane sẽ tự động điều phối để hiện thực hóa điều đó.

---
[⬆ Quay lại đầu trang](#top)
