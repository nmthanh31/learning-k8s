# Deployments 

Bài viết này giải thích cách Kubernetes Deployments hỗ trợ việc triển khai ứng dụng, đảm bảo cập nhật hiệu quả và tính sẵn sàng cao trong môi trường production.

Trong bài viết này, chúng tôi sẽ giải thích cách Deployment tinh giản quy trình triển khai ứng dụng, đảm bảo các bản cập nhật diễn ra suôn sẻ và hệ thống luôn hoạt động ổn định.

Khi triển khai một ứng dụng như web server, bạn thường chạy nhiều phiên bản (instances) để xử lý tải và đảm bảo thời gian hoạt động. Khi các phiên bản mới của ứng dụng có mặt trên Docker registry, việc nâng cấp không gián đoạn là cực kỳ quan trọng. Vì việc nâng cấp tất cả các phiên bản cùng lúc có thể gây gián đoạn cho người dùng đang truy cập, Kubernetes Deployments hỗ trợ Rolling Updates — cập nhật từng phiên bản một. Ngoài ra, nếu một bản cập nhật gây ra lỗi, bạn có thể nhanh chóng Rollback (hoàn tác) các thay đổi. Deployment cũng cho phép bạn đóng gói nhiều thay đổi — như cập nhật phiên bản web server, mở rộng tài nguyên, hoặc điều chỉnh cấu hình — và áp dụng chúng cùng một lúc.

Kubernetes Deployments được xây dựng dựa trên các khái niệm nền tảng: Pods bao bọc các phiên bản ứng dụng riêng lẻ, trong khi ReplicaSets quản lý nhiều Pods. Deployment là một cấu trúc cấp cao hơn, không chỉ tạo ra ReplicaSet mà còn điều phối các hoạt động cập nhật cuốn chiếu, hoàn tác, và tạm dừng/tiếp tục.

Dưới đây là ví dụ về định nghĩa ReplicaSet để bạn dễ hình dung ngữ cảnh:
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-replicaset
  labels:
    app: myapp
    type: front-end
spec:
  replicas: 3
  selector:
    matchLabels:
      type: front-end
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp
        type: front-end
    spec:
      containers:
      - name: nginx-container
        image: nginx
```

Định nghĩa Deployment tương ứng rất giống với ReplicaSet, thay đổi chính nằm ở trường kind. Dưới đây là file YAML của Deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
    type: front-end
spec:
  replicas: 3
  selector:
    matchLabels:
      type: front-end
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp
        type: front-end
    spec:
      containers:
      - name: nginx-container
        image: nginx
```

Để tạo Deployment, hãy lưu nội dung YAML trên vào một tệp (ví dụ: deployment-definition.yml) và chạy lệnh:

```bash
kubectl create -f deployment-definition.yml
```


Sau khi áp dụng, bạn có thể xác nhận việc tạo Deployment bằng các lệnh sau:

- Liệt kê các Deployment:
    ```bash
    kubectl get deployments
    ```

- Kiểm tra ReplicaSet liên quan:
    ```bash
    kubectl get replicaset
    ```

- Xem các Pod được tạo bởi Deployment:
    ```bash
    kubectl get pods
    ```

Ví dụ, bạn có thể thấy kết quả tương tự như sau:

```bash
> kubectl create -f deployment-definition.yml
deployment "myapp-deployment" created

> kubectl get deployments
NAME                DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
myapp-deployment    3         3         3            3           21s

> kubectl get replicaset
NAME                             DESIRED   CURRENT   READY   AGE
myapp-deployment-6795844b58     3         3         3       2m

> kubectl get pods
NAME                                           READY   STATUS    RESTARTS   AGE
myapp-deployment-6795844b58-5rbj1             1/1     Running   0          2m
myapp-deployment-6795844b58-h4w55             1/1     Running   0          2m
myapp-deployment-6795844b58-1fjhv             1/1     Running   0          2m
```

Để xem tất cả các đối tượng đã tạo cùng một lúc, hãy sử dụng:

```bash
kubectl get all
```

Lệnh này hiển thị Deployment cùng với ReplicaSet và các Pod liên quan của nó.

