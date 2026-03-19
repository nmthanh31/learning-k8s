# Kiến Trúc Cluster (Cluster Architecture)

Bài viết này cung cấp cái nhìn tổng quan về kiến trúc cluster Kubernetes, chi tiết hóa vai trò và các thành phần của Master Node và Worker Node trong việc quản lý các ứng dụng container hóa.

Kubernetes đơn giản hóa việc triển khai, mở rộng và quản lý các ứng dụng container thông qua tự động hóa. Để giải thích khái niệm này, hãy tưởng tượng hai loại tàu: tàu chở hàng (Worker Nodes) mang theo các container, và tàu chỉ huy (Master Nodes) giám sát và quản lý các tàu chở hàng. Trong Kubernetes, một cluster bao gồm các node—có thể là máy vật lý hoặc máy ảo, tại chỗ (on-premises) hoặc trên đám mây (cloud)—nơi lưu trữ các ứng dụng container của bạn.

![alt text](https://drek4537l1klr.cloudfront.net/luksa/Figures/11fig01.png)

## Các Thành Phần Của Master Node

Master node chứa một số thành phần của bảng điều khiển (Control Plane) giúp quản lý toàn bộ cluster Kubernetes. Nó theo dõi tất cả các node, quyết định nơi ứng dụng sẽ chạy và liên tục giám sát cluster. Hãy coi Master Node như trung tâm điều huy trung ương điều phối cả đội tàu.

Trong một bến cảng bận rộn, nhiều container được xếp dỡ hàng ngày. Kubernetes duy trì thông tin chi tiết về từng container và node tương ứng của nó trong một kho lưu trữ key-value có tính sẵn sàng cao gọi là etcd. Etcd sử dụng định dạng key-value đơn giản cùng với cơ chế bầu chọn (quorum), đảm bảo lưu trữ dữ liệu tin cậy và nhất quán trên toàn bộ cluster.

Khi một container mới (hoặc "hàng hóa trên tàu") đã sẵn sàng, Kubernetes Scheduler—tương tự như các cần cẩu tại cảng—sẽ xác định Worker Node (hoặc "tàu") nào nên tiếp nhận nó. Scheduler xem xét tải hiện tại, yêu cầu tài nguyên và các ràng buộc cụ thể như taints, tolerations hoặc các quy tắc node affinity. Quá trình lập lịch này là tối quan trọng để vận hành cluster hiệu quả.

Mẹo: Các bộ điều khiển (Replication Controller) và các controller khác hoạt động giống như nhân viên văn phòng bến tàu, đảm bảo số lượng container mong muốn đang chạy và quản lý các hoạt động của node.

Các thành phần chính khác của Master Node bao gồm:

- ETCD Cluster: Lưu trữ cấu hình và dữ liệu trạng thái trên toàn bộ cluster.

- Kube Scheduler: Xác định node tốt nhất cho các lần triển khai container mới.

- Controllers: Quản lý vòng đời của node, sao chép container và sự ổn định của hệ thống.

- Kube API Server: Đóng vai trò là trung tâm liên lạc và quản lý của toàn bộ cluster.

## Các Thành Phần Của Worker Node

Các Worker Node, có thể so sánh với tàu chở hàng, chịu trách nhiệm chạy các ứng dụng container. Mỗi node được quản lý bởi Kubelet, "thuyền trưởng" của node, người đảm bảo các container đang chạy theo đúng chỉ dẫn.

- Kubelet: Quản lý vòng đời container trên từng node riêng lẻ. Nó nhận chỉ thị từ Kube API Server để tạo, cập nhật hoặc xóa các container, và thường xuyên báo cáo trạng thái của node.

- Kube Proxy: Cấu hình các quy tắc mạng trên các Worker Node, nhờ đó cho phép giao tiếp thông suốt giữa các container trên các node khác nhau. Ví dụ, nó cho phép một web server trên node này tương tác với cơ sở dữ liệu trên node khác.

Lưu ý: Toàn bộ hệ thống điều khiển đều được container hóa. Dù bạn sử dụng Docker, Containerd hay CRI-O, mọi node (bao gồm cả master node với các thành phần container hóa) đều yêu cầu một công cụ thực thi container (Container Runtime Engine) tương thích.

Kiến trúc Worker Node cấp cao đảm bảo các ứng dụng luôn sẵn sàng và phản hồi tốt, ngay cả khi chúng giao tiếp qua một mạng lưới phân tán.
