# Pods (Đơn vị nhỏ nhất trong K8s)

Bài viết này cung cấp hướng dẫn chuyên sâu về Kubernetes Pods, bao gồm việc triển khai, mở rộng quy mô và quản lý chúng trong một cluster Kubernetes.

Với Kubernetes, mục tiêu là chạy các container trên các worker node, nhưng thay vì triển khai container trực tiếp, Kubernetes bao bọc chúng trong một đối tượng gọi là Pod. Pod đại diện cho một phiên bản duy nhất của ứng dụng và là đơn vị nhỏ nhất có thể triển khai trong Kubernetes.

## Kịch bản cơ bản

Trong trường hợp đơn giản nhất, một cluster Kubernetes đơn node có thể chạy một phiên bản ứng dụng của bạn bên trong một Docker container được bao bọc bởi một pod.

## Mở rộng quy mô (Scaling)

Khi lưu lượng người dùng tăng lên, bạn có thể mở rộng ứng dụng của mình bằng cách tạo thêm các phiên bản mới—mỗi phiên bản chạy trong một pod riêng biệt. Cách tiếp cận này giúp cô lập từng phiên bản, cho phép Kubernetes phân phối các pod trên các node có sẵn khi cần thiết.

Thay vì thêm nhiều container vào cùng một pod, các pod bổ sung sẽ được tạo ra. Ví dụ, việc chạy hai phiên bản trong các pod riêng biệt cho phép chia sẻ tải trên node hoặc thậm chí trên nhiều node nếu nhu cầu tăng cao và cần thêm tài nguyên cluster.

**Ghi nhớ:** Việc mở rộng ứng dụng trong Kubernetes liên quan đến việc tăng hoặc giảm số lượng pods, chứ không phải số lượng container trong một pod duy nhất.

## Multi-container Pods (Pod đa container)

Thông thường, mỗi pod chứa một container duy nhất chạy ứng dụng chính của bạn. Tuy nhiên, một pod cũng có thể chứa nhiều container, thường là các container bổ sung hỗ trợ lẫn nhau chứ không phải là bản sao của nhau.

Ví dụ: Bạn có thể bao gồm một "helper container" (container hỗ trợ) bên cạnh container ứng dụng chính để thực hiện các tác vụ như xử lý dữ liệu hoặc tải tệp lên. Cả hai container trong pod chia sẻ cùng một không gian mạng (cho phép giao tiếp trực tiếp qua localhost), các volume lưu trữ và vòng đời, đảm bảo chúng khởi động và dừng lại cùng nhau.

So sánh với Docker thuần túy

Để hiểu rõ hơn, hãy xem xét một ví dụ Docker cơ bản. Giả sử ban đầu bạn triển khai ứng dụng bằng một lệnh đơn giản:
```bash
docker run python-app
```

Khi tải tăng lên, bạn có thể khởi chạy thêm các phiên bản thủ công:
```bash
docker run python-app
docker run python-app
docker run python-app
```

Bây giờ, nếu ứng dụng của bạn cần một container hỗ trợ để giao tiếp với từng phiên bản, việc quản lý các liên kết (links), mạng tùy chỉnh và các volume chia sẻ thủ công sẽ trở nên phức tạp. Bạn sẽ phải chạy các lệnh như:
```bash
docker run helper --link app1
docker run helper --link app2
```

Với Kubernetes pods, những thách thức này được giải quyết tự động. Khi một pod được định nghĩa với nhiều container, chúng sẽ tự động chia sẻ lưu trữ, không gian mạng và vòng đời—đảm bảo sự phối hợp liền mạch và đơn giản hóa việc quản lý.

## Triển khai Pods

Một phương pháp phổ biến để triển khai pod là sử dụng lệnh kubectl run. Ví dụ, lệnh sau đây tạo một pod triển khai một phiên bản của image nginx, lấy từ kho lưu trữ Docker:

```bash
kubectl run nginx --image nginx
```


Sau khi triển khai, bạn có thể kiểm tra trạng thái của pod bằng lệnh kubectl get pods. Ban đầu, pod có thể ở trạng thái "ContainerCreating", sau đó chuyển sang trạng thái "Running" khi container ứng dụng hoạt động. Dưới đây là ví dụ về một phiên làm việc:
```bash
kubectl get pods
# Kết quả:
# NAME                   READY   STATUS              RESTARTS   AGE
# nginx-8586cf59-whssr   0/1     ContainerCreating   0          2s

kubectl get pods
# Kết quả sau vài giây:
# NAME                   READY   STATUS    RESTARTS   AGE
# nginx-8586cf59-whssr   1/1     Running   0          8s
```

