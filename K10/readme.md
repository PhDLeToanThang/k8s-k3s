# Phần 1. Khảo sát mô hình K8S:

>>>Mô hình kịch bản: Tôi có 1 hệ thống K8S chạy ở 2 môi trường: Prod thì có 1 master Ubuntu 24.04 LTS + 3 node làm Slave  thuộc nhóm K8S chạy VM ubuntu 24.04 LTS trên nền vSphere 8.u03 và làm vSphere Storage CNS mount từ 2 LUN của SAN physical khoảng 7TB - 14TB; và 1 cụm Dev cũng có cấu hình 1 master Ubuntu 24.04 LTS + 3 node làm slave thuộc nhóm K8s chạy VM ubuntu 24.04 LTS trên nền vSphere 8.0u3 và làm vSphere CNS mount từ 2 LUN của SAN physical khoảng < 1 TB để chạy thử nghiệm; ngoài ra có 4 minio Object Storage S3 chạy trên Oracle linux làm 1 cụm cluster Object có kết nối làm BLOB/CLOB S3;  
Hãy phân tích mô hình xác định cấu hình tiêu chuẩn đáp ứng về Veeam K10 Kasten, Licensing, cấu hình VMs + DB cho K10 và cấu hình Proxy, Network bandwitch, IOPS, Storage và Proxy cho Backup/Restore/Replicate hoặc SureBackup/Sure Restore để giúp tôi triển khai Kasten 10 backup 2 cụm K8s trên?

---

Dưới đây là phân tích chi tiết và xác định cấu hình tiêu chuẩn nhằm triển khai giải pháp Veeam Kasten K10 để bảo vệ toàn diện cho 2 cụm Kubernetes (Prod và Dev) theo đúng mô hình hạ tầng hiện tại của bạn.
------------------------------
## 1. Mô hình Kiến trúc & Thành phần Kasten K10
Veeam Kasten K10 hoạt động theo cơ chế Kubernetes-native (chạy trực tiếp bên trong cụm K8s dưới dạng một Namespace riêng biệt). Đối với hạ tầng vSphere 8.0u3 kết hợp cụm MinIO Object Storage, kiến trúc Kasten sẽ vận hành như sau: [1] 

* Giai đoạn Snapshot (Local): K10 gọi API của vSphere CSI Driver (Cloud Native Storage - CNS) để ra lệnh cho hệ thống SAN Physical thực hiện snapshot nhanh ở tầng phần cứng (First Class Disk - FCD). [2, 3] 
* Giai đoạn Export (Backup sang S3): K10 sử dụng các worker pod tạm thời (Data Movers) để đọc dữ liệu từ block snapshot của vSphere CNS, nén/mã hóa và đẩy trực tiếp sang cụm MinIO Cluster S3 qua giao thức Object Storage làm kho lưu trữ lâu dài (Long-term Retention). [4, 5, 6] 

------------------------------
## 2. Chính sách Cấp phép (Licensing)
Veeam Kasten tính bản quyền dựa trên số lượng Worker Nodes (Slave) thực tế được bảo vệ trong cụm K8s. [7] 

* Cụm Dev (3 Slave Nodes): Có thể sử dụng phiên bản Kasten K10 Free Edition (Miễn phí hoàn toàn cho cụm dưới 5 nodes) đầy đủ tính năng backup/restore sang MinIO. [8] 
* Cụm Prod (3 Slave Nodes): Sử dụng bản quyền Kasten Enterprise Edition (Mua thương mại dạng VUL - Veeam Universal License hoặc Kasten Node-based License cho 3 Nodes). [7, 9] 
* Quản lý tập trung: Triển khai tính năng Kasten Multi-Cluster Manager. Cụm Prod đóng vai trò là Primary Cluster nhận License, sau đó tự động phân phối và giám sát chính sách backup cho cả cụm Dev (Secondary) qua một giao diện Dashboard duy nhất. [10] 

------------------------------
## 3. Cấu hình Tài nguyên VM & Database nội bộ cho Kasten K10
Kasten K10 sử dụng các Database nội bộ tích hợp sẵn bên trong kiến trúc Microservices của nó (như Catalog DB lưu metadata, Prometheus lưu số liệu metric, NoSQL cho hệ thống config). Bạn không cần dựng VM DB riêng ngoài, tuy nhiên cần đảm bảo cấu hình tối thiểu cho các Node Slave trong K8s để gánh được các Pod hệ thống này: [11, 12] 

