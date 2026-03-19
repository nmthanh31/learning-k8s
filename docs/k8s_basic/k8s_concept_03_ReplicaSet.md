# ReplicaSets

- Bài viết này tìm hiểu về ReplicaSets trong Kubernetes, tập trung vào vai trò của chúng trong việc quản lý các bản sao của pod để đảm bảo tính sẵn sàng cao và khả năng mở rộng.

- Trong bài học này, chúng ta sẽ tìm hiểu khái niệm về bản sao (replicas) trong Kubernetes và tầm quan trọng của các bộ điều khiển sao chép (replication controllers) trong việc đảm bảo tính sẵn sàng cao. Hãy tưởng tượng một kịch bản nơi ứng dụng của bạn chỉ chạy một pod duy nhất. Nếu pod đó gặp sự cố, người dùng sẽ mất quyền truy cập vào ứng dụng. Để tránh thời gian gián đoạn (downtime), việc chạy nhiều phiên bản (hoặc pod) cùng lúc là cực kỳ quan trọng. Một replication controller đảm bảo rằng số lượng pod mong muốn luôn chạy trong cluster của bạn, cung cấp cả tính sẵn sàng cao và cân bằng tải.

- Ngay cả khi bạn chỉ định chạy một pod duy nhất, replication controller sẽ tự động khởi tạo một pod mới nếu pod hiện tại bị lỗi. Cho dù bạn cần một pod hay một trăm pod, replication controller sẽ duy trì con số đó và phân phối tải trên nhiều phiên bản. Ví dụ, nếu lượng người dùng tăng lên, các pod bổ sung có thể được triển khai. Trong trường hợp một node hết tài nguyên, các pod mới có thể tự động được lập lịch trên các node khác.

- Như hình minh họa trên, replication controller trải rộng trên nhiều node, đảm bảo cân bằng tải hiệu quả và khả năng mở rộng ứng dụng khi nhu cầu tăng cao.

Một khía cạnh quan trọng khác là hiểu sự khác biệt giữa Replication Controller và ReplicaSet. Cả hai đều quản lý các bản sao của pod, nhưng Replication Controller là công nghệ cũ hơn, đang dần được thay thế bằng ReplicaSet tiên tiến hơn. Mặc dù có những khác biệt nhỏ trong cách thực hiện, chức năng cốt lõi của chúng là tương tự nhau. Trong tất cả các bản demo và triển khai sau này, chúng ta sẽ tập trung vào việc sử dụng ReplicaSets.

## Tạo một ReplicationController

Để tạo một ReplicationController, hãy bắt đầu bằng việc xác định tệp cấu hình có tên rc-definition.yaml. Giống như bất kỳ tệp định nghĩa Kubernetes nào, nó bao gồm các phần sau: apiVersion, kind, metadata và spec.

- API Version: Đối với ReplicationController, sử dụng v1.

- Kind: Đặt là ReplicationController.

- Metadata: Cung cấp một cái tên duy nhất (ví dụ: myapp-rc) cùng với các nhãn (labels) để phân loại ứng dụng (như app và type).

- Spec: Định nghĩa trạng thái mong muốn của đối tượng:
  - Chỉ định số lượng bản sao (replicas).

  - Bao gồm phần template cho định nghĩa pod. (Lưu ý: không bao gồm apiVersion và kind từ tệp pod gốc; chỉ bao gồm metadata, labels và spec của pod, được thụt lề như một phần tử con của template.)

Dưới đây là một ví dụ về định nghĩa ReplicationController:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: myapp-rc
  labels:
    app: myapp
    type: front-end
spec:
  replicas: 3
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

Sau khi tệp đã sẵn sàng, hãy tạo replication controller bằng lệnh:

```bash
kubectl create -f rc-definition.yaml
```

Sau khi tạo, hãy kiểm tra chi tiết replication controller bằng:

```bash
kubectl get replicationcontroller
```

Lệnh này hiển thị số lượng bản sao mong muốn, số lượng hiện tại và số lượng pod đã sẵn sàng. Để liệt kê các pod được tạo bởi replication controller, hãy chạy:

```bash
kubectl get pods
```

