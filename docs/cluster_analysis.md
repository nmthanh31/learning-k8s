# Phân tích Chi tiết Hiện trạng Cluster Kubernetes (VNPOST)

Dựa trên các hình ảnh được cung cấp từ hệ thống, dưới đây là báo cáo phân tích chi tiết về hiện trạng của Kubernetes Cluster.

## 1. Thông tin Node & Topology (`nodes-wides.png`)

![nodes-wides.png](/imgs/nodes-wides.png)
- **Phiên bản Kubernetes**: v1.33.6+rke2r1 (sử dụng bản phân phối RKE2).
- **Hệ điều hành**: Ubuntu 24.04.3 LTS (Kernel 6.8.0-90-generic).
- **Container Runtime**: containerd://2.1.5-k3s1.
- **Topology (6 Nodes)**:
  - **Control Plane**: 3 nodes (`cp01`, `cp02`, `cp03`) có role `control-plane,etcd,master`. Các node này có dải IP nội bộ là `172.18.0.51 - 172.18.0.53`. 
    - **Cơ chế cấu hình HA**: Cluster được thiết kế theo mô hình High Availability (HA) chuẩn của phân phối RKE2. Database `etcd` được triển khai dạng nhúng (embedded) và chạy trực tiếp phân tán trên cả 3 master nodes để thiết lập Quorum (cơ chế bầu cử đồng thuận, chống split-brain). API Server cũng được khởi chạy song song để luân chuyển lưu lượng xử lý.
  - **Worker**: 3 nodes (`wk01`, `wk02`, `wk03`) có vai trò chạy các workload. Dải IP `172.18.0.61 - 172.18.0.63`.
    - **Cấu hình Runtime**: Các node được định cấu hình bằng `containerd` làm Runtime mặc định. Việc sử dụng RKE2 kết hợp containerd cho thấy hệ thống rất chú trọng vào tính gọn nhẹ và bảo mật ở tầng host.
  - **Trạng thái**: Tất cả các node đều đang ở trạng thái `Ready`. Tuổi đời khoảng 49 ngày.

## 2. Workload & Core Components (`pods.png`, `pods-2.png`, `pods-ns-kube-system.png`)
Cluster đang vận hành các thành phần hạ tầng cốt lõi và ứng dụng sau:

![pods-ns-kube-system.png](/imgs/pods-ns-kube-system.png)

### Hạ tầng và Core K8s (`kube-system`):
- **BGP / CNI / Mạng**: Sử dụng **Cilium** làm CNI chính (`cilium`, `cilium-envoy`, `cilium-operator`). Có pod `hubble-peer` phục vụ cho việc quan sát mạng.
- **DNS**: `rke2-coredns` đang chạy tốt.
- **Lưu trữ tĩnh (CSI)**: Sử dụng **Openstack Cinder CSI** (`openstack-cinder-csi-controllerplugin` và các `nodeplugin`) để giao tiếp với hệ thống lưu trữ bên dưới (OpenStack).
- **HA Control Plane (VIP)**: Có sử dụng **kube-vip** (chạy dạng DaemonSet - `kube-vip-ds`) nhằm cung cấp Virtual IP ảo hoặc Load Balancer cho Kubernetes Control Plane đảm bảo tính sẵn sàng cao (HA). Đáng chú ý: một số pod của `kube-vip` đang có lượng restarts khá cao (khoảng 290 - 307 lần), điều này cần theo dõi thêm.
- **Workload Controllers**: API Server, Scheduler, Controller Manager (bản phân phối của RKE2) đều Running ổn định. `rke2-snapshot-controller` có hiện tượng restart ~158 lần.

![pods.png](/imgs/pods.png)

![pods-2.png](/imgs/pods-2.png)

### Các Ứng dụng & Dịch vụ (Workloads):
- **ELK Stack (`elk`)**:
  - Gồm 3 nodes Elasticsearch master (`elasticsearch-master-0`, `1`, `2`) và 1 Kibana pod.
- **Keycloak & Database (`keycloak`)**:
  - 2 instances của Keycloak (`keycloak-0`, `keycloak-1`).
  - 3 instances của PostgreSQL dành riêng cho Keycloak (`psql-keycloak-1`, `2`, `3`).
- **Cloud Native PG (`cnpg-system`)**: Dùng toán tử Cloud Native PostgreSQL (`cnpg-controller-manager`) để quản trị cụm PostgreSQL DB.
- **Snipe-IT (`snipe-it`)**: Dựa trên thông tin PersistentVolume có thể suy ra hệ thống đang host dịch vụ Snipe-IT (Quản lý tài sản) bao gồm cơ sở dữ liệu (`datadir-snipe-it-db-cluster-0`, `1`, `2`).
- **MySQL Operator (`mysql-operator`)**: Đang chạy 1 pod MySQL Operator trên namespace `mysql-operator`.
- **Ingress (`nginx-ingress`)**: Cài đặt nginx-ingress controller ở namespace `nginx-ingress` (chạy 1 pod nginx-ingress).
- **Sonobuoy (`sonobuoy`)**: Công cụ chẩn đoán cluster (khả năng dùng để đánh giá tính tuân thủ tiêu chuẩn của cluster k8s) đang chạy 1 aggregator pod.

## 3. Storage & Volumes (`pv.png`, `storageclass.png`)



![storageclass.png](/imgs/storageclass.png)