| Thành phần tài nguyên | Mức tối thiểu (Min) | Mức khuyến nghị tiêu chuẩn (Chạy ổn định) |
|---|---|---|
| CPU phục vụ K10 | 4 Cores cấp phát dư (Slack) trên tổng cụm | Cấu hình thêm 1-2 Core vật lý cho mỗi Node Slave |
| RAM phục vụ K10 | 8 GB RAM trên tổng cụm | Cấu hình thêm 4 GB RAM cho mỗi Node Slave |
| Storage lưu trữ Catalog DB | 20 GB Persistent Volume | 100 GB Volume bằng vSphere CNS (SSD/SAN) |
| Ephemeral Storage trên Node | Tối thiểu 20-30 GB tự do | 100 GB không gian trống trên mỗi Ubuntu VM (để tránh lỗi disk-pressure khi xuất/nhập dữ liệu snapshot). |

------------------------------
## 4. Cấu hình Mạng (Network Bandwidth) & Hạ tầng Proxy## Kết nối Mạng & Băng thông (Bandwidth)

* Băng thông kết nối: Do dung lượng cụm Prod khá lớn (7TB – 14TB), mạng kết nối giữa các Node Slave K8s và Cụm MinIO bắt buộc phải đạt tối thiểu 10 Gbps (Khuyến nghị 25 Gbps hoặc cấu hình LACP/Bonding 2x10Gbps).
* Nếu dùng mạng 1 Gbps, việc Export 14TB dữ liệu sang MinIO sẽ gây nghẽn nghiêm trọng và không thể hoàn thành trong khung thời gian backup (Backup Window).
* Phân tách mạng (VLAN): Cần tách biệt Traffic mạng Backup (K8s Nodes <-> MinIO Cluster) độc lập với Traffic chạy ứng dụng (Production/User Traffic) nhằm tránh xung đột băng thông.

## Cơ chế Proxy / Data Mover

* Kasten K10 không sử dụng "VM Proxy" rời như giải pháp Veeam Backup cho máy ảo truyền thống.
* Thay vào đó, khi có lệnh backup, Kasten tự động tạo ra các Pod Data Mover tạm thời (ephemeral pods) nằm ngay trên các Slave Node chứa ứng dụng. Các Pod này trực tiếp mount Snapshot từ vSphere CNS và truyền tải thẳng qua mạng S3 đến MinIO. Do đó, năng lực xử lý (Proxy) sẽ tự động co giãn (scale-out) theo số lượng Slave Node ứng dụng của bạn. [4, 6, 13, 14] 

------------------------------
## 5. Yêu cầu Hiệu năng IOPS & Lưu trữ (Storage Target)## Phía SAN Physical & vSphere CNS (Source)

* IOPS đột biến: Khi Kasten kích hoạt chính sách Backup, hệ thống SAN sẽ phải chịu áp lực nhân đôi: Vừa gánh IOPS đọc/ghi của ứng dụng hiện tại, vừa gánh IOPS đọc dữ liệu tuần tự từ Snapshot để export.
* Yêu cầu: Vùng LUN 7TB-14TB trên SAN của cụm Prod nên nằm trên các phân vùng SSD/NVMe (All-Flash) hoặc Tiering SAS 15K có bộ đệm lớn, đảm bảo khả năng cung cấp tối thiểu 5,000 đến 10,000 IOPS trong giai đoạn chạy backup mà không làm tăng độ trễ (latency) của ứng dụng vượt quá 20ms. [14] 

## Phía cụm 4 Node MinIO S3 Cluster (Target)

* Dung lượng (Capacity): Với 14TB dữ liệu gốc, áp dụng chính sách lưu trữ (Retention) 30 ngày kèm incremental và tính năng nén (Compression) / chống trùng lặp (Deduplication) của Kasten, bạn cần chuẩn bị dung lượng khả dụng trên MinIO từ 20TB đến 25TB cho cụm Prod. Cụm Dev cần khoảng 1.5TB đến 2TB.
* Tốc độ Ghi (Write IOPS): Đảm bảo hạ tầng ổ đĩa trên 4 Node Oracle Linux chạy MinIO có cấu hình RAID (như RAID 60 hoặc MinIO Erasure Coding công thức $N/2$) đạt tốc độ ghi tuần tự ổn định từ 300 MB/s đến 500 MB/s.