Bạn sẽ nhận thấy rằng tên các pod bắt đầu bằng tên của replication controller (ví dụ: myapp-rc-xxxx), cho thấy chúng được tạo tự động.

## Giới thiệu về ReplicaSets

Một ReplicaSet hoạt động tương tự như ReplicationController nhưng có một số khác biệt chính:

- API Version và Kind:
  - Đối với ReplicaSet, hãy đặt `apiVersion` thành `apps/v1` (thay vì v1).

  - `Kind` phải là `ReplicaSet`.

- Yêu cầu về Selector (Bộ chọn):
  - ReplicaSet yêu cầu một bộ chọn (selector) rõ ràng trong cấu hình của nó. Thường được định nghĩa dưới matchLabels, bộ chọn xác định pod nào mà ReplicaSet sẽ quản lý. Tính năng này cũng cho phép ReplicaSet "nhận nuôi" các pod hiện có khớp với các nhãn đã chỉ định, ngay cả khi chúng không phải do nó tạo ra.

Dưới đây là một ví dụ về tệp định nghĩa ReplicaSet có tên replicaset-definition.yml:

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

Tạo ReplicaSet bằng lệnh:

```bash
kubectl create -f replicaset-definition.yml
```

Sau đó kiểm tra ReplicaSet và các pod của nó:

```bash
kubectl get replicaset
kubectl get pods
```

ReplicaSet giám sát các pod có nhãn khớp. Nếu các pod đã tồn tại với các nhãn này, ReplicaSet sẽ tiếp nhận chúng thay vì tạo mới ngay lập tức. Tuy nhiên, phần template vẫn rất cần thiết để tạo các pod mới nếu bất kỳ pod nào đang được quản lý gặp sự cố.

## Vai trò của Labels và Selectors

Nhãn (Labels) đóng vai trò quan trọng trong việc tổ chức và lựa chọn các tập hợp đối tượng trong Kubernetes. Khi triển khai nhiều phiên bản (ví dụ: ba pod cho một ứng dụng front-end), ReplicaSet sử dụng labels và selectors để quản lý các pod này một cách hiệu quả. Hãy xem xét cấu hình mẫu làm nổi bật mối quan hệ giữa nhãn của pod và bộ chọn của ReplicaSet:

```yml
# replicaset-definition.yml
selector:
  matchLabels:
    tier: front-end

# pod-definition.yml
metadata:
  name: myapp-pod
  labels:
    tier: front-end
```

ReplicaSet sử dụng bộ chọn của nó để giám sát và quản lý các pod có nhãn tier: front-end. Khái niệm labels và selectors này được sử dụng rộng rãi trong Kubernetes để duy trì trật tự và hiệu quả.

## Mở rộng (Scaling) ReplicaSets

Mở rộng ReplicaSets cho phép ứng dụng của bạn thích ứng với nhu cầu thay đổi. Giả sử bạn bắt đầu với ba bản sao và sau đó cần tăng lên sáu. Có nhiều cách tiếp cận:

Chỉnh sửa tệp định nghĩa:
Cập nhật trường replicas trong tệp định nghĩa ReplicaSet thành 6, sau đó chạy:

```bash
kubectl replace -f replicaset-definition.yml
```

Sử dụng lệnh Scale:
Ngoài ra, bạn có thể sử dụng lệnh kubectl scale:

```bash
kubectl scale --replicas=6 -f replicaset-definition.yml
```

Hoặc nếu bạn muốn sử dụng tên của ReplicaSet:

```bash
kubectl scale --replicas=6 replicaset/myapp-replicaset
```

<Callout icon="lightbulb" color="#1CB2FE">
Hãy nhớ rằng, nếu bạn sử dụng lệnh scale, các thay đổi chỉ được cập nhật trong trạng thái của cluster. Tệp định nghĩa gốc vẫn sẽ hiển thị số lượng bản sao trước đó cho đến khi nó được sửa đổi thủ công.
</Callout>

Cũng có các tùy chọn nâng cao để tự động mở rộng (auto-scaling) ReplicaSets dựa trên tải; tuy nhiên, chủ đề đó nằm ngoài phạm vi bài học này.

