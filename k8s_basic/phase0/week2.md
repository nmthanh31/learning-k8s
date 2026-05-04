# Phase 0 - Week 2: Workload, Network, Storage cơ bản - Cách mọi thứ kết nối?

<a name="top"></a>
## 📑 Mục lục
- [1. Workloads là gì?](#1-workloads-là-gì)
    - [1.1 Phân loại Workloads](#11-phân-loại-workloads)
    - [1.2 Tại sao cần Deployment?](#12-tại-sao-cần-deployment)
    - [1.3 Cách thức hoạt động của Deployment](#13-cách-thức-hoạt-động-của-deployment)
- [2. Label và Selector (Sợi dây liên kết)](#2-label-và-selector)
- [3. Network cơ bản (Service)](#3-network-cơ-bản-service)

---

## 1. Workloads là gì?
Trong K8s, **Workload** là một ứng dụng chạy trên container. Thay vì quản lý từng Pod riêng lẻ (rất vất vả và dễ sai sót), chúng ta sử dụng các đối tượng Workload để K8s tự động quản lý vòng đời, số lượng và trạng thái của ứng dụng.

### 1.1 Phân loại Workloads
Tùy vào tính chất của ứng dụng mà ta chọn loại Workload phù hợp:

| Loại Workload | Đặc điểm | Ví dụ thực tế |
| :--- | :--- | :--- |
| **Deployment** | Ứng dụng không lưu trạng thái (**Stateless**). Dễ dàng scale up/down. | Web App, API, Nginx. |
| **StatefulSet** | Ứng dụng có lưu trạng thái (**Stateful**). Pod có định danh cố định. | Database (MySQL, MongoDB), Redis. |
| **DaemonSet** | Chạy duy nhất 1 Pod trên **mỗi Node** trong cluster. | Log Collector (Fluentd), Monitoring Agent. |
| **Job / CronJob** | Chạy xong rồi dừng (Job) hoặc chạy định kỳ theo thời gian (CronJob). | Backup dữ liệu, Gửi email báo cáo. |

[⬆ Quay lại đầu trang](#top)

---

### 1.2 Tại sao cần Deployment?
Nhiều người thắc mắc: *"ReplicaSet đã giúp tự động hồi sinh Pod rồi, tại sao cần thêm Deployment?"*

**Câu trả lời ngắn gọn:** ReplicaSet quản lý **số lượng**, còn Deployment quản lý **phiên bản (version)**.

*   **Nếu chỉ dùng ReplicaSet:** Khi bạn muốn cập nhật code mới (đổi Image), bạn phải xóa ReplicaSet cũ và tạo cái mới thủ công $\rightarrow$ Gây gián đoạn ứng dụng (**Downtime**).
*   **Khi dùng Deployment:** Nó sẽ tự động điều phối việc thay thế ReplicaSet cũ bằng ReplicaSet mới mà không gây downtime (**Rolling Update**).

---

### 1.3 Cách thức hoạt động của Deployment

#### 1.3.1 Cơ chế Tự chữa lành (Self-healing)
Khi một Pod bị "chết" hoặc bị xóa nhầm:
1.  **ReplicaSet** (do Deployment quản lý) phát hiện: *Số thực tế < Số mong muốn*.
2.  **ReplicaSet** lập tức tạo Pod mới để bù đắp.
3.  **Deployment** đóng vai trò "giám sát cấp cao", nó không trực tiếp tạo Pod mà ra lệnh cho ReplicaSet làm việc đó.

#### 1.3.2 Cơ chế Cập nhật (Rolling Update)
Đây là "phép màu" của Deployment:
1.  Bạn cập nhật Image mới cho Deployment.
2.  Deployment tạo ra một **ReplicaSet Mới** (V2).
3.  **V2** tạo dần 1 Pod mới $\rightarrow$ **V1** (cũ) xóa dần 1 Pod cũ.
4.  Quá trình tiếp diễn cho đến khi toàn bộ Pod là của **V2**.
5.  **Lưu ý:** ReplicaSet cũ (V1) không bị xóa hẳn, nó chỉ đưa số lượng Pod về **0**. Điều này giúp bạn **Rollback** (quay xe) về phiên bản cũ cực nhanh nếu bản mới bị lỗi.

[⬆ Quay lại đầu trang](#top)

---

## 2. Label và Selector (Sợi dây liên kết)

Nếu Pod là "công nhân", thì Label và Selector chính là cách để "quản đốc" tìm đúng nhóm công nhân của mình.

### 2.1 Label (Nhãn dán)
Là các cặp **Key-Value** được gắn vào Pod (hoặc bất kỳ tài nguyên nào). Bạn có thể gắn bao nhiêu nhãn tùy thích.
*   **Ví dụ:** `app: nginx`, `env: production`, `version: v1`.
*   **Mục đích:** Để phân loại và định danh. K8s không quan tâm tên Pod là gì, nó chỉ nhìn vào nhãn.

### 2.2 Selector (Bộ lọc)
Là một "bộ lọc" dùng để tìm kiếm các tài nguyên có nhãn khớp với yêu cầu.
*   **Ví dụ:** "Hãy tìm cho tôi tất cả các Pod có nhãn `app: nginx`".

### 2.3 Tại sao chúng lại quan trọng? (Cơ chế Loose Coupling)
Trong K8s, các Controller (như ReplicaSet) không quản lý Pod theo tên. Chúng liên tục quét cluster bằng Selector để tìm "đàn con" của mình.

**Kịch bản thực tế:**
Giả sử bạn có 1 Deployment yêu cầu `replicas: 3` với selector `app: nginx`.
1.  **Nếu chưa có Pod nào:** ReplicaSet sẽ tạo mới 3 Pod có gắn nhãn `app: nginx`.
2.  **Nếu bỗng dưng có ai đó tạo lẻ 1 Pod** có nhãn `app: nginx` bên ngoài: ReplicaSet sẽ quét thấy và "nhận" luôn Pod đó làm con. Nó thấy tổng cộng đã có 1 Pod, nên nó chỉ tạo thêm 2 Pod nữa cho đủ 3.
3.  **Nếu bạn gỡ nhãn `app: nginx` khỏi 1 Pod đang chạy:** ReplicaSet sẽ ngay lập tức coi như Pod đó đã biến mất (vì không tìm thấy qua selector nữa) và nó sẽ tạo 1 Pod mới để thay thế ngay lập tức.

> [!IMPORTANT]
> **Lưu ý cực kỳ quan trọng:** Trong file YAML của Deployment, nhãn trong phần `spec.selector.matchLabels` **PHẢI KHỚP HÀN TOÀN** với nhãn trong phần `spec.template.metadata.labels`. Nếu không, Deployment sẽ tạo ra Pod nhưng không bao giờ tìm thấy chúng, dẫn đến việc tạo Pod vô tận!

[⬆ Quay lại đầu trang](#top)

---

## 3. Network cơ bản (Service)

Pod trong K8s rất "mỏng manh" và IP của nó sẽ thay đổi mỗi khi khởi động lại. Để các ứng dụng giao tiếp được với nhau ổn định, chúng ta cần **Service**.

| Loại Service | Mục đích sử dụng |
| :--- | :--- |
| **ClusterIP** | Tạo một IP nội bộ cố định. Chỉ các dịch vụ **trong cluster** mới gọi được nhau. (Mặc định) |
| **NodePort** | Mở một cổng (30000-32767) trên toàn bộ các Node. Cho phép truy cập từ **ngoài vào** qua IP của Node. |
| **LoadBalancer** | Tự động tạo một bộ cân bằng tải trên Cloud (AWS/GCP/Azure) để đưa ứng dụng ra Internet. |

---
[⬆ Quay lại đầu trang](#top)