------------------------------
## 6. Kiến trúc Kiểm thử Tự động (SureBackup / SureRestore)
Đối với môi trường container Kubernetes, khái niệm SureBackup của máy ảo vSphere được Kasten K10 hiện thực hóa bằng tính năng K10 Disaster Recovery (DR) kết hợp Automated Restore Testing: [11, 15] 

* Tận dụng Cụm Dev làm môi trường Lab: Do bạn có sẵn cụm Dev chạy vSphere 8.0u3 độc lập, cấu hình tiêu chuẩn là cấu hình một chính sách Kasten Replicate Policy. [2] 
* Luồng vận hành:
1. Kasten tại cụm Prod tiến hành Backup ứng dụng lên MinIO Cluster.
   2. Kasten tại cụm Dev kết nối vào cùng Bucket MinIO đó (đọc chung Location Profile).
   3. Định kỳ (hằng tuần/hằng tháng), cụm Dev tự động kéo dữ liệu Metadata và PVC từ MinIO về, tiến hành Restore thử nghiệm thành các Namespace có tiền tố -test hoặc -dr trên cụm Dev.
   4. Kasten ứng dụng các bộ lọc biến đổi (TransformSets) để tự động sửa đổi StorageClass từ Prod sang Dev, thay đổi cấu hình CPU/RAM thấp xuống cho phù hợp tài nguyên cụm Dev (< 1TB).
   5. Sau khi Restore thành công, một đoạn mã script tự động (Health Check Hook) sẽ kiểm tra trạng thái Pod (Status = Running). Nếu đạt, hệ thống gửi thông báo xác nhận trạng thái an toàn dữ liệu qua Slack/Email và tự động xóa (Clean up) Namespace thử nghiệm để giải phóng tài nguyên. [5, 15, 16, 17, 18] 

------------------------------
## 7. Khuyến nghị chuẩn bị trước khi cài đặt (Pre-requisites Checklist)

Để quá trình cài đặt bằng Helm Chart diễn ra thuận lợi, bạn hãy thực hiện các bước chuẩn bị sau:

   1. Cài đặt vSphere CSI Driver: Đảm bảo toàn bộ PVC trên hai cụm K8s đang sử dụng vsphere-csi loại hỗ trợ Volume Snapshot.
   2. Khởi tạo vSphere Infrastructure Profile: Trên giao diện Kasten K10, tạo một hồ sơ hạ tầng cấp quyền tài khoản vCenter (như administrator@vsphere.local) để Kasten có quyền tương tác ra lệnh Snapshot FCD tầng SAN.
   3. Tạo S3 Bucket trên MinIO: Tạo sẵn các Bucket riêng biệt cho Prod và Dev trên cụm MinIO kèm tài khoản Access Key và Secret Key có quyền ghi đọc tuyệt đối. Bật tính năng Object Locking (WORM) trên MinIO nếu bạn muốn kích hoạt khả năng chống Ransomware (Immutable Backups) của Kasten. [2, 3, 8, 15] 

