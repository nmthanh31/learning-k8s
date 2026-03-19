# Services

Tài liệu này tìm hiểu về Kubernetes Services, tập trung vào việc cho phép giao tiếp giữa các thành phần ứng dụng và chi tiết về cách tạo cũng như sử dụng service NodePort

Kubernetes Services cho phép các nhóm Pod khác nhau tương tác với nhau. Cho dù là kết nối giữa Front-end và các dịch vụ Backend hay tích hợp các nguồn dữ liệu bên ngoài, Service cung cấp một địa chỉ IP và cổng ổn định để các ứng dụng có thể giao tiếp với nhau một cách tin cậy.

## Các loại Kubernetes Services
Kubernetes hỗ trợ một số loại dịch vụ, mỗi loại dịch vụ có một mục đích riêng:
- NodePort: Ánh xạ một cổng trên node tới một cổng trên Pod
- ClusterIP: Tạo một IP ảo để giao tiếp nội bộ giữa các dịch vụ
- LoadBalancer: Cung cấp bộ cân bằng tải bên ngoài (được hỗ trợ trong cloud) để phân phối lưu lượng truy cập qua nhiều Pod
** Ghi nhớ **: Loại dịch vụ NodePort ánh xạ một cổng node cụ thể tới cổng mục tiêu trên Pod của bạn. Điều này giúp truy cập từ bên ngoài trong khi vẫn giữ nguyên việc nhằm mục tiêu cổng nội bộ.

## Service NodePort

### Các port quan trọng
Với dịch vụ NodePort, có ba cổng quan trọng cần xem xét:
1. Target Port (Cổng mục tiêu): Cổng trên Pod nơi ứng dụng đang lắng nghe
2. Port (Cổng dịch vụ): Cổng ảo trên chính dịch vụ đó trong cụm
3. NodePort: Cổng bên ngoài trên Kubernetes node (mặc định 30000-32767)
```
(Client bên ngoài)
        |
        |  truy cập NodeIP:NodePort
        v
+---------------------------+
|   Kubernetes Node         |
|                           |
|   NodePort: 30080         |  <-- Cổng mở trên Node (external)
|        |                  |
|        v                  |
|   Service Port: 80        |  <-- Cổng ảo của Service (ClusterIP)
|        |                  |
|        v                  |
|   Pod (Container)         |
|   TargetPort: 8080        |  <-- App thực sự lắng nghe ở đây
+---------------------------+
```

### Tạo một service NodePort
Quá trình tạo service NodePort bắt đầu bằng việc định nghĩa dịch vụ trong một tệp YAML. Tên định nghĩa tuân thủ cấu trúc tương tự như Deployment hoặc ReplicaSet, bao gồm apiVersion, kind, metadata và spec.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30080
  type: NodePort #Type của service
```

### Áp dụng Service NodePort

```bash
kubectl create -f service-nodeport.yaml
```

### Kiểm tra Service NodePort

```bash
kubectl get services
```

### Kết quả

```
NAME             TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)
myapp-service    NodePort   10.10.10.10     <none>        80:30080/TCP
```

### Truy cập Service NodePort

```bash
curl http://<NodeIP>:<NodePort>
```

## Service ClusterIP

Service ClusterIP là loại dịch vụ mặc định trong Kubernetes. Nó chỉ có thể truy cập được từ bên trong cụm. Nó hỗ trợ giao tiếp ổn định giữa các Pod (pod-to-pod)

### Trường hợp cần ClusterIP
- Vấn đề nan giải:
    - IP thay đổi liên tục: Các Pod trong Kubernetes có tính chất "tạm thời". Nếu một Pod back-end bị lỗi và được khởi tạo lại, nó sẽ nhận một địa chỉ IP mới. Nếu Front-end lưu cứng IP cũ, kết nối sẽ bị ngắt ngay lập tức.
    - Khó khăn khi mở rộng (Scaling): Nếu bạn có 3 Pod back-end, làm sao Front-end biết nên gửi yêu cầu đến IP nào trong 3 cái đó?
- Giải pháp với ClusterIP:
    - ClusterIP đóng vai trò như một "Số điện thoại tổng đài" duy nhất cho một nhóm Pod.
    - Thay vì gọi trực tiếp từng số di động cá nhân (IP của Pod), Front-end chỉ cần gọi vào số tổng đài (Cluster IP).
    - Tổng đài này sẽ tự động chuyển máy (Load Balancing) đến một trong các Pod back-end đang rảnh.
    - Ngay cả khi các Pod bên trong thay đổi, "Số tổng đài" (Cluster IP) vẫn giữ nguyên không đổi.
- Ghi nhớ: Mỗi dịch vụ (Service) trong Kubernetes sẽ tự động được gán một IP nội bộ và một tên miền (DNS) trong cụm. Các Pod khác nên dùng Cluster IP này để đảm bảo kết nối luôn thông suốt và ổn định.

### Tạo service ClusterIP

File YAML của service ClusterIP:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector: # Chọn các pod có label app: myapp
    app: myapp 
  ports:
    - protocol: TCP
      port: 80 # Cổng của service
      targetPort: 8080 # Cổng của pod
  type: ClusterIP #Type của service
```

