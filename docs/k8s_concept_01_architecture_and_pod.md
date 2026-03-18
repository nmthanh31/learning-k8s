# Bài 1: Kiến trúc Kubernetes và "Hạt nhân" Pod

Chào mừng bạn đến với Kubernetes (K8s) từ con số 0! Hãy quên đi các hệ thống lớn lao, chúng ta sẽ bắt đầu từ những khái niệm nhỏ nhất và xây dựng dần lên.

## 1. Tại sao lại cần Kubernetes?
Trước đây, chúng ta chạy ứng dụng trực tiếp trên máy chủ vật lý, rất dễ xung đột thư viện. Sau đó, **Docker (Container)** ra đời giúp đóng gói ứng dụng và tập lệnh chạy nó vào một "hộp" (container) cô lập, chạy ở đâu cũng giống nhau.

Nhưng khi bạn có hàng trăm container:
- Trừ khi bạn canh chừng 24/7, làm sao biết container nào vừa chết để bật lại?
- Khi lượng truy cập tăng vọt, làm sao tự động nhân bản thêm vùng chứa ứng dụng?
- Làm sao các container trên máy chủ A nói chuyện được với container trên máy chủ B?

**Kubernetes** ra đời để làm **Người Khúc Trưởng (Orchestrator)** quản lý đội quân container đó.

## 2. Cấu trúc của một Cụm (Cluster)
Một "cụm" (cluster) K8s là một tập hợp các máy chủ kết nối với nhau, chia làm 2 phe:

### A. Master Node (Control Plane) - Ban Lãnh Đạo
Đây là "bộ não" đưa ra các quyết định hệ thống (vd: "hãy chạy thêm 3 ứng dụng này cho tôi").
- **kube-apiserver**: Cánh cửa duy nhất. Bất cứ ai (kể cả bạn hay các phần mềm khác) muốn yêu cầu K8s làm gì đều phải gọi qua API Server.
- **etcd**: Cuốn "sổ cái" lưu trữ toàn bộ sự thật về hệ thống (cấu hình, trạng thái hiện tại). Nếu API Server bị crash nhưng `etcd` còn, hệ thống vẫn phục hồi được.
- **kube-scheduler**: Người sắp xếp. Khi có một ứng dụng mới cần chạy, scheduler sẽ nhìn vào các máy chủ xem máy nào còn rảnh RAM, CPU để nhét ứng dụng đó vào.
- **kube-controller-manager**: Đội ngũ thanh tra. Liên tục so sánh trạng thái hiện tại (Current State) với trạng thái bạn mong muốn (Desired State). Nếu cái bạn muốn là 3 ứng dụng chạy, nhưng hiện tại chỉ có 2, Controller Manager sẽ bắt tay vào việc tạo thêm 1.

### B. Worker Node - Đội Ngũ Thi Công
Đây là các máy ảo hoặc máy vật lý trực tiếp chạy ứng dụng (container) của bạn.
- **kubelet**: Kẻ "chỉ điểm" báo cáo tình trạng sức khỏe của Node lên cho API Server, đồng thời nhận lệnh từ API Server để khởi động các ứng dụng.
- **kube-proxy**: Trạm trung chuyển mạng, đảm bảo các kết nối mạng có thể đi đến đúng ứng dụng bên trong Node.
- **Container Runtime**: Phần mềm trực tiếp chạy container (như containerd, Docker, CRI-O).

---

## 3. Khái niệm Cốt Lõi: POD
Đây là bài học quan trọng nhất: **KUBERNETES KHÔNG QUẢN LÝ TRỰC TIẾP CONTAINER.**

Thay vào đó, nó bọc container vào một khối gọi là **Pod**. Pod là "nguyên tử" (đơn vị nhỏ nhất) trong K8s.

- **Một Pod = Một môi trường cô lập**. 
- Thường thì **1 Pod chỉ chứa 1 Container** (Mô hình 1-1). 
- Tuy nhiên, K8s cho phép **1 Pod chứa nhiều Container** nếu chúng cực kỳ gắn bó với nhau (Ví dụ: 1 container chạy Web Server, 1 container chạy song song để làm nhiệm vụ kéo code từ Github về liên tục. Chúng cần chung ổ cứng, chung localhost).

### Đặc điểm sinh học của Pod
- **Pod dùng chung Network**: Các container trong cùng 1 Pod có thể nói chuyện với nhau thông qua `localhost`.
- **Pod dùng chung Storage**: Nếu cấu hình, các container trong Pod có thể đọc/ghi chung một thư mục.
- **Pod là "phù du" (ephemeral)**: Pod được sinh ra, lớn lên và CHẾT ĐI. Khi một máy chủ (Worker Node) bị sập, K8s sẽ KHÔNG cứu sống lại cái Pod cũ đó. Thay vào đó, nó sẽ đẻ ra một bản SAO CHÉP (Pod mới y hệt) ở một máy chủ khác.

> Vì Pod có thể chết bất cứ lúc nào, làm thế nào để ta đảm bảo ứng dụng luôn chạy nếu lỡ Pod bị hỏng? Hãy sang Bài 2.
