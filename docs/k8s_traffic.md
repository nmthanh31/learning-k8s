# BÁO CÁO PHÂN TÍCH KIẾN TRÚC LƯU LƯỢNG MẠNG (SOUTH–NORTH & EAST–WEST)

## 1. Tổng quan

Trong kiến trúc Kubernetes hiện đại, lưu lượng mạng được chia thành hai trục chính: **South–North** (vào/ra cụm) và **East–West** (nội bộ giữa các dịch vụ). Việc hiểu rõ các lớp xử lý từ Load Balancer đến tận NIC và Pod thông qua eBPF là chìa khóa để tối ưu hiệu năng và bảo mật.

### Sơ đồ luồng lưu lượng (Traffic Flow)
![traffic-flow](/imgs/traffic-flow.png)

---

## 2. Chi tiết về South–North Traffic (Lưu lượng Nam–Bắc)

### 2.1. Định nghĩa
Lưu lượng South–North đại diện cho các kết nối từ người dùng (User) bên ngoài đi vào cụm hoặc các yêu cầu quản trị hệ thống. Trong sơ đồ, luồng này di chuyển theo chiều dọc.

### 2.2. Luồng di chuyển (Data Flow) theo sơ đồ:
**A. Incoming (Chiều vào):**
1.  **Người dùng (User)** kết nối tới điểm truy cập duy nhất.
2.  **LB/VIP (Load Balancer/Virtual IP)**: Tiếp nhận request.
    -   **Quản trị**: `kubectl` kết nối tới **Control Plane** (để quản lý cụm và các node).
    -   **Ứng dụng**: Traffic đi tới **Service/NodePort** của Worker Node.
3.  **Xử lý tại Worker Node**:
    -   **Ingress**: Định tuyến dựa trên host/path.
    -   **Service/ClusterIP**: Cân bằng tải nội bộ.
    -   **Pod**: Xử lý logic nghiệp vụ.
    -   **Cilium/eBPF**: Lớp mạng kernel-level xử lý gói tin với hiệu suất cực cao và áp dụng chính sách bảo mật (Network Policy).
    -   **NIC (Network Interface Card)**: Chuyển gói tin ra/vào lớp vật lý.

**B. Outgoing/Egress (Chiều ra):**
1.  **Pod** → **Cilium/eBPF** → **NIC**.
2.  **Gateway**: Tiếp nhận traffic từ các Worker Node.
3.  **Internet**: Điểm đến cuối cùng của các yêu cầu ra bên ngoài.

### 2.3. Trọng tâm quản lý
-   **Security**: WAF, SSL Termination tại Ingress.
-   **Routing**: Điều hướng dựa trên Domain/Path tại Layer 7.

---

## 3. Chi tiết về East–West Traffic (Lưu lượng Đông–Tây)

### 3.1. Định nghĩa
Lưu lượng East–West là giao tiếp giữa các Microservices bên trong cụm (Internal Cluster). Theo sơ đồ, đây là luồng di chuyển theo chiều ngang giữa các Worker Node.

### 3.2. Luồng di chuyển (Data Flow):
Khi một Pod ở Worker Node A cần giao tiếp với Pod ở Worker Node B:
1.  **Pod (A)** → **Cilium/eBPF** (V-Switch kernel xử lý).
2.  **NIC (Node A)** đóng gói gói tin vào **Overlay - VXLAN**.
3.  Traffic di chuyển qua hạ tầng mạng vật lý tới **NIC (Node B)**.
4.  **NIC (Node B)** giải mã VXLAN → **Cilium/eBPF** (Kiểm tra bảo mật/mTLS).
5.  **Pod (B)** tiếp nhận dữ liệu.

### 3.3. Overlay Network (VXLAN)
Trong sơ đồ, lớp **Overlay - VXLAN** bao phủ các NIC của các Worker Node, tạo ra một mạng phẳng ảo (Virtual Flat Network) giúp các Pod có thể giao tiếp với nhau dù nằm ở các máy chủ vật lý khác nhau mà không cần cấu hình phức tạp tại lớp mạng router/switch vật lý.

### 3.4. Công nghệ chủ chốt: Cilium & eBPF
-   **eBPF**: Cho phép lập trình trực tiếp vào Linux Kernel để xử lý gói tin mà không cần qua lớp `iptables` chậm chạp. Trong hình, Cilium/eBPF đóng vai trò là "cửa ngõ" kernel ngay trước khi traffic đi vào Pod hoặc ra NIC.
-   **Cilium**: Cung cấp khả năng quan sát (Observability), bảo mật mạng (Network Policy) và cân bằng tải hiệu năng cao cho giao tiếp nội bộ.

---

## 4. Kết luận
Kiến trúc trong sơ đồ cho thấy sự kết hợp chặt chẽ giữa các thành phần truyền thống (LB, Ingress) và công nghệ hiện đại (Cilium/eBPF). Việc South-North traffic đi qua eBPF trước khi ra NIC đảm bảo rằng mọi chính sách bảo mật đều được áp dụng ngay tại tầng sâu nhất của hệ điều hành.