Nếu bạn đã sẵn sàng, hãy cho tôi biết loại ứng dụng / database nào chính (như MySQL, PostgreSQL, hay ứng dụng Web stateless...) đang chạy trên cụm K8s Prod để tôi hỗ trợ cung cấp Mẫu Blueprint (Application Hook) giúp Kasten thực hiện backup đóng băng ứng dụng (Quiesce) một cách toàn vẹn nhất nhé!
```
[1] [https://www.veeam.com](https://www.veeam.com/products/cloud/kubernetes-data-protection.html)
[2] [https://docs.kasten.io](https://docs.kasten.io/8.0.2/install/vmware/vsphere)
[3] [https://helpcenter.veeam.com](https://helpcenter.veeam.com/docs/vbr/kasten_integration/data_recovery.html)
[4] [https://docs.kasten.io](https://docs.kasten.io/latest/operating/footprint/)
[5] [https://community.veeam.com](https://community.veeam.com/onboarding-for-veeam-kasten-for-kubernetes-179/step-4-operating-k10-10545)
[6] [https://vforum.ch](https://vforum.ch/slides/Kasten_K10_by_Veeam_Kubernetes_Native_Data_Management.pdf)
[7] [https://docs.kasten.io](https://docs.kasten.io/8.0.9/install/advanced)
[8] [https://www.veeam.com](https://www.veeam.com/products/free/kubernetes.html)
[9] [https://www.eon.io](https://www.eon.io/blog/veeam-pricing)
[10] [https://docs.kasten.io](https://docs.kasten.io/8.5.7/multicluster/concepts/license/)
[11] [https://geoffburkeblog.com](https://geoffburkeblog.com/2025/09/kasten-disaster-recovery-setup/)
[12] [https://artifacthub.io](https://artifacthub.io/packages/helm/kasten/k10/6.5.1)
[13] [https://forums.veeam.com](https://forums.veeam.com/viewtopic.php?f=2&t=92162&start=0)
[14] [https://community.veeam.com](https://community.veeam.com/veeam-kasten-kubernetes-data-protection-support-92/k10-compute-requirements-2185)
[15] [https://surajblog.medium.com](https://surajblog.medium.com/kasten-k10-disaster-recovery-dr-setup-guide-for-kubernetes-openshift-3a36a360384c)
[16] [https://docs.kasten.io](https://docs.kasten.io/8.0.4/references/best-practices/)
[17] [https://www.youtube.com](https://www.youtube.com/watch?v=ZP33cSRcYPA)
[18] [https://www.veeam.com](https://www.veeam.com/blog/kasten-k10-kubernetes-application-mobility.html)
```
---

# Phần 2. Demo hoặc Hand On Labs K10:

Veeam và Kasten cung cấp đầy đủ các chương trình Hands-On Labs (Môi trường Lab thật 100% miễn phí) và Interactive Demos (Demo tương tác mô phỏng) trực tuyến để bạn trải nghiệm, cấu hình thực tế trước khi triển khai hệ thống thật.
Dưới đây là các kênh chính thức bạn có thể truy cập ngay:

## 1. Hệ thống Veeam Hands-On Labs (Môi trường Lab thực tế)
Đây là môi trường web-based giúp bạn tự thao tác trên một cụm Lab được cấu hình sẵn hạ tầng. Hệ thống sẽ tự động khởi tạo Cluster cho bạn thực hành gõ lệnh và cấu hình trên giao diện K10. [1, 2] 

