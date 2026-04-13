1. Scale về 0 (khuyên dùng)
kubectl scale deployment -n neuvector --all --replicas=0
kubectl scale daemonset -n neuvector --all --replicas=0

👉 Kết quả:

Enforcer biến mất → không còn chặn traffic
Cluster network trở lại bình thường
Cách 2: Xoá hẳn (mạnh tay)
kubectl delete ns neuvector

👉 Chỉ dùng khi:

Bạn chưa cần security
Muốn môi trường “clean” để học
2. BẬT LẠI NeuVector

Nếu bạn dùng cách scale:

kubectl scale deployment -n neuvector --all --replicas=1
kubectl scale daemonset -n neuvector --all --replicas=1
Quan trọng

Sau khi bật lại:

Nó sẽ quay lại mode Protect → lại chặn tiếp

→ nên phải xử lý rule (phần dưới)

kubectl patch ns monitoring -p '{"metadata":{"annotations":{"neuvector.io/policy-mode":"Discover"}}}'
