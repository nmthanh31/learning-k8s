# Bài 3: Khám phá mớ mạng bùng nhùng (Networking, Service & Ingress)

Ở bài 1 và 2, ta đã có các Pod ứng dụng chạy ổn định. Nay phát sinh một bài toán: **Ứng dụng Web (Frontend) muốn gọi API đến Ứng dụng Backend**.

## Vấn đề IP "phù du"
Mỗi Pod có một địa chỉ IP nội bộ sinh ra ngẫu nhiên.
1. Tuy nhiên Pod hay chết. Lúc nó phục sinh bởi Deployment, địa chỉ **IP cũ MẤT, IP mới XUẤT HIỆN**. 
2. Chưa kể nếu ta chạy 3 bản sao Backend (3 Pods backend), Frontend nên gọi số IP của thằng số 1, số 2 hay số 3?

Giải pháp: Đừng bao giờ hardcode dựa vào IP của Pod. K8s đã tạo ra một bức tường ổn định mang tên **Service**.

---

## 1. Service là gì?
Service như một "Trung tâm tổng đài". Tổng đài có 1 Số điện thoại (IP tĩnh) đứng chờ khách gọi, sau đó nó tự phân phối cuộc gọi về cho các trạm viên (các Pods). Lỡ một trạm viên có nghỉ việc và thay người mới thì Số tổng đài vẫn y chang.

Service định vị được các Pod thông qua **Labels** (nhãn). Pod nào dán giấy "app: backend" thì tự động trở thành lính của nhóm Service quản lý "backend".

Có 4 loại Service chính mà bạn phải nhớ:

### A. ClusterIP (Mặc định - Chỉ Dùng Nội Bộ)
- Service tạo ra một IP Tĩnh chỉ có thể giao tiếp **Bên trong Cụm**.
- VD: Website không thể bị hack nếu nó không public ra internet. Đặt Database theo dạng ClusterIP. Frontend gọi tên Service của DB (VD: `http://db-service:3306`), K8s sẽ tự động chuyển hướng nó cho Pod bên trong.

### B. NodePort (Mở cửa ra thế giới)
- Xây dựng trên nền ClusterIP, nhma nó "đục" thêm một cổng vào CÁC MÁY CHỦ VẬT LÝ (Nodes).
- Cổng này nằm trong khoảng `30000 - 32767`.
- Bạn có thể vào trình duyệt web gõ `<Bất_Kỳ_IP_Của_Node_Nào>:<Số_NodePort>` -> K8s sẽ bắt traffic và đưa tới Service -> Đưa đến Pod.
- *Nhược điểm*: Khi public web, người vào thấy số port lạ lùng đằng sau IP sẽ sợ. Nên thường đây là môi trường test.

### C. LoadBalancer (Định tuyến đám mây)
- Xây dựng trên nền NodePort, gọi API cho Cung cấp đám mây (Cloud Provider như AWS, GCP hay OpenStack) để thuê một con LoadBalancer vật lý/ảo hóa có IP Public chuẩn. IP đó trỏ thẳng vào NodePort.
- *Ưu điểm:* Người dùng truy cập IP Public mượt mà. Đăng nhập web qua cổng 80/443 không cần nhớ Port.
- *Nhược điểm:* Đắt đỏ. Cứ có 1 dịch vụ web là phải tốn tiền gọi cấp 1 con LB Cloud ngoài đời.

### D. Headless Service
- Dành cho các hệ thống ngáo (ví dụ Database Statefulset) lười qua trung gian.
- Bạn set ClusterIP thành `None`. Service không lấy IP nào cả, nó chỉ cung cấp tính năng phân giải DNS, để các Pod biết IP gốc trực tiếp của nhau mà gọi.

---

## 2. Ingress - Người kiểm soát giao thông tối cao

Bạn có 2 websites: `A.com` (Frontend 1) và `B.com` (Frontend 2).
Sẽ ra sao nếu tạo 2 LoadBalancer tốn đống tiền? Hay trỏ thẳng cả 2 bằng NodePort rồi bắt người ta nhớ `<IP>:30001` cho web A và `<IP>:30002` cho webB?

Không. Chúng ta dùng **Ingress**.

**Ingress** cung cấp khả năng gom (Routing) theo Domain (tên miền) và Path (đường dẫn), tất cả qua 1 cánh cửa.

Quy trình:
1. Bạn tạo ra 1 con Nginx Ingress Controller (Một server Nginx thực thụ chạy trong Pod K8s, được Expose dạng NodePort hoặc duy nhất 1 con Loadbalancer để bắt traffic cổng 80/443 do thiên hạ đập vào).
2. Lập trình viên viết file Ingress Rules miêu tả như sau:
   - "Nếu mày thấy ai gọi Tên Miền là `A.com/api` -> Truyền nó qua Service tên là `Backend-ClusterIP`"
   - "Nếu ai gọi Tên miền `B.com/` -> Truyền qua Service `Frontend_2_ClusterIP`".
3. Lập trình viên trỏ DNS A Record của `A.com` và `B.com` về địa chỉ Nginx Controller đó dẹp lép bài toán Routing.

**Tổng kết**:
- Muốn nối Pod với Pod -> Dùng **ClusterIP Service**.
- Muốn nối internet thẳng vào App cho rẻ -> **NodePort**.
- Muốn có 1 cục Loadbalancer vật lý xịn xò -> Dùng **LoadBalancer Service**.
- Muốn quản trị nhiều Tên Miền / Cấu hình HTTP TLS cao cấp -> Dùng **Ingress** (sau đó trỏ Traffic về các ClusterIP Service tương ứng).