- **Cấu hình StorageClass (SC)**: Cụm K8s kết nối với hạ tầng OpenStack bên dưới thông qua CSI plugin, cung cấp 2 SC chính:
  - `csi-cinder-sc-delete`: 
    - **Cấu hình ReclaimPolicy: Delete**. Khi một ứng dụng bị gỡ bỏ (kéo theo xóa PVC), volume đĩa trên OpenStack sẽ bị định cấu hình tự động xóa để giải phóng tài nguyên lưu trữ. 
    - Ứng dụng: Phù hợp cho các Workload tạm thời hoặc có kiến trúc dữ liệu tự phục hồi (Điển hình như cụm phân tán của ELK Elasticsearch).
  - `csi-cinder-sc-retain`: 
    - **Cấu hình ReclaimPolicy: Retain**. Khi ứng dụng bị xóa đi, ổ đĩa vật lý trên Cinder Kénh OpenStack vẫn được chốt giữ lại. Cần thao tác can thiệp thủ công từ quản trị viên để giải phóng.
    - Ứng dụng: Đây là cấu hình mang tính an toàn cứng cho các Database cốt lõi (như PostgreSQL của Keycloak và Snipe-IT), phòng tránh rủi ro mất mát dữ liệu do lỡ tay gỡ bỏ Pod/PVC.
  - Cả 2 SC này đều có tham số **VolumeBindingMode: Immediate** (Cấp phát tạo mới ổ đĩa ngay khi có PVC gửi yêu cầu, không cần chờ lịch làm việc của Pod) và **AllowVolumeExpansion: true** (Cho phép quản trị viên resize dung lượng PV trực tiếp trên OpenStack mà không phải downtime ứng dụng).

![pv.png](/imgs/pv.png)
- **Persistent Volumes (PV)**: Cấp phát thành công và đang ở trạng thái Bound:
  - 3 PVs dung lượng **20Gi** (SC `retain`) cấp cho cụm cơ sở dữ liệu của `Snipe-IT` (`datadir-snipe-it-db-cluster-0` đến `2`).
  - 3 PVs dung lượng **30Gi** (SC `delete`) cấp phát cho 3 instance tĩnh của `Elasticsearch`.
  - 3 PVs dung lượng **20Gi** (SC `retain`) cấp cho DB `PostgreSQL` của `Keycloak` (`psql-keycloak-1` đến `3`).

## 4. Ngữ cảnh Mạng, Ingress & Services (`svc.png`, `ingress.png`)

### Cấu hình Hạ tầng Network cốt lõi
- **Kiến trúc CNI (Container Network Interface)**: Dựa vào danh sách pod `kube-system`, có thể thấy hệ thống K8s không dùng mạng Calico/Canal cơ bản mà được tinh chỉnh sử dụng **Cilium**. Cấu hình Cilium (sử dụng công nghệ eBPF chạy trực tiếp ở cấp độ Kernel của Linux) đem lại hiệu năng định tuyến cực cao, khả năng chặn lọc gói tin xuất sắc (NetworkPolicies) và khả năng quan sát hệ thống vi mô (thông qua service con `hubble-peer` đi kèm).
- **Cấu hình Control Plane VIP**: Triển khai giải pháp **kube-vip** (DaemonSet `kube-vip-ds`). Cấu hình này giúp móc nối IP của 3 Master Node, tạo ra 1 địa chỉ Virtual IP (VIP) ảo nổi trên toàn cluster thông qua giao thức ARP / BGP. Nếu 1 master node bị sập đột ngột, VIP sẽ tự động "nhảy" sang master node còn sống khác trong tích tắc, cung cấp khả năng chịu lỗi (Fault-tolerance) hoàn hảo cho API Server nội bộ mà không cần phụ thuộc Load Balancer bên ngoài.

![svc.png](/imgs/svc.png)

### Services
- Tất cả các dịch vụ nội bộ (Core DNS, Metrics Server, DB, CNI) đều đang dùng loại `ClusterIP`. Trừ một số DB dạng phân tán hoặc Discovery Service có cấu hình thêm dạng headless (`None`) như `elasticsearch-master-headless`, `keycloak-discovery`, `cilium-envoy`.
- **Nginx Ingress Service**: Expose ra ngoài bằng `NodePort`, lắng nghe tại port 80 (chuyển sang `32053/TCP`), và 443 (chuyển sang `31489/TCP`). IP nội bộ của Node sẽ được sử dụng kèm 2 cổng trên để truy cập lớp Load Balancer vào Ingress.

![ingress.png](/imgs/ingress.png)

### Ingress
Cấu trúc rules theo host trên Ingress Controllers (đều chỉ định Ingress Class là `nginx`):
- `kibana.vnpost.cloud`: trỏ routing tới dịch vụ `kibana-kibana` ở namespace `elk`.
- `elasticsearch.vnpost.cloud`: trỏ routing tới dịch vụ `elasticsearch-master` ở namespace `elk`.
- `iam.vnpost.cloud`: trỏ routing tới hệ thống phân quyền/định danh `keycloak` ở namespace `keycloak`.

## 5. Các Vấn đề Cần Lưu ý (Troubleshooting Alerts)
Mặc dù cluster đang ở trạng thái `Ready` và các dịch vụ đều báo `Running`, có vài điểm có khả năng tiềm ẩn rủi ro hoặc biểu hiện của sự thiếu ổn định trước đây (thông qua số lượng Restarts):
1. **`kube-vip-ds`**: Các pod daemonset thực thi IP tĩnh cho control plane đã khởi động lại khá nhiều (khoảng 300 lần). Cần xem lại lịch sử hệ thống có các tình trạng đứt đoạn kết nối mạng, rớt ARP/BGP session nội bộ nào ở layer vật lí gây crash hoặc rớt VIP hay không.
2. **`openstack-cinder-csi-controllerplugin`**: Có ~628 restarts trong 49 ngày hoạt động. Điều này ám chỉ kết nối CSI driver đến API của hạ tầng OpenStack bên dưới có thời điểm bị gián đoạn, hoặc timeout dẫn tới liveness probe failed. Cần check log của Controller này.
3. **`cnpg-controller-manager`**: Restart 76 lần. Nên theo dõi logs của toán tử này để tránh rủi ro quản lí HA cho database PostgreSQL.
## 6. Phân tích chi tiết Tài nguyên và Workload theo Node

Dựa trên kết quả `kubectl describe node` mới được tra cứu, cấu hình chi tiết phân bổ tài nguyên và ứng dụng trên từng máy chủ (Node) được bộc lộ rõ:

### A. Nhóm Control Plane (cp01, cp02, cp03)
**Đặc tả chung tài nguyên mỗi Master Node:**
- **CPU:** 4 Cores (Allocatable: 3.5 Cores)
- **RAM:** ~8GB (Allocatable: ~7.3GB)
- **Đặc điểm chung**: Cả 3 master node đều được Taint với nhãn `node-role.kubernetes.io/control-plane=true:NoSchedule`, tức là **chống lập lịch ứng dụng thường** lên cấu hình này. Chỉ những Pod hệ thống thiết yếu (như CNI, CSI, Ingress Controllers đặc thù nếu có tolerations) mới được phép vào.

**Tình trạng phân bổ Workloads trên Master Nodes:**
Cả 3 máy chủ master chạy đối xứng (Symmetric) các thành phần K8s cốt lõi:
- **Core K8s Components**: `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `etcd`.
- **Hạ tầng Mạng**: DaemonSet của `cilium` và `kube-vip-ds`.
- **Hạ tầng Storage**: Kênh kết nối `openstack-cinder-csi-nodeplugin` có mặt trên cả 3 node để mount ổ cứng nếu lỡ cần thiết.
- **DNS**: Sự phân bổ DNS linh hoạt. Node `cp01` và `cp02` đồng gánh chịu `rke2-coredns` pods. 
- **Đánh giá tải**: Các master node rất nhẹ. Tải CPU request dao động ở mức 24% - 27%, RAM requests ở mức 27% - 29%. Hệ thống master dư sức mở rộng quản lí thêm hàng chục node worker nữa mà không lo nghẽn API.

---

### B. Nhóm Worker Nodes (wk01, wk02, wk03)
**Đặc tả chung tài nguyên mỗi Worker Node:**
- **CPU:** 8 Cores (Allocatable: 7.6 Cores)
- **RAM:** ~16GB (Allocatable: ~15GB)

**Tình trạng phân bổ Workloads mang ý nghĩa nghiệp vụ (với sai lệch cực lớn giữa các node):**

1. **Node `wk01` (Hạt nhân Gánh Tải - Nặng Nhất):**
   - **Tải Requests (Chiệm dụng trước):** CPU (50%), RAM (54%) - Rất cao.
   - **Tải Limits (Có nguy cơ chạm nóc):** CPU (80%), RAM (69%) - Có khả năng bị khứa CPU (throttling) hoặc bị OOMKilled nếu có spike lượng truy cập.
   - **Ứng dụng đóng đô:** Node này đang gánh hàng loạt con **"Quái vật" Database/Cluster**: 
     - 1 pod Elasticsearch Master (`elasticsearch-master-1`).
     - 2 pod PostgreSQL của Keycloak (`psql-keycloak-1`, `psql-keycloak-2`).
     - Toán tử PostgreSQL (`cnpg-controller-manager`).
     - Node chính phục vụ Keycloak IAM (`keycloak-0`).

2. **Node `wk02` (Hạt Nhân Gánh Tải - Tầm Trung):**
   - **Tải Requests:** CPU (42%), RAM (46%).
   - **Ứng dụng đóng đô:** Chia lửa với `wk01`, bao gồm:
     - 1 pod Elasticsearch Master (`elasticsearch-master-0`) và Kibana (`kibana-kibana-894d6648-qstwd`).
     - 1 pod Keycloak IAM phụ trợ (`keycloak-1`) và 1 node PostgreSQL Keycloak còn lại (`psql-keycloak-3`).
     - **Nginx Ingress Controller** (`nginx-ingress-69cb797b4d-jvljc`) đón toàn bộ network traffic từ ngoài vào K8s.
     - MySQL Operator.

3. **Node `wk03` (Rảnh Rỗi Quá Mức):**
   - **Tải Requests:** CPU (1%), RAM (~0%). Node gần như "nằm chơi".
   - **Ứng dụng đóng đô:** Không có ứng dụng nghiệp vụ nào nặng nề chạy ở đây, ngoại trừ một pod chẩn đoán `sonobuoy` đang nằm ngủ, và vài pod cắm rễ mạng (`cilium-sjdhc`, `cilium-envoy`).

**Nhận định và Khuyến nghị cho K8s Scheduler:**
- Sự chênh lệch thảm hoạ giữa `wk01`, `wk02` (quá tải DB/Web) và `wk03` (nằm chơi) là do K8s Scheduler mặc định đang ưu tiên gom vào node nào đó, hoặc các tài nguyên PV (Pod chạy gắn với PVC thuộc AZ1 Openstack bị kẹt vị trí vật lý).
- **Tuy nhiên**, để ý Pod `elasticsearch-master-2` hiện đang chạy ở `wk01` cùng với `master-1`. Điều này dẫn đến nguy cơ: Nếu Node `wk01` sập, thì **2 trên tổng số 3** node `elasticsearch` đi đời, cụm Elasticsearch sẽ mất Quorum và bị sập hoàn toàn (do luật phải quá bán = 2 node sống mới bầu cử được).
- **-> Cần triển khai cấu hình `PodAntiAffinity` (Chống dính Pod)** cho các Deployment/StatefulSet Core như Elasticsearch, PostgreSQL, và Keycloak để bắt buộc Kube-scheduler phải tản đều 3 Pods ra đều 3 máy `wk01`, `wk02`, `wk03`. Đặc biệt là Elasticsearch Master.

---
### Tổng Kết
Hệ thống là một cluster Kubernetes ở mức **Production-ready** dựa trên định dạng **RKE2** (phiên bản K8s mới), với kiến trúc High Availability (3 master / 3 worker). Cấu trúc cluster hiện đại khi sử dụng Cilium CNI, OpenStack Cinder dự phòng Storage, VIP Controller, và quản lý Database phân tán bằng các thư viện Operator như CloudNativePG/MySQL Operator. 


<details>
<summary>Logs: Chi tiết lệnh describe nodes (Bấm để mở rộng)</summary>

```bash
kubectl get nodes
NAME   STATUS   ROLES                       AGE   VERSION
cp01   Ready    control-plane,etcd,master   49d   v1.33.6+rke2r1
cp02   Ready    control-plane,etcd,master   49d   v1.33.6+rke2r1
cp03   Ready    control-plane,etcd,master   49d   v1.33.6+rke2r1
wk01   Ready    <none>                      49d   v1.33.6+rke2r1
wk02   Ready    <none>                      49d   v1.33.6+rke2r1
wk03   Ready    <none>                      49d   v1.33.6+rke2r1
root@vm1:~# kubectl describe node cp01
Name:               cp01
Roles:              control-plane,etcd,master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=cp01
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/control-plane=true
                    node-role.kubernetes.io/etcd=true
                    node-role.kubernetes.io/master=true
                    topology.cinder.csi.openstack.org/zone=AZ1
Annotations:        csi.volume.kubernetes.io/nodeid: {"cinder.csi.openstack.org":"ab153774-9a2b-420d-ba3f-a54c935946b0"}
                    etcd.rke2.cattle.io/local-snapshots-timestamp: 2026-03-17T09:05:40Z
                    etcd.rke2.cattle.io/node-address: 172.18.0.51
                    etcd.rke2.cattle.io/node-name: cp01-e23811f3
                    node.alpha.kubernetes.io/ttl: 0
                    rke2.io/encryption-config-hash: start-fe58e2c6f41f7f532302c9e1e1e74ae0ed4fe8fbe2915925a91bf2e03ad9db4b
                    rke2.io/node-args:
                      ["server","--token","********","--tls-san","172.18.0.50","--tls-san","k8s.vnpost.cloud","--tls-san","cluster.local","--tls-san-security","...
                    rke2.io/node-config-hash: QEHPHJKYF3IIULIGE2QMQ3YYDFUD3PQXMUCHSHPY3IDOIL4YYESA====
                    rke2.io/node-env: {}
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Tue, 27 Jan 2026 02:38:54 +0000
Taints:             node-role.kubernetes.io/control-plane=true:NoSchedule
Unschedulable:      false
Lease:
  HolderIdentity:  cp01
  AcquireTime:     <unset>
  RenewTime:       Tue, 17 Mar 2026 09:12:22 +0000
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Tue, 27 Jan 2026 02:41:07 +0000   Tue, 27 Jan 2026 02:41:07 +0000   CiliumIsUp                   Cilium is running on this node
  EtcdIsVoter          True    Tue, 17 Mar 2026 09:10:06 +0000   Sun, 15 Feb 2026 09:56:21 +0000   MemberNotLearner             Node is a voting member of the etcd cluster
  MemoryPressure       False   Tue, 17 Mar 2026 09:09:31 +0000   Tue, 27 Jan 2026 02:38:53 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Tue, 17 Mar 2026 09:09:31 +0000   Tue, 27 Jan 2026 02:38:53 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Tue, 17 Mar 2026 09:09:31 +0000   Tue, 27 Jan 2026 02:38:53 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Tue, 17 Mar 2026 09:09:31 +0000   Tue, 27 Jan 2026 02:41:06 +0000   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  172.18.0.51
  Hostname:    cp01
Capacity:
  cpu:                4
  ephemeral-storage:  29378688Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             8131816Ki
  pods:               110
Allocatable:
  cpu:                3500m
  ephemeral-storage:  231853216
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             7373759687
  pods:               110
System Info:
  Machine ID:                 1a88e6e1b5dd52a16c0ca499cb995877
  System UUID:                ab153774-9a2b-420d-ba3f-a54c935946b0
  Boot ID:                    809b5c97-5545-4505-bf5e-0d803ab0f5f3
  Kernel Version:             6.8.0-90-generic
  OS Image:                   Ubuntu 24.04.3 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://2.1.5-k3s1
  Kubelet Version:            v1.33.6+rke2r1
  Kube-Proxy Version:         
PodCIDR:                      10.100.0.0/24
PodCIDRs:                     10.100.0.0/24
Non-terminated Pods:          (11 in total)
  Namespace                   Name                                                     CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                                     ------------  ----------  ---------------  -------------  ---
  kube-system                 cilium-4qqbf                                             100m (2%)     0 (0%)      10Mi (0%)        0 (0%)         49d
  kube-system                 cilium-envoy-wdvgz                                       0 (0%)        0 (0%)      0 (0%)           0 (0%)         49d
  kube-system                 cilium-operator-669f5bbc66-gbh49                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         49d
  kube-system                 etcd-cp01                                                200m (5%)     0 (0%)      512Mi (7%)       0 (0%)         49d
  kube-system                 kube-apiserver-cp01                                      250m (7%)     0 (0%)      1Gi (14%)        0 (0%)         35d
  kube-system                 kube-controller-manager-cp01                             200m (5%)     0 (0%)      256Mi (3%)       0 (0%)         49d
  kube-system                 kube-scheduler-cp01                                      100m (2%)     0 (0%)      128Mi (1%)       0 (0%)         49d
  kube-system                 kube-vip-ds-6j2k4                                        0 (0%)        0 (0%)      0 (0%)           0 (0%)         49d
  kube-system                 openstack-cinder-csi-nodeplugin-8nltk                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         49d
  kube-system                 rke2-coredns-rke2-coredns-85d6696775-865ss               100m (2%)     100m (2%)   128Mi (1%)       128Mi (1%)     49d
  kube-system                 rke2-coredns-rke2-coredns-autoscaler-665b7f6f86-fqpjt    25m (0%)      100m (2%)   16Mi (0%)        64Mi (0%)      49d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests      Limits
  --------           --------      ------
  cpu                975m (27%)    200m (5%)
  memory             2074Mi (29%)  192Mi (2%)
  ephemeral-storage  0 (0%)        0 (0%)
  hugepages-1Gi      0 (0%)        0 (0%)
  hugepages-2Mi      0 (0%)        0 (0%)
Events:              <none>
```

```bash
kubectl describe node cp02
Name:               cp02
Roles:              control-plane,etcd,master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=cp02
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/control-plane=true
                    node-role.kubernetes.io/etcd=true
                    node-role.kubernetes.io/master=true
                    topology.cinder.csi.openstack.org/zone=AZ1
Annotations:        csi.volume.kubernetes.io/nodeid: {"cinder.csi.openstack.org":"eb0a398e-0de0-45bf-a7a6-3ca30959eb0a"}
                    etcd.rke2.cattle.io/local-snapshots-timestamp: 2026-03-17T09:12:36Z
                    etcd.rke2.cattle.io/node-address: 172.18.0.52
                    etcd.rke2.cattle.io/node-name: cp02-3ca985c1
                    node.alpha.kubernetes.io/ttl: 0
                    rke2.io/encryption-config-hash: start-fe58e2c6f41f7f532302c9e1e1e74ae0ed4fe8fbe2915925a91bf2e03ad9db4b
                    rke2.io/node-args:
                      ["server","--server","https://172.18.0.50:9345","--token","********","--tls-san","172.18.0.50","--tls-san","k8s.vnpost.cloud","--tls-san",...
                    rke2.io/node-config-hash: DRBZ5WVSEQKOKUY6S6TEMDABTMLHIXDCKGBYV3N4MFEKLFMHJNTQ====
                    rke2.io/node-env: {}
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Tue, 27 Jan 2026 02:43:34 +0000
Taints:             node-role.kubernetes.io/control-plane=true:NoSchedule
Unschedulable:      false
Lease:
  HolderIdentity:  cp02
  AcquireTime:     <unset>
  RenewTime:       Tue, 17 Mar 2026 09:15:54 +0000
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Tue, 27 Jan 2026 02:44:08 +0000   Tue, 27 Jan 2026 02:44:08 +0000   CiliumIsUp                   Cilium is running on this node
  EtcdIsVoter          True    Tue, 17 Mar 2026 09:13:51 +0000   Tue, 17 Mar 2026 01:23:19 +0000   MemberNotLearner             Node is a voting member of the etcd cluster
  MemoryPressure       False   Tue, 17 Mar 2026 09:15:21 +0000   Tue, 27 Jan 2026 02:43:33 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Tue, 17 Mar 2026 09:15:21 +0000   Tue, 27 Jan 2026 02:43:33 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Tue, 17 Mar 2026 09:15:21 +0000   Tue, 27 Jan 2026 02:43:33 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Tue, 17 Mar 2026 09:15:21 +0000   Tue, 27 Jan 2026 02:44:04 +0000   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  172.18.0.52
  Hostname:    cp02
Capacity:
  cpu:                4
  ephemeral-storage:  29378688Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             8131808Ki
  pods:               110
Allocatable:
  cpu:                3500m
  ephemeral-storage:  231853216
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             7373751905
  pods:               110
System Info:
  Machine ID:                 1a88e6e1b5dd52a16c0ca499cb995877
  System UUID:                eb0a398e-0de0-45bf-a7a6-3ca30959eb0a
  Boot ID:                    6097be49-baab-4e91-8154-c45e68ac395a
  Kernel Version:             6.8.0-90-generic
  OS Image:                   Ubuntu 24.04.3 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://2.1.5-k3s1
  Kubelet Version:            v1.33.6+rke2r1
  Kube-Proxy Version:         
PodCIDR:                      10.100.1.0/24
PodCIDRs:                     10.100.1.0/24
Non-terminated Pods:          (9 in total)
  Namespace                   Name                                          CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                          ------------  ----------  ---------------  -------------  ---
  kube-system                 cilium-envoy-hvs7w                            0 (0%)        0 (0%)      0 (0%)           0 (0%)         49d
  kube-system                 cilium-p9q9t                                  100m (2%)     0 (0%)      10Mi (0%)        0 (0%)         49d
  kube-system                 etcd-cp02                                     200m (5%)     0 (0%)      512Mi (7%)       0 (0%)         49d
  kube-system                 kube-apiserver-cp02                           250m (7%)     0 (0%)      1Gi (14%)        0 (0%)         35d
  kube-system                 kube-controller-manager-cp02                  200m (5%)     0 (0%)      256Mi (3%)       0 (0%)         49d
  kube-system                 kube-scheduler-cp02                           100m (2%)     0 (0%)      128Mi (1%)       0 (0%)         49d
  kube-system                 kube-vip-ds-4mvfl                             0 (0%)        0 (0%)      0 (0%)           0 (0%)         49d
  kube-system                 openstack-cinder-csi-nodeplugin-2pmkt         0 (0%)        0 (0%)      0 (0%)           0 (0%)         49d
  kube-system                 rke2-coredns-rke2-coredns-85d6696775-jcgrc    100m (2%)     100m (2%)   128Mi (1%)       128Mi (1%)     49d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests      Limits
  --------           --------      ------
  cpu                950m (27%)    100m (2%)
  memory             2058Mi (29%)  128Mi (1%)
  ephemeral-storage  0 (0%)        0 (0%)
  hugepages-1Gi      0 (0%)        0 (0%)
  hugepages-2Mi      0 (0%)        0 (0%)
Events:              <none>
```

```bash
kubectl describe node cp03
Name:               cp03
Roles:              control-plane,etcd,master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=cp03
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/control-plane=true
                    node-role.kubernetes.io/etcd=true
                    node-role.kubernetes.io/master=true
                    topology.cinder.csi.openstack.org/zone=AZ1
Annotations:        csi.volume.kubernetes.io/nodeid: {"cinder.csi.openstack.org":"781b7364-0f01-4183-80be-96fa43ca452f"}
                    etcd.rke2.cattle.io/local-snapshots-timestamp: 2026-03-17T09:11:28Z
                    etcd.rke2.cattle.io/node-address: 172.18.0.53
                    etcd.rke2.cattle.io/node-name: cp03-49bfd34c
                    node.alpha.kubernetes.io/ttl: 0
                    rke2.io/encryption-config-hash: start-fe58e2c6f41f7f532302c9e1e1e74ae0ed4fe8fbe2915925a91bf2e03ad9db4b
                    rke2.io/node-args:
                      ["server","--server","https://172.18.0.50:9345","--token","********","--tls-san","172.18.0.50","--tls-san","k8s.vnpost.cloud","--tls-san",...
                    rke2.io/node-config-hash: DRBZ5WVSEQKOKUY6S6TEMDABTMLHIXDCKGBYV3N4MFEKLFMHJNTQ====
                    rke2.io/node-env: {}
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Tue, 27 Jan 2026 02:44:17 +0000
Taints:             node-role.kubernetes.io/control-plane=true:NoSchedule
Unschedulable:      false
Lease:
  HolderIdentity:  cp03
  AcquireTime:     <unset>
  RenewTime:       Tue, 17 Mar 2026 09:17:06 +0000
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Tue, 27 Jan 2026 02:44:51 +0000   Tue, 27 Jan 2026 02:44:51 +0000   CiliumIsUp                   Cilium is running on this node
  EtcdIsVoter          True    Tue, 17 Mar 2026 09:13:36 +0000   Tue, 27 Jan 2026 02:44:26 +0000   MemberNotLearner             Node is a voting member of the etcd cluster
  MemoryPressure       False   Tue, 17 Mar 2026 09:16:52 +0000   Tue, 27 Jan 2026 02:44:17 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Tue, 17 Mar 2026 09:16:52 +0000   Tue, 27 Jan 2026 02:44:17 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Tue, 17 Mar 2026 09:16:52 +0000   Tue, 27 Jan 2026 02:44:17 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Tue, 17 Mar 2026 09:16:52 +0000   Tue, 27 Jan 2026 02:44:49 +0000   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  172.18.0.53
  Hostname:    cp03
Capacity:
  cpu:                4
  ephemeral-storage:  29378688Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             8131808Ki
  pods:               110
Allocatable:
  cpu:                3500m
  ephemeral-storage:  231853216
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             7373751905
  pods:               110
System Info:
  Machine ID:                 1a88e6e1b5dd52a16c0ca499cb995877
  System UUID:                781b7364-0f01-4183-80be-96fa43ca452f
  Boot ID:                    60884f3f-7ac1-4548-ad33-849b9f46e1eb
  Kernel Version:             6.8.0-90-generic
  OS Image:                   Ubuntu 24.04.3 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://2.1.5-k3s1
  Kubelet Version:            v1.33.6+rke2r1
  Kube-Proxy Version:         
PodCIDR:                      10.100.2.0/24
PodCIDRs:                     10.100.2.0/24
Non-terminated Pods:          (8 in total)
  Namespace                   Name                                     CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                     ------------  ----------  ---------------  -------------  ---
  kube-system                 cilium-envoy-6cbrv                       0 (0%)        0 (0%)      0 (0%)           0 (0%)         49d
  kube-system                 cilium-ftr8g                             100m (2%)     0 (0%)      10Mi (0%)        0 (0%)         49d
  kube-system                 etcd-cp03                                200m (5%)     0 (0%)      512Mi (7%)       0 (0%)         49d
  kube-system                 kube-apiserver-cp03                      250m (7%)     0 (0%)      1Gi (14%)        0 (0%)         35d
  kube-system                 kube-controller-manager-cp03             200m (5%)     0 (0%)      256Mi (3%)       0 (0%)         49d
  kube-system                 kube-scheduler-cp03                      100m (2%)     0 (0%)      128Mi (1%)       0 (0%)         49d
  kube-system                 kube-vip-ds-frtgv                        0 (0%)        0 (0%)      0 (0%)           0 (0%)         49d
  kube-system                 openstack-cinder-csi-nodeplugin-slf8x    0 (0%)        0 (0%)      0 (0%)           0 (0%)         49d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests      Limits
  --------           --------      ------
  cpu                850m (24%)    0 (0%)
  memory             1930Mi (27%)  0 (0%)
  ephemeral-storage  0 (0%)        0 (0%)
  hugepages-1Gi      0 (0%)        0 (0%)
  hugepages-2Mi      0 (0%)        0 (0%)
Events:              <none>
```

```bash
kubectl describe node wk01
Name:               wk01
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=wk01
                    kubernetes.io/os=linux
                    topology.cinder.csi.openstack.org/zone=AZ1
Annotations:        csi.volume.kubernetes.io/nodeid: {"cinder.csi.openstack.org":"ca6fe25f-4d7b-4c8f-894a-d06cdfedeb2d"}
                    node.alpha.kubernetes.io/ttl: 0
                    rke2.io/node-args:
                      ["agent","--token","********","--server","https://172.18.0.50:9345","--protect-kernel-defaults","true","--resolv-conf","/etc/resolv.conf",...
                    rke2.io/node-config-hash: N4PDJNODTP6PNX4O4VQM43ZSHYYUK6I76FPH3NWP777J7W3UQTJQ====
                    rke2.io/node-env: {}
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Tue, 27 Jan 2026 02:59:46 +0000
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  wk01
  AcquireTime:     <unset>
  RenewTime:       Tue, 17 Mar 2026 09:20:20 +0000
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Tue, 27 Jan 2026 03:00:27 +0000   Tue, 27 Jan 2026 03:00:27 +0000   CiliumIsUp                   Cilium is running on this node
  MemoryPressure       False   Tue, 17 Mar 2026 09:18:22 +0000   Tue, 27 Jan 2026 02:59:46 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Tue, 17 Mar 2026 09:18:22 +0000   Tue, 27 Jan 2026 02:59:46 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Tue, 17 Mar 2026 09:18:22 +0000   Tue, 27 Jan 2026 02:59:46 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Tue, 17 Mar 2026 09:18:22 +0000   Tue, 27 Jan 2026 03:00:22 +0000   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  172.18.0.61
  Hostname:    wk01
Capacity:
  cpu:                8
  ephemeral-storage:  49691512Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             16375648Ki
  pods:               250
Allocatable:
  cpu:                7600m
  ephemeral-storage:  32233775476
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             15327072Ki
  pods:               250
System Info:
  Machine ID:                 1a88e6e1b5dd52a16c0ca499cb995877
  System UUID:                ca6fe25f-4d7b-4c8f-894a-d06cdfedeb2d
  Boot ID:                    5dd9fafd-c088-4321-8bfd-3228d31ef8de
  Kernel Version:             6.8.0-90-generic
  OS Image:                   Ubuntu 24.04.3 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://2.1.5-k3s1
  Kubelet Version:            v1.33.6+rke2r1
  Kube-Proxy Version:         
PodCIDR:                      10.100.4.0/24
PodCIDRs:                     10.100.4.0/24
Non-terminated Pods:          (11 in total)
  Namespace                   Name                                         CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                         ------------  ----------  ---------------  -------------  ---
  cnpg-system                 cnpg-controller-manager-7d4d7f5854-5br77     100m (1%)     100m (1%)   100Mi (0%)       200Mi (1%)     49d
  elk                         elasticsearch-master-1                       1 (13%)       1 (13%)     2Gi (13%)        2Gi (13%)      35d
  elk                         elasticsearch-master-2                       1 (13%)       1 (13%)     2Gi (13%)        2Gi (13%)      37d
  keycloak                    keycloak-0                                   500m (6%)     2 (26%)     1700Mi (11%)     2000Mi (13%)   49d
  keycloak                    psql-keycloak-1                              500m (6%)     1 (13%)     1Gi (6%)         2Gi (13%)      35d
  keycloak                    psql-keycloak-2                              500m (6%)     1 (13%)     1Gi (6%)         2Gi (13%)      49d
  kube-system                 cilium-7dqkx                                 100m (1%)     0 (0%)      10Mi (0%)        0 (0%)         49d
  kube-system                 cilium-envoy-c5c2d                           0 (0%)        0 (0%)      0 (0%)           0 (0%)         49d
  kube-system                 openstack-cinder-csi-nodeplugin-7r8k9        0 (0%)        0 (0%)      0 (0%)           0 (0%)         49d
  kube-system                 rke2-metrics-server-7c4c577547-6m22k         100m (1%)     0 (0%)      200Mi (1%)       0 (0%)         35d
  kube-system                 rke2-snapshot-controller-696989ffdd-dlldh    0 (0%)        0 (0%)      0 (0%)           0 (0%)         49d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests      Limits
  --------           --------      ------
  cpu                3800m (50%)   6100m (80%)
  memory             8154Mi (54%)  10392Mi (69%)
  ephemeral-storage  0 (0%)        0 (0%)
  hugepages-1Gi      0 (0%)        0 (0%)
  hugepages-2Mi      0 (0%)        0 (0%)
Events:              <none>
```

```bash
kubectl describe node wk02
Name:               wk02
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=wk02
                    kubernetes.io/os=linux
                    topology.cinder.csi.openstack.org/zone=AZ1
Annotations:        csi.volume.kubernetes.io/nodeid: {"cinder.csi.openstack.org":"5a2e1b4d-8357-4648-b1f6-c0c88d898cb9"}
                    node.alpha.kubernetes.io/ttl: 0
                    rke2.io/node-args:
                      ["agent","--token","********","--server","https://172.18.0.50:9345","--protect-kernel-defaults","true","--resolv-conf","/etc/resolv.conf",...
                    rke2.io/node-config-hash: N4PDJNODTP6PNX4O4VQM43ZSHYYUK6I76FPH3NWP777J7W3UQTJQ====
                    rke2.io/node-env: {}
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Tue, 27 Jan 2026 02:59:44 +0000
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  wk02
  AcquireTime:     <unset>
  RenewTime:       Tue, 17 Mar 2026 09:25:50 +0000
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Tue, 27 Jan 2026 03:00:23 +0000   Tue, 27 Jan 2026 03:00:23 +0000   CiliumIsUp                   Cilium is running on this node
  MemoryPressure       False   Tue, 17 Mar 2026 09:23:59 +0000   Tue, 27 Jan 2026 02:59:44 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Tue, 17 Mar 2026 09:23:59 +0000   Tue, 27 Jan 2026 02:59:44 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Tue, 17 Mar 2026 09:23:59 +0000   Tue, 27 Jan 2026 02:59:44 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Tue, 17 Mar 2026 09:23:59 +0000   Tue, 27 Jan 2026 03:00:18 +0000   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  172.18.0.62
  Hostname:    wk02
Capacity:
  cpu:                8
  ephemeral-storage:  49691512Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             16375644Ki
  pods:               250
Allocatable:
  cpu:                7600m
  ephemeral-storage:  32233775476
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             15327068Ki
  pods:               250
System Info:
  Machine ID:                 1a88e6e1b5dd52a16c0ca499cb995877
  System UUID:                5a2e1b4d-8357-4648-b1f6-c0c88d898cb9
  Boot ID:                    2b7e8f1d-d06f-42f0-9157-53e6801b92ab
  Kernel Version:             6.8.0-90-generic
  OS Image:                   Ubuntu 24.04.3 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://2.1.5-k3s1
  Kubelet Version:            v1.33.6+rke2r1
  Kube-Proxy Version:         
PodCIDR:                      10.100.3.0/24
PodCIDRs:                     10.100.3.0/24
Non-terminated Pods:          (10 in total)
  Namespace                   Name                                                      CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                                      ------------  ----------  ---------------  -------------  ---
  elk                         elasticsearch-master-0                                    1 (13%)       1 (13%)     2Gi (13%)        2Gi (13%)      37d
  elk                         kibana-kibana-894d6648-qstwd                              1 (13%)       1 (13%)     2Gi (13%)        2Gi (13%)      37d
  keycloak                    keycloak-1                                                500m (6%)     2 (26%)     1700Mi (11%)     2000Mi (13%)   35d
  keycloak                    psql-keycloak-3                                           500m (6%)     1 (13%)     1Gi (6%)         2Gi (13%)      49d
  kube-system                 cilium-envoy-9nrhb                                        0 (0%)        0 (0%)      0 (0%)           0 (0%)         49d
  kube-system                 cilium-zh4zh                                              100m (1%)     0 (0%)      10Mi (0%)        0 (0%)         49d
  kube-system                 openstack-cinder-csi-controllerplugin-746d568bb4-8gc7p    0 (0%)        0 (0%)      0 (0%)           0 (0%)         49d
  kube-system                 openstack-cinder-csi-nodeplugin-rqlqv                     0 (0%)        0 (0%)      0 (0%)           0 (0%)         49d
  mysql-operator              mysql-operator-868f798d97-qjld7                           0 (0%)        0 (0%)      0 (0%)           0 (0%)         43d
  nginx-ingress               nginx-ingress-69cb797b4d-jvljc                            100m (1%)     0 (0%)      128Mi (0%)       0 (0%)         38d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests      Limits
  --------           --------      ------
  cpu                3200m (42%)   5 (65%)
  memory             6958Mi (46%)  8144Mi (54%)
  ephemeral-storage  0 (0%)        0 (0%)
  hugepages-1Gi      0 (0%)        0 (0%)
  hugepages-2Mi      0 (0%)        0 (0%)
Events:              <none>
```

```bash
kubectl describe node wk03
Name:               wk03
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=wk03
                    kubernetes.io/os=linux
                    topology.cinder.csi.openstack.org/zone=AZ1
Annotations:        csi.volume.kubernetes.io/nodeid: {"cinder.csi.openstack.org":"2b43a2f6-ebb4-4069-b12b-a023806147c5"}
                    node.alpha.kubernetes.io/ttl: 0
                    rke2.io/node-args:
                      ["agent","--token","********","--server","https://172.18.0.50:9345","--protect-kernel-defaults","true","--resolv-conf","/etc/resolv.conf",...
                    rke2.io/node-config-hash: N4PDJNODTP6PNX4O4VQM43ZSHYYUK6I76FPH3NWP777J7W3UQTJQ====
                    rke2.io/node-env: {}
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Tue, 27 Jan 2026 02:59:47 +0000
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  wk03
  AcquireTime:     <unset>
  RenewTime:       Tue, 17 Mar 2026 09:27:11 +0000
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Tue, 27 Jan 2026 03:00:26 +0000   Tue, 27 Jan 2026 03:00:26 +0000   CiliumIsUp                   Cilium is running on this node
  MemoryPressure       False   Tue, 17 Mar 2026 09:24:16 +0000   Tue, 27 Jan 2026 02:59:47 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Tue, 17 Mar 2026 09:24:16 +0000   Tue, 27 Jan 2026 02:59:47 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Tue, 17 Mar 2026 09:24:16 +0000   Tue, 27 Jan 2026 02:59:47 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Tue, 17 Mar 2026 09:24:16 +0000   Tue, 27 Jan 2026 03:00:22 +0000   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  172.18.0.63
  Hostname:    wk03
Capacity:
  cpu:                8
  ephemeral-storage:  49691512Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             16375640Ki
  pods:               250
Allocatable:
  cpu:                7600m
  ephemeral-storage:  32233775476
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             15327064Ki
  pods:               250
System Info:
  Machine ID:                 1a88e6e1b5dd52a16c0ca499cb995877
  System UUID:                2b43a2f6-ebb4-4069-b12b-a023806147c5
  Boot ID:                    d8fadafd-2519-46f3-ac47-06a4d9c6553d
  Kernel Version:             6.8.0-90-generic
  OS Image:                   Ubuntu 24.04.3 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://2.1.5-k3s1
  Kubelet Version:            v1.33.6+rke2r1
  Kube-Proxy Version:         
PodCIDR:                      10.100.5.0/24
PodCIDRs:                     10.100.5.0/24
Non-terminated Pods:          (4 in total)
  Namespace                   Name                                     CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                     ------------  ----------  ---------------  -------------  ---
  kube-system                 cilium-envoy-dp5fl                       0 (0%)        0 (0%)      0 (0%)           0 (0%)         49d
  kube-system                 cilium-sjdhc                             100m (1%)     0 (0%)      10Mi (0%)        0 (0%)         49d
  kube-system                 openstack-cinder-csi-nodeplugin-2v65w    0 (0%)        0 (0%)      0 (0%)           0 (0%)         49d
  sonobuoy                    sonobuoy                                 0 (0%)        0 (0%)      0 (0%)           0 (0%)         35d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests   Limits
  --------           --------   ------
  cpu                100m (1%)  0 (0%)
  memory             10Mi (0%)  0 (0%)
  ephemeral-storage  0 (0%)     0 (0%)
  hugepages-1Gi      0 (0%)     0 (0%)
  hugepages-2Mi      0 (0%)     0 (0%)
Events:              <none>
```</details>