### Áp dụng Service ClusterIP

```bash
kubectl create -f service-clusterip.yaml
```

### Kiểm tra Service ClusterIP

```bash
kubectl get services
```

### Kết quả

```
NAME             TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)
myapp-service    ClusterIP   10.10.10.10     <none>        80:30080/TCP
```

## Service LoadBalancer

Service LoadBalancer là loại dịch vụ được sử dụng để truy cập ứng dụng từ bên ngoài cụm. Nó được hỗ trợ trong cloud
Hạn chế của NodePort: Nếu bạn dùng NodePort, người dùng muốn truy cập web sẽ phải dùng địa chỉ IP của một trong các Node kèm theo một số cổng cao (ví dụ: http://192.168.1.2:30008).

- Vấn đề 1: Người dùng không muốn nhớ những con số IP và cổng phức tạp này. Họ chỉ muốn nhập http://votingapp.com.
- Vấn đề 2: Nếu Node đó bị hỏng, người dùng phải đổi sang IP của Node khác. Điều này cực kỳ bất tiện.

Giải pháp thủ công:
- Bạn có thể tự dựng một máy chủ riêng chạy Nginx hoặc HAProxy để làm "trạm điều hướng", nhận yêu cầu từ người dùng rồi chia tải vào các Node bên trong. Nhưng việc này lại làm tăng thêm công sức quản lý máy chủ đó.

Giải pháp với LoadBalancer:
- Nếu bạn triển khai Kubernetes trên các nền tảng đám mây lớn (như Google Cloud - GCP, AWS, hoặc Azure), mọi thứ trở nên cực kỳ đơn giản. Bạn chỉ cần đổi loại dịch vụ từ NodePort sang LoadBalancer.

Cách nó hoạt động:
- Khi bạn khai báo type: LoadBalancer, Kubernetes sẽ "nhờ" nền tảng đám mây (GCP/AWS/Azure) tự động tạo ra một bộ cân bằng tải (Load Balancer) chuyên nghiệp phía trước cụm.

- Bộ cân bằng tải này sẽ cung cấp cho bạn một địa chỉ IP duy nhất (External IP).

- Người dùng chỉ cần truy cập qua IP này, và nó sẽ tự động phân phối lưu lượng vào các Node bên trong một cách thông minh.

** Ghi nhớ **: Loại dịch vụ LoadBalancer chỉ hoạt động thực sự trên các môi trường đám mây được hỗ trợ. Trong các môi trường giả lập (như VirtualBox hoặc máy cá nhân), nó thường sẽ hoạt động giống như NodePort và không cung cấp IP bên ngoài tự động.

Dưới đây là cách bạn khai báo dịch vụ LoadBalancer cho ứng dụng của mình:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: voting-service
spec:
  type: LoadBalancer # Chỉ cần đổi từ NodePort sang LoadBalancer
  ports:
    - port: 80         # Cổng mà Load Balancer bên ngoài lắng nghe
      targetPort: 80   # Cổng ứng dụng bên trong Pod
  selector:
    app: voting-app
```

Quy trình kiểm tra

Sau khi tạo dịch vụ, bạn dùng lệnh sau để xem IP mà đám mây đã cấp cho bạn:
```bash
kubectl get services
```

Kết quả mong đợi (trên đám mây):
```terminal
NAME             TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
voting-service   LoadBalancer   10.10.10.150    35.190.24.12    80:31250/TCP   5m
```