# 🛠️ Kubernetes (kubectl) Commands & Syntax Cheat Sheet

Tài liệu này tổng hợp các cú pháp và lệnh `kubectl` phổ biến nhất trong quá trình thao tác, vận hành nhà quản lý tài nguyên trên Kubernetes Cluster.

---

## 1. 🚀 Lệnh Cơ Bản (Cluster & Trạng thái chung)

```bash
# Xem thông tin điều phối của hệ thống (Cluster info)
kubectl cluster-info

# Liệt kê tất cả các Worker/Master Node trong Cluster
kubectl get nodes
kubectl get nodes -o wide # Xem chi tiết cả IP nội bộ, OS, Container Runtime

# Xem cấu hình kết nối của kubectl hiện tại
kubectl config view

# Chuyển đổi context (nơi thao tác qua lại giữa nhiều Cluster khác nhau)
kubectl config use-context <context-name>
```

---

## 2. 📦 Quản lý Pod (Pods)

```bash
# Xem danh sách Pod (trong namespace mặc định `default`)
kubectl get pods

# Xem danh sách Pod ở TẤT CẢ namespace
kubectl get pods --all-namespaces
# Hoặc viết tắt: kubectl get pods -A

# Xem chi tiết cụ thể của một Pod (Events, Status, Label) -> Dùng nhiều khi Debug lỗi
kubectl describe pod <tên-pod>

# Xem log của một Pod (Rất quan trọng)
kubectl logs <tên-pod>
kubectl logs <tên-pod> -f             # Theo dõi trực tiếp (tail feed - cập nhật giống lệnh tail -f ở linux)
kubectl logs <tên-pod> -c <tên-cont>  # Xem log của 1 container cụ thể (nếu Pod có Sidecar)
kubectl logs -p <tên-pod>             # Xem log của pod TRƯỚC KHI nó bị crash/restart

# Chui vào (exec / SSH) trực tiếp bên trong một Pod đang chạy
kubectl exec -it <tên-pod> -- /bin/sh   # Hoặc /bin/bash

# Tạo tạm một chiếc Pod để test mạng (Ví dụ: tạo pod chạy nginx tạm)
kubectl run tmp-pod --image=nginx:alpine --restart=Never

# Xóa một Pod (Pod sẽ bị Terminated rồi mới xoá)
kubectl delete pod <tên-pod>
kubectl delete pod <tên-pod> --force --grace-period=0  # Xóa ép lực (trong trường hợp Pod bị treo ở trạng thái Terminating)
```

---

## 3. 🏗️ Quản lý Deployment & Điểu phối Phiên Bản

```bash
# Xem danh sách Deployment hiện tại
kubectl get deployments

# Tăng giảm số lượng Pod (Scaling)
kubectl scale deployment <tên-deploy> --replicas=5

# Cập nhật phiên bản image mới cho Deployment (Rolling Update)
kubectl set image deployment/<tên-deploy> <tên-container>=<image-mới:tag>

# Xem trạng thái quá trình cập nhật (Rolling Update có đang mượt mà hay đang kẹt)
kubectl rollout status deployment/<tên-deploy>

# Xem lịch sử các lần cập nhật (Để hờ lúc tính rollback)
kubectl rollout history deployment/<tên-deploy>

# Rút lui (Rollback) về phiên bản TRƯỚC ĐÓ ngay lập tức
kubectl rollout undo deployment/<tên-deploy>

# Rollback về một phiên bản/revision cụ thể bằng ID
kubectl rollout undo deployment/<tên-deploy> --to-revision=2
```

---

## 4. 🌐 Quản lý Service & Network

```bash
# Xem danh sách các Service hiện có
kubectl get services
kubectl get svc    # Viết tắt

# Từ bản Deployment đang chạy, tạo nhanh một Service (Expose) NodePort
kubectl expose deployment <tên-deploy> --type=NodePort --port=80 --target-port=8080

# Chuyển tiếp cổng (Port Mapping) từ trạm máy cá nhân của Bạn vào tận trong Pod/Service để Test
kubectl port-forward <tên-pod> 8080:80
kubectl port-forward service/<tên-svc> 8080:80
# Mẹo: Truy cập http://localhost:8080 trên trình duyệt máy bạn, traffic sẽ lặn xuống tận port 80 của Pod. Rất hữu ích khi dev local!
```

---

## 5. ⚙️ Tương tác với File Cấu hình (Declarative YAML)

Đây là cách chuẩn chỉ và phổ biến nhất (Infrastructure as Code) thay vì gõ lệnh trên Terminal.

```bash
# Khởi tạo hoặc cập nhật tài nguyên dựa trên file YAML
kubectl apply -f resource.yaml

# Áp dụng/Chạy toàn bộ các file YAML chứa trong một thư mục
kubectl apply -f ./my-folder/

# Tiêu diệt/Xóa mọi thứ được định nghĩa trong file YAML đó
kubectl delete -f resource.yaml

# Export lại cấu hình tài nguyên đang chạy trên hệ thống ra một file YAML mới (để lưu trữ lại)
kubectl get pod <tên-pod> -o yaml > save-pod-config.yaml
```

---

## 6. 🚨 Troubleshooting & Debugging Nâng Cao

```bash
# Xem tài nguyên CPU / RAM tiêu thụ của các Worker Node
kubectl top nodes

# Xem tài nguyên CPU / RAM tiêu thụ của các Pod 
kubectl top pods

# Xem các loại sự kiện (Events) mới nhất trong namespace để phát hiện lỗi Cảnh báo (Warning)
kubectl get events --sort-by='.metadata.creationTimestamp'
```

---

## 7. 💡 Mẹo & Kỹ thuật "Khung Xương" (Dry-run Template)

Thay vì luôn phải học thuộc lòng rồi hì hục gõ một file YAML từ số không, bạn có thể "nhờ" K8s in ra bộ khung giả định với thông số cơ bản bằng cơ chế `--dry-run`:

```bash
# Generate file yaml mẫu cho Pod mà KHÔNG hề ghi hay khởi tạo Pod trên hệ thống
kubectl run my-pod --image=nginx --dry-run=client -o yaml > templates-pod.yaml

# Generate file yaml mẫu cho Deployment
kubectl create deployment my-deploy --image=nginx --dry-run=client -o yaml > deploy.yaml

# Generate Service YAML xịn xò
kubectl expose deployment my-deploy --port=80 --target-port=8080 --dry-run=client -o yaml > svc.yaml
```