* **Veeam Hands-On Labs (Tổng hợp)**: Truy cập [Veeam Hands-On-Labs Experience](https://go.veeam.com/hands-on-lab-experience?product=K10) và chọn bộ sản phẩm K10. [2] 

* **Các bài Lab nổi bật**:
* **Test Drive Kasten K10**: Bài lab tổng quan giúp bạn tìm hiểu kiến trúc, cách Kasten khám phá (Discover) ứng dụng và lập chính sách backup.
   * **Kubernetes App Mobility**: Bài lab nâng cao hướng dẫn cách di chuyển ứng dụng (Mobility) từ cụm K8s này sang cụm K8s khác (rất phù hợp để bạn thử nghiệm mô hình đồng bộ từ Prod sang Dev).
   * **Kasten K10 & Red Hat OpenShift / VMware**: Thực hành cấu hình Storage Class, Snapshot Class và thiết lập bảo vệ toàn diện dữ liệu. [3, 4] 

## 2. Kasten K10 Interactive Demos (Demo tương tác nhanh)
Nếu bạn không muốn mất thời gian chờ khởi tạo tài nguyên Lab hoàn chỉnh và chỉ muốn xem nhanh luồng giao diện UI xử lý thế nào, bạn có thể sử dụng trang Demo mô phỏng (Click-through). [5] 

* **Trang mục lục Demo**: Truy cập [Veeam Kasten Demos Catalog](https://veeamkasten.dev/demos/).
* Bài học đề xuất cho mô hình của bạn: Chọn bài [Getting Started with Kasten K10 by Veeam Interactive Demo](https://veeamkasten.dev/demo-getting-started). Nó sẽ hướng dẫn bạn qua 4 bước: Giới thiệu hệ thống, Quét tìm ứng dụng (Discovering), Thiết lập chính sách Backup (Backup Policies) và Các tính năng nâng cao (Advanced Features). [6, 7] 

## 3. Video Walkthrough & Mã nguồn mẫu (Dành cho tự học)

***Kênh Youtube**: Bạn có thể tham khảo playlist [Kasten K10 Product Demos trên YouTube](https://www.youtube.com/playlist?list=PLYAs0sHjyBtmQ7m7DoSsS-b1M3SozJ9Hs) để xem các chuyên gia Veeam demo trực tiếp tính năng Instant Recovery với vSphere FCD và MinIO. [8, 9] 

* **Lộ trình Onboarding**: Cộng đồng Veeam có cung cấp một bài viết hướng dẫn từng bước rất chi tiết từ khâu lập kế hoạch, cài đặt bằng Helm cho đến vận hành nâng cao tại [Onboarding for Veeam Kasten for Kubernetes](https://community.veeam.com/onboarding-for-veeam-kasten-for-kubernetes-179/onboarding-for-veeam-kasten-for-kubernetes-10535). [10] 

------------------------------
**Khuyến nghị thực hiện:** Bạn nên đăng ký ngay một tài khoản miễn phí trên trang Veeam Hands-On Labs để thử tạo chính sách Backup ứng dụng mẫu và xuất (Export) dữ liệu sang một kho lưu trữ Object Storage giả lập. [2] 
Khi bạn làm Lab, nếu gặp khó khăn ở bước cấu hình StorageClass snapshot hoặc kết nối S3 Profile, hãy nhắn tôi để tôi hướng dẫn trực tiếp các câu lệnh (CLI) tương ứng nhé!
```
[1] [https://instruqt.com](https://instruqt.com/blog/demand-gen-hands-on-learning-platform)
[2] [https://go.veeam.com](https://go.veeam.com/hands-on-lab-experience?product=K10)
[3] [https://go.veeam.com](https://go.veeam.com/hands-on-lab-experience?product=K10)
[4] [https://community.veeam.com](https://community.veeam.com/veeam-hands-on-labs-118/new-kasten-hands-on-labs-9562)
[5] [https://www.veeam.com](https://www.veeam.com/demo-kubernetes.html)
[6] [https://veeamkasten.dev](https://veeamkasten.dev/demos/)
[7] [https://veeamkasten.dev](https://veeamkasten.dev/demo-getting-started)
[8] [https://www.youtube.com](https://www.youtube.com/playlist?list=PLYAs0sHjyBtmQ7m7DoSsS-b1M3SozJ9Hs)
[9] [https://www.youtube.com](https://www.youtube.com/watch?v=-UF68kJjWsM&t=38)
[10] [https://community.veeam.com](https://community.veeam.com/onboarding-for-veeam-kasten-for-kubernetes-179/onboarding-for-veeam-kasten-for-kubernetes-10535)
```

# Phần 3. Bảng khảo sát đánh giá xác định các yêu cầu đáp ứng K10 dùng cho K8s ?

# Khảo sát Yêu cầu Kỹ thuật & Thông số Kỹ thuật: Sẵn sàng Sao lưu Kubernetes (K10)

**Mục đích:** Bản khảo sát này nhằm thu thập các thông số kỹ thuật và kiến trúc quan trọng cho cả hai môi trường Kubernetes: Phát triển (DEV) và Sản xuất (PROD). Thông tin thu thập được sẽ đảm bảo quá trình cài đặt diễn ra suôn sẻ, phân quyền chính xác và cấu hình các chính sách sao lưu Kasten K10 có khả năng phục hồi cao.

---

### Phần 1: Cơ sở hạ tầng Kubernetes & Kiến trúc Cluster
Phần này xác định mô hình triển khai nền tảng, cấu hình control plane và các kiểm soát truy cập quản lý từng cluster.

| Hạng mục Thông số | Chi tiết Môi trường DEV | Chi tiết Môi trường PROD |
| :--- | :--- | :--- |
| **Loại / Bản phân phối Kubernetes** *(ví dụ: EKS, AKS, GKE, OpenShift, Rancher, Upstream vanilla K8s)* | | |
| **Phiên bản Kubernetes** *(ví dụ: v1.30.x)* | | |
| **Mô hình Triển khai / Lưu trữ** *(ví dụ: On-Premises, AWS, Azure, GCP, Hybrid)* | | |
| **Kiến trúc Node trong Cluster** *(Tổng số lượng, HĐH của Worker Node, Kiến trúc CPU ví dụ: x86_64, ARM64)* | | |
| **Mô hình Xác thực & Truy cập** *(ví dụ: OIDC, Active Directory, AWS IAM Roles for Service Accounts - IRSA)* | | |
| **Chiến lược RBAC của Cluster** *(K10 sẽ được cấp quyền `cluster-admin` toàn cục, hay cần một ClusterRole giới hạn phạm vi?)* | | |

---

### Phần 2: Cơ sở hạ tầng Lưu trữ & Xác thực CSI

Kasten K10 tích hợp sâu với Giao diện lưu trữ container (CSI) để thực hiện các snapshot volume ở cấp độ lưu trữ. Phần này đánh giá mức độ sẵn sàng của hạ tầng lưu trữ.

| Hạng mục Thông số | Chi tiết Môi trường DEV | Chi tiết Môi trường PROD |
| :--- | :--- | :--- |
| **Nhà cung cấp Lưu trữ Khối (Block Storage) Chính** *(ví dụ: AWS EBS, Azure Disk, Ceph, Pure Storage, NetApp)* | | |
| **Nhà cung cấp Lưu trữ Tập tin (File Storage) Chính** *(ví dụ: EFS, Azure Files, NFS)* | | |
| **Driver CSI đã cài đặt & Phiên bản** *(ví dụ: `://aws.com`, driver `gp3`)* | | |
| **Định nghĩa StorageClass** *(Liệt kê các lớp lưu trữ sẽ yêu cầu thiết lập bản đồ sao lưu)* | | |
| **Hỗ trợ Snapshot Volume qua CSI** *(`VolumeSnapshotClass` đã được triển khai và cấu hình chưa? Có/Không)* | | |
| **Các tính năng của Driver CSI được bật** *(Driver có hỗ trợ `VolumeSnapshot` và theo dõi block thay đổi/CBT không?)* | | |

### Phần phụ lục 2.1: Cấu trúc Lưu trữ & Phân bổ Ranh giới (Storage Topology & StorageClass)
Phần này làm rõ cấu hình lưu trữ đa vùng (Multi-Zone) và cách Kubernetes gắn kết dữ liệu (Volume Binding Mode) để tránh lỗi bất đối xứng vùng khi K10 khôi phục pod.

| Hạng mục Thông số Topology Storage | Chi tiết Môi trường DEV | Chi tiết Môi trường PROD |
| :--- | :--- | :--- |
| **Cấu hình Hạ tầng Đa vùng** *(Cluster triển khai trên 1 Zone hay Multi-Zone? ví dụ: us-east-1a, 1b, 1c)* | | |
| **Thuộc tính `volumeBindingMode`** *(Cấu hình của StorageClass là `Immediate` hay `WaitForFirstConsumer`?)* | | |
| **Chế độ Gắn kết Volume (`accessModes`)** *(Các PV chủ yếu sử dụng `ReadWriteOnce` - RWO hay `ReadWriteMany` - RWX?)* | | |
| **Ranh giới CSI Topology (Allowed Topologies)** *(StorageClass có giới hạn vùng chạy cố định không? ví dụ: `topology.kubernetes.io/zone`)* | | |
| **Cơ chế Nhân bản Lưu trữ Vật lý** *(Dữ liệu gốc được đồng bộ đồng thời giữa các Vùng hay chỉ nằm cố định tại một tủ đĩa/vùng cụ thể?)* | | |

---

### Phần 3: Mạng, Bảo mật & Kết nối
Các thành phần của K10 yêu cầu giao tiếp nội bộ cluster, kết nối đến hạ tầng lưu trữ đối tượng bên ngoài và quyền truy cập ingress để quản trị dashboard.

| Hạng mục Thông số | Chi tiết Môi trường DEV | Chi tiết Môi trường PROD |
| :--- | :--- | :--- |
| **Plugin Mạng / CNI** *(ví dụ: Calico, Cilium, Flannel, AWS VPC CNI)* | | |
| **Trạng thái Chính sách Mạng (Network Policies)** *(Các NetworkPolicy nghiêm ngặt về cả hướng ra - egress và hướng vào - ingress có được thực thi không? Có/Không)* | | |
| **Quyền Truy cập Internet Hướng ra ngoài** *(Egress của cluster có bị giới hạn bởi proxy hoặc tường lửa mạng cô lập/air-gapped không?)* | | |
| **Kết nối Lưu trữ Đối tượng Bên ngoài** *(Cluster sẽ kết nối đến đích sao lưu bằng cách nào? ví dụ: Internet công cộng, VPC Endpoints, Private Link)* | | |
| **Chiến lược Ingress Controller** *(Dashboard K10 sẽ được public bằng cách nào? ví dụ: NGINX Ingress, AWS ALB, NodePort + Port-Forward)* | | |
| **Quản lý Chứng chỉ TLS/SSL** *(ví dụ: cert-manager, Let's Encrypt, Chứng chỉ CA nội bộ doanh nghiệp)* | | |


### Phần phụ lục 3.1: Sơ đồ Mạng & Ranh giới Định tuyến (Network Topology & Isolation)
Phần này thu thập cấu trúc định tuyến lớp mạng để đảm bảo các tiến trình chạy ngầm (K10 Data Movers) có thể giao tiếp thông suốt với các thành phần khác mà không bị chặn bởi tường lửa phân vùng.

| Hạng mục Thông số Topology Network | Chi tiết Môi trường DEV | Chi tiết Môi trường PROD |
| :--- | :--- | :--- |
| **Mô hình Mạng của Cụm (Cluster Network Mode)** *(Sử dụng chế độ Overlay - ví dụ: VXLAN/Geneve hay Non-Overlay/Direct Routing - ví dụ: AWS VPC CNI, Calico BGP)* | | |
| **Phân vùng Mạng cho Các Node (Subnet Topology)** *(Các Node nằm trên cùng một Subnet phẳng hay chia tách giữa các Subnet/VLAN khác nhau?)* | | |
| **Vị trí của Đích chứa Sao lưu (Backup Target Location)** *(Hệ thống Lưu trữ Đối tượng S3 nằm cùng Trung tâm dữ liệu/VPC hay nằm ở vùng mạng/Cloud khác?)* | | |
| **Kiểm soát Tường lửa Hạ tầng (Infrastructure Firewalls)** *(Có tường lửa cứng/Security Group chặn giao tiếp giữa các Node thuộc các Vùng khác nhau không?)* | | |
| **Dịch vụ Mesh hoặc Proxy Tầng Ứng dụng** *(Cluster có cài đặt Service Mesh - ví dụ: Istio, Linkerd làm thay đổi luồng đi của mTLS không?)* | | |

---

### Phần 4: Đích đến Sao lưu & Lưu trữ Đối tượng (Location Profiles)
Kasten K10 xuất các bản snapshot ra một hệ thống lưu trữ đối tượng bên ngoài để lưu trữ dài hạn và phục hồi sau thảm họa **(Disaster Recovery).**

1. **Nhà cung cấp Lưu trữ Đối tượng Đích:**
   * [ ] AWS S3
   * [ ] Azure Blob Storage
   * [ ] Google Cloud Storage
   * [ ] Lưu trữ tương thích S3 On-Premises (ví dụ: MinIO, Ceph, Cloudian)
   * [ ] Khác: ___________________________

2. **Chi tiết & Tính năng của Bucket:**
   * **Tên Bucket:** (DEV: ___________________________ / PROD: ___________________________)
   * **Yêu cầu Khóa Đối tượng / Tính Không thể sửa đổi (Immutability):** [ ] Có  [ ] Không
   * **Yêu cầu Mã hóa:** [ ] Quản lý bởi KMS  [ ] Khóa do khách hàng quản lý  [ ] Mặc định của hệ thống lưu trữ

3. **Cơ chế Xác thực từ K10 đến Lưu trữ Đối tượng:**
   * [ ] Hồ sơ Thực thể IAM / Quyền của Node (Node Roles)
   * [ ] Danh tính Khối lượng Công việc (Workload Identity) / IRSA / AKS Pod Identity
   * [ ] Sử dụng trực tiếp Access/Secret Key hoặc File JSON chứa Khóa của Service Account

---

### Phần 5: Đánh giá Khối lượng Công việc, Phạm vi & Kỳ vọng SLA
Việc hiểu rõ quy mô và đặc thù của khối lượng công việc giúp đảm bảo K10 được cấp phát tài nguyên hợp lý và các khung thời gian sao lưu không ảnh hưởng đến hiệu năng hệ thống.

| Hạng mục Thông số | Chi tiết Môi trường DEV | Chi tiết Môi trường PROD |
| :--- | :--- | :--- |
| **Số lượng Namespace Dự kiến Cần Bảo vệ** | | |
| **Loại Khối lượng Công việc Chính** *(Ứng dụng có trạng thái - Stateful, Không trạng thái - Stateless, hoặc Hỗn hợp)* | | |
| **Các Cơ sở Dữ liệu Nội khu Cluster Cần Bảo vệ** *(ví dụ: PostgreSQL, MySQL, MongoDB, Redis)* | | |
| **Tương tác Cơ sở Dữ liệu Lớn** *(Có cần các blueprint cấp ứng dụng để tạm dừng/đóng băng - quiescing/freezing cơ sở dữ liệu khi sao lưu không?)* | | |
| **Mục tiêu Điểm Phục hồi (RPO) mong muốn** *(ví dụ: Mỗi 1 giờ, Mỗi 24 giờ)* | | |
| **Mục tiêu Thời gian Phục hồi (RTO) mong muốn** *(ví dụ: Dưới 15 phút, Dưới 4 giờ)* | | |
| **Chính sách Giữ lại Bản sao lưu** *(ví dụ: 7 bản hàng ngày, 4 bản hàng tuần, 12 bản hàng tháng)* | | |

---

### Phần 6: Thông tin Liên hệ Quản trị & Mức độ Sẵn sàng của Hệ thống
* **Quản trị viên Kubernetes Chính:** ___________________________
* **Trưởng nhóm Lưu trữ / Cơ sở hạ tầng:** ___________________________
* **Đầu mối Liên hệ Vận hành Mạng / Bảo mật:** ___________________________
* **Ngày Triển khai Kasten K10 Dự kiến:** ___________________________


---

***Ghi chú:***
💡 Tại sao 2 bảng bổ sung phụ lục 2.1 và 3.1 này lại bắt buộc phải có cho K10?

Để giúp bạn hiểu sâu hơn khi làm việc với đội nhóm hạ tầng, dưới đây là lý do kỹ thuật đằng sau:
1. **Rủi ro từ volumeBindingMode: Immediate (Storage Topology):**
**Nếu StorageClass** của bạn để chế độ Immediate, khi K10 khôi phục (Restore) một ứng dụng sang một Node mới, Volume có thể bị hệ thống cấp phát ngẫu nhiên ở Zone A, trong khi Pod của ứng dụng lại bị scheduler đẩy sang Zone B. 
Kết quả là Pod sẽ bị kẹt vĩnh viễn ở trạng thái **ContainerCreating** hoặc **VolumeAffinity** do không thể kéo đĩa băng qua các Zone khác nhau. Chúng ta cần cấu hình **WaitForFirstConsumer.**

2.**Thách thức từ Multi-Zone Cluster (Storage Topology):**
Kasten K10 cần biết driver CSI có hiểu được các nhãn (Labels) về vùng địa lý hay không (topology.kubernetes.io/zone). 
Nếu không, K10 không thể đưa ra quyết định thông minh khi chỉ định vị trí tạo Snapshot hoặc khôi phục dữ liệu gốc.

3. **Luồng dữ liệu của K10 Data Mover (Network Topology)**: Kasten K10 không chỉ chạy ở Node Master. Khi có lệnh sao lưu, K10 sẽ tự động dựng lên các Pod tạm thời gọi là K10 Data Movers rải rác trên các Worker Node để đọc dữ liệu đĩa và đẩy ra Cloud Object Storage (S3). Nếu mạng của bạn là hạ tầng Hybrid (Lai) hoặc có tường lửa chặn cổng (_ví dụ: chặn HTTPS/443 hoặc các cổng lưu trữ nội bộ_) giữa các Subnet của Worker Node, tiến trình sao lưu sẽ bị treo ở mức 0% và báo lỗi Timeout.