Tại giai đoạn này, lưu ý rằng quyền truy cập bên ngoài vào web server nginx vẫn chưa được cấu hình. Dịch vụ này chỉ có thể truy cập được trong nội bộ node. Trong các bài viết tới, chúng ta sẽ tìm hiểu cách cấu hình truy cập bên ngoài thông qua mạng và dịch vụ của Kubernetes.

Ghi nhớ: Sau khi thành thạo việc triển khai pod, hãy tiến tới cấu hình mạng (networking) và dịch vụ (service) để công khai ứng dụng của bạn cho người dùng cuối.

Hướng dẫn viết file YAML cho Pod

Hướng dẫn này giúp bạn hiểu cấu trúc và cách tự viết một file cấu hình YAML để triển khai Pod trong Kubernetes.

Trong Kubernetes, các đối tượng (như Pod, Service, Deployment) thường được tạo thông qua các file cấu hình YAML. Cách tiếp cận này được gọi là Declarative (Khai báo), giúp bạn có thể lưu trữ, chia sẻ và quản lý hạ tầng dưới dạng mã (Infrastructure as Code).

# Cách viết file YAML cho POD

## 1. Cấu trúc 4 trường bắt buộc (Cấp cao nhất)

Mọi file YAML trong Kubernetes đều phải bắt đầu bằng 4 trường dữ liệu chính sau đây:

- apiVersion: Xác định phiên bản của API Kubernetes. Đối với Pod, giá trị luôn là v1.

- kind: Xác định loại đối tượng bạn muốn tạo. Ở đây là Pod.

- metadata: Chứa các thông tin định danh như name (tên Pod) và labels (nhãn).

- spec: Phần quan trọng nhất, định nghĩa cấu hình chi tiết bên trong (ví dụ: container nào, image nào).

<Callout icon="lightbulb" color="#1CB2FE">
Ghi nhớ: Hãy chú ý đến việc thụt lề (indentation). Kubernetes sử dụng khoảng trắng (spaces) chứ không phải phím Tab để phân cấp dữ liệu.
</Callout>

## 2. Chi tiết từng thành phần

### apiVersion & kind

Đây là phần khai báo loại đối tượng:
```bash
apiVersion: v1
kind: Pod
```

### Metadata (Dữ liệu đặc tả)

Đây là nơi bạn đặt tên cho Pod. Tên này phải là duy nhất trong một Namespace.
```bash
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type: front-end
```

labels: Giúp bạn nhóm các Pod lại với nhau để quản lý sau này (ví dụ: dùng Service để trỏ tới các Pod có nhãn app: myapp).

### Spec (Đặc tả kỹ thuật)

Phần này định nghĩa các container sẽ chạy trong Pod. Trường containers là một danh sách (list), vì vậy mỗi phần tử sẽ bắt đầu bằng dấu gạch ngang (-).
```bash
spec:
  containers:
    - name: nginx-container
      image: nginx
      ports:
        - containerPort: 80
```

## 3. Một file YAML hoàn chỉnh

Dưới đây là file pod-definition.yaml hoàn chỉnh để bạn có thể sao chép và sử dụng:

```bash
apiVersion: v1
kind: Pod
metadata:
  name: web-server-pod
  labels:
    env: production
    role: web
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      resources:
        limits:
          memory: "128Mi"
          cpu: "500m"
      ports:
        - containerPort: 80
```

## 4. Các lệnh thao tác với file YAML

Sau khi bạn đã soạn thảo xong file .yaml, hãy sử dụng các lệnh kubectl sau để làm việc với cluster:

Tạo Pod từ file:
```bash
kubectl create -f pod-definition.yaml
```

(Hoặc sử dụng kubectl apply -f để cập nhật nếu file đã tồn tại)

Kiểm tra trạng thái:
```bash
kubectl get pods
```

Xem chi tiết cấu hình và lỗi:
```bash
kubectl describe pod web-server-pod
```
Xóa Pod:
```bash
kubectl delete -f pod-definition.yaml
``` 

## 5. Các lỗi thường gặp khi viết YAML

Sai thụt lề: Chỉ cần lệch 1 khoảng trắng cũng sẽ khiến Kubernetes không hiểu được file.

Dùng phím Tab: Luôn sử dụng phím Space (dấu cách).

Sai apiVersion: Một số đối tượng như Deployment yêu cầu apps/v1 thay vì chỉ v1.

Kết luận: Viết file YAML là kỹ năng cơ bản nhất của một kỹ sư DevOps khi làm việc với Kubernetes. Hãy bắt đầu với các Pod đơn giản trước khi tiến tới các đối tượng phức tạp hơn như Deployments hay StatefulSets!