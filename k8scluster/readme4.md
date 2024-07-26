# Phần 4. Triển khai Cụm Kubernetes trong Ubuntu 24.04 LTS server trên vSphere vSAN Cluster hoặc vSphere vVOLUME phiên bản 8x:

## 4.1. vSphere vSAN và VMware vVOLUME là gì ?

### 4.1.1. vSAN (Virtual SAN):

Là một giải pháp lưu trữ phân tán và software-defined của VMware, cho phép tổ chức lưu trữ dữ liệu trên cụm máy chủ vSphere. 
vSAN kết hợp các ổ đĩa ổ cứng và ổ SSD từ các máy chủ thành một hệ thống lưu trữ hiệu suất cao, tự động phân phối và quản lý dữ liệu.

![image](https://github.com/user-attachments/assets/f2c9c25d-b326-40b7-b09a-b3f12dd4a360)

**vSAN** bao gồm nhiều tính năng để tăng hiệu quả và khả năng khôi phục cho môi trường lưu trữ và tính toán dữ liệu:

1​. **Shared storage support:**	Hỗ trợ các tính năng của Vmware như HA, vMotion, DRS (các tinh năng này yêu cầu phải có một bộ lưu trữ dùng chung).

2​. **On-disk format	vSAN:** cung cấp các định dạng đĩa giúp hỗ trợ quản lí bản sao và ảnh chụp nhanh có khả năng mở rộng cao.

3​. **All-flash and hybrid configurations:** vSAN có thể cấu hình cho cụm All Flash và Hybrid.

4​. **Fault domains:** vSAN hỗ trợ cấu hình miền lỗi để bảo vệ máy chủ khỏi lỗi rack hoặc chassis khi cụm vSAN kéo dài trên nhiều rack hoặc chassis.

5​. **File service:**	Cho phep tạo và chia sẻ tệp trong kho dữ liệu vSAN.

6​. **iSCSI target service:**	Cho phép máy chủ vật lý nằm bên ngoài cụm vSAN truy cập vào kho dữ liệu vSAN.

7​. **Stretched cluster and Two node cluster:**	Hỗ trợ kéo dài cụm vSAN trên 2 vị trí địa lý.

8​. **Support for Windows Server Failover Clusters (WSFC):**	Hỗ trợ SCSI3-PR ở cấp độ đĩa ảo mà cụm WSFC yêu cầu để cấu chia sẻ ổ đĩa chung cho các máy ảo trên kho dữ liệu vSAN.

9​. **vSAN health service:**	Được cấu hình sẵn để kiểm tra sức khỏe vSAN, khác phục sự cố, chuẩn đoán nguyên nhân gây ra sự cố, xác định các rủi ro tìm ẩn,..

10​. **vSAN performance service:** Là các biểu đồ thống kê được sử dụng để giám sát IOPS, throughput, latency, … của cụm vSAN, máy chủ, disk group, máy ảo.

11​. **Integration with vSphere storage features:**	Tích hợp với các tính năng mà vSphere thường sử dụng với bộ lưu trữ VMFS, NFS (snapshot, linked clone, vShphere Replication).

12​. **Virtual Machine Storage Policies:**	vSAN sử dụng các chính sách lưu trữ để hỗ trợ phương pháp quản lý lưu trữ lấy VM làm trung tâm.

13​. **Rapid provisioning:**	Cung cấp nhanh chóng dung lượng lưu trữa trong vCenter Server trong quá trình tạo và triển khai máy ảo.

14. **Deduplication and compression:**	Thực hiện nén và chống trùng lặp ở cấp độ khối để tiết kiệm dung lượng lưu trữ (all flash).
    
16. **Data at rest encryption:**	Mã hoá dữ liệu khi lưu trữ (dữ liệu được xử ly cuối cùng, sau khi thực hiện các quá trình xử lý khác như chống trùng lặp).
    
18. **Data in transit encryption:**	Mã hóa khi truyền dữ liệu qua các máy chủ trong cụm (tất cả lưu lượng dữ liệu và siêu dữ liệu giữa các host).
    
20. **SDK support	VMware vSAN SDK:** là phần mở rộng của VMware vSphere Management SDK. Bao gồm tài liệu, thư viện và code example giúp nhà phát triển tự động hóa việc cài đặt, cấu hình, giám sát và khắc phục sự cố của vSAN.

**Các thành phần và định nghĩa tham gia trong kiến trúc vSAN:**

**Disk Group (vSAN Original Storage Architecture):**

- Trong mỗi máy chủ ESXi (tối thiểu 3 esxi node trong 1 cụm DRS, tối đa 24 esxi node dùng kèm GPU, hoặc tối đa 32 esxi node không có GPU trong 1 cụm DRS có license Standard vSAN, 20 ESXi nodes mỗi site với mô hình vSAN Stretched Clusters, tối đa 5 Disk group / 1 node - kiến trúc OSA hoặc tối đa 24 disk trên 1 Node - kiến trúc ESA) sẽ đóng góp các disk cục bộ của nó bào cụm vSAN, các disk này được sắp xếp thành các nhóm đĩa (disk group).
- Mỗi disk group phải có 1 fash cache và 1 đến 7 disk dung lượng (HDD hoặc SSD).
- Mỗi máy chủ ESXi có tối đa 5 disk group.

**Storage Pool - vSAN Express Storage Architecture:**

- Thay thế cho Disk Group trong kiến trúc OSA.
- Mỗi máy chủ ESXi chứa một Storage Pool, mỗi thiết bị trong Storage Pool đều đóng góp vào vùng Cache và Capacity (dung lượng), khác với Disk group là phải xác định thiết bị cho tầng Cache và Capacity size.

![image](https://github.com/user-attachments/assets/261a12f9-199d-46d0-9575-56230df81e7a)

**Consumed Capacity:**

Dung lượng vật lý được tiêu thụ bởi một hay nhiều máy ảo tại bất kì thời điểm nào.

**Object-Based Storage:**

vSAN lưu trữ và quản lý dữ liệu dưới dạng các thùng chứa dữ liệu linh hoạt gọi là đối tượng (Object). Object là một khối logic có dữ liệu và siêu dữ liệu phân bổ trên toàn cụm (ví dụ .vmdk là một đối tượng).

**vSAN Datastore:**
Sau khi bật vSAN trên một cụm, một kho dữ liệu duy nhất sẽ được tạo có thể truy cập được đối với tất cả máy chủ trong cụm cho dù nó có đóng góp dung lượng lưu trữ hay không.

**Objects and Components:**
Mỗi đối tượng bao gồm một tập hợp các thành phần, được xác định bởi các khả năng được sử dụng trong VM Storage Policy. 
_Ví dụ: với Failures to tolerate được đặt thành 1, vSAN đảm bảo rằng các thành phần bảo vệ, chẳng hạn như bản sao và witness, được đặt trên các máy chủ riêng biệt trong cụm vSAN, trong đó mỗi bản sao là một thành phần đối tượng._

Ngoài ra, trong cùng một chính sách, nếu “Số lượng dải đĩa trên mỗi đối tượng” (Number of disk stripes per object) được cấu hình thành hai hoặc nhiều hơn, vSAN cũng phân chia đối tượng đó trên nhiều thiết bị dung lượng và mỗi sọc được coi là một thành phần của đối tượng được chỉ định. Khi cần, vSAN cũng có thể chia các đối tượng lớn thành nhiều thành phần.

_Ví dụ bên dưới là một đối tượng (vmdk) có chính sách lưu trữ là Raid-1, số lượng dải đĩa trên mỗi đối tượng là 2 do đó đối tượng của VM sẽ được nhân bản ra 2 trên 2 host (theo raid-1) và trên mỗi host đối tượng sẽ chia thành 2 (tức C1=100GB, C2=100 GB) và trải dài trên 2 host_

![image](https://github.com/user-attachments/assets/b4337df9-2fb2-4603-b0d1-3d67b3528caa)

**Kho dữ liệu vSAN chứa các loại đối tượng sau:**

**VM Home Namespace:** lưu trữ các tệp cấu hình máy ảo như .vmx, log, .vmdk, snapshot delta

**VMDK:** đĩa máy ảo hoặc tệp .vmdk lưu trữ nội dung trên ổ cứng của máy ảo

**VM Swap Object:** được tạo khi bật máy ảo

**Snapshot Delta VMDKs:** được tạo khi chụp nhanh máy ảo (không áp dụng cho kiến trúc ESA)

**Memory object:** được tạo khi tùy chọn snapshot memory được chọn khi tạo snapshot hoặc tạm dừng máy ảo

**Virtual Machine Compliance Status: Compliant and Noncompliant**  Một máy ảo được coi là không tuân thủ (Noncompliant ) khi một hoặc nhiều đối tượng của nó không đáp ứng được các yêu cầu của chính sách lưu trữ được chỉ định.

**Component State: Degraded and Absent States:**  vSAN xác nhận các trạng thái lỗi sau đối với các thành phần:

**Degraded (Giảm sút chất lượng):**  - Một thành phần bị xuống cấp khi vSAN phát hiện lỗi thành phần vĩnh viễn và xác định rằng thành phần bị lỗi không thể khôi phục về trạng thái hoạt động ban đầu. Kết quả là vSAN bắt đầu xây dựng lại các thành phần bị xuống cấp ngay lập tức. Trạng thái này có thể xảy ra khi một thành phần trên thiết bị bị lỗi.

**Absent (Vắng mặt):**  - Một thành phần vắng mặt khi vSAN phát hiện lỗi thành phần tạm thời trong đó các thành phần, bao gồm tất cả dữ liệu của nó, có thể khôi phục và đưa vSAN về trạng thái ban đầu. Trạng thái này có thể xảy ra khi bạn khởi động lại máy chủ hoặc nếu bạn rút phích cắm thiết bị khỏi máy chủ vSAN. vSAN bắt đầu xây dựng lại các thành phần ở trạng thái vắng mặt sau khi chờ 60 phút.

**Object State: Healthy and Unhealthy:** - Tùy thuộc vào loại và số lượng lỗi trong cụm, một đối tượng có thể ở một trong các trạng thái sau:

**Healthy (Khỏe mạnh):**  - Khi có ít nhất một máy nhân bản RAID 1 đầy đủ hoặc có sẵn số lượng phân đoạn dữ liệu tối thiểu cần thiết, đối tượng được coi là hoạt động tốt.
Unhealthy (Không khỏe mạnh). Một đối tượng được coi là không tốt khi không có sẵn bản sao đầy đủ hoặc số lượng phân đoạn dữ liệu tối thiểu cần thiết không có sẵn cho các đối tượng RAID 5 hoặc RAID 6. Nếu có ít hơn 50 phần trăm số phiếu bầu của một đối tượng thì đối tượng đó là Unhealthy. Nhiều lỗi trong cụm có thể khiến các đối tượng trở nên không tốt. Khi trạng thái hoạt động của một đối tượng được coi là không tốt, nó sẽ ảnh hưởng đến tính khả dụng của VM liên quan.

**Witness:** - Witness (Nhân chứng / điều khiển trung gian) là thành phần chỉ chứa siêu dữ liệu và không chứa bất kỳ dữ liệu ứng dụng thực tế nào.
Đóng vai trò như một công cụ ngắt kết nối khi không tồn tại số đại biểu trong cụm vSAN. Nhân chứng tiêu tốn khoảng 2 MB dung lượng cho siêu dữ liệu trên kho dữ liệu vSAN khi sử dụng định dạng trên đĩa 1.0 và 4 MB cho định dạng trên đĩa phiên bản 2.0 trở lên.
Mỗi thành phần của một đối tượng sẽ có 1 hoặc nhiều phiếu bầ

**Storage Policy-Based Management (SPBM):** - vSAN đảm bảo rằng các máy ảo được triển khai vào kho dữ liệu vSAN được chỉ định ít nhất một chính sách lưu trữ máy ảo.
Mỗi VM có thể được gán chính sách riêng cho nó, tùy thuộc vào nhu cầu. Nếu không áp dụng chính sách lưu trữ khi triển khai máy ảo, vSAN sẽ tự động gán chính sách vSAN mặc định với Failures to tolerate được đặt thành 1, một disk stripe cho từng đối tượng và đĩa ảo được cung cấp ở chế độ thin provisioned .

**vSphere PowerCLI:** - VMware vSphere PowerCLI bổ sung hỗ trợ tập lệnh dòng lệnh cho vSAN, để giúp bạn tự động hóa các tác vụ quản lý và cấu hình.
<hr></hr>

#### Về thiết bị lưu trữ:

**Kiến trúc OSA**

***Cache​:***
1. Một ổ SSD (SAS hoặc SATA) hoặc PCIe flash 
1. Cung cấp ít nhất 10% dung lượng lưu trữ dự kiến sẽ tiêu thụ trên các ổ Capacity (không bao gồm các bản sao)
1. Thiết bị flash không có định dạng VMFS hoặc định dạng của các hệ thống tệp khác

***Capacity​:***
1. Ít nhất một ổ SAS hoặc NL-SAS (kiến trúc hybrid)
1. Ít nhất một ổ SSD (SAS hoặc SATA) hoặc PCIe flash (kiến trúc all flash)

***Storage controllers​***
1. Card HBA (SAS hoặc SATA) hoặc Raid Controller ở chế độ pasthrough hoặc Raid 0
1. Không kết hợp controller cho các ổ đĩa vSAN và không phải vSAN (nếu vSAN ở mode Raid thì các ổ non-vSAN (ổ đĩa không dùng để đóng góp vào dung lượng vSAN) cũng ở mode Raid)
1. Không chạy VM từ đĩa hoặc Raid group đã chia sẻ bộ điều khiển (controller) của nó cho đĩa vSAN hoặc Raid Group
1. Không passthrought các ổ đĩa non-vSAN tới máy ảo dưới dạng RDM (Raw Device Mapping)
1. Nếu các ổ non- vSAN sử dụng VMFS thì chỉ sử dụng các VMFS datatore này cho scratch, logging, core dumps


**Kiến trúc ESA:**

***Cache and capacity​:*** Có ít nhất 4 thiết bị NVMe cho mỗi Storage Pool

***Về bộ nhớ máy chủ:*** Kiến trúc ESA yêu cầu ít nhất 512 GB mỗi máy chủ

***Về thiết bị Boot:*** 
1. Nếu bộ nhớ máy chủ ít hơn 512 GB: có thể khởi động máy chủ từ USB (dung lượng phải lớn hơn 4GB), SD (dung lượng phải lớn hơn 4GB), SATADOM
1. Nếu bộ nhớ máy chủ nhiều hơn 512 GB: có thể khởi động máy chủ từ SATADOM (dùng single-level cell - SLC) hoặc ổ cứng (dung lượng phải lớn hơn 16 GB)
1. Khi khởi động từ USB/SD vSAN trace log sẽ được ghi vào RAMDisk sau đó tải xuống Presistent do đó khi mất điện các vSAN trace log cũng sẽ mất

***Về cơ sở hạ tầng mạng:***

****Host Bandwidth​:****
1. vSAN OSA: 1 Gbps cho cấu hình hybrid, 10 Gbps cho cấu hình all flash
1. vSAN ESA: 25 Gbps

****Connection between hosts​:**** Các máy chủ ESXi đều phải có bộ điều hợp mạng Vmkernel cho lưu lượng vSAN

****Host network​:**** Các máy chủ trongg cụm vSAN đều phải kết nối với mạng vSAN Layer 2 hoặc Layer 3

****Network latency​:**** 
1. Tối đa 1 ms RTT cho vSAN Standard Cluster (lưu lượng giữa các host)
1. Tối đa 5 ms RTT cho vSAN Stretched Cluster (lưu lượng giữa 2 site)
1. Tối đa 200ms RTT từ site chính đến vSAN witness host

***Về giấy phép:***
1. Sau khi bật tính năng vSAN trên một cluster cần phải gán cho cluster đó giấy phép vSAN thích hợp
1. Giấy phép vSAN được tính bằng tổng số CPU trong các máy chủ tham gia vào cụm. Ví dụ: một cụm vSAN chứa 4 máy chủ, mỗi máy chủ có 8 CPU, hãy gán cho cụm đó một giấy phép vSAN có dung lượng tối thiểu là 32 CPU.
1. Khi giấy phép Evaluation hết hạn, bạn có thể sử dụng các tài nguyên trên cluster, tuy nhiên không thể cấu hình thêm ổ đĩa, Disk Group,..

## 4.2.1. Các kiến trúc và mô hình xây dựng vSAN Cluster:

### Mô hình 1. vSAN Original Storage Architecture:

- Kiến trúc OSA được thiết kế cho nhiều loại thiết bị lưu trữ (HDD và SSD). Mỗi máy chủ đóng góp dung lượng lưu trữ có một hoặc nhiều Disk Group. Mỗi Disk Group chứa một SDD dùng làm bộ đệm và nhiều ổ đĩa (SSD và HDD) dùng để đóng góp vào dung lượng.

![image](https://github.com/user-attachments/assets/94edcf61-e877-4e73-ae2a-b3028e079278)

### Mô hình 2. vSAN Express Storage Architecture:

- Kiến trúc ESA được thiết kế cho các thiết bị Flash Nvme. Mỗi máy chủ đóng góp dung lượng lưu trữ có một Storage Pool duy nhất (gồm 4 thiết bị flash trở lên), mỗi thiết bị flash vừa dùng làm bộ đệm vừa đóng góp vào dung lượng của cụm vSAN.

![image](https://github.com/user-attachments/assets/26a45f25-096e-47f7-bc12-260be5fc3c4f)

### Mô hình 3. vSAN Ready Node:

- Là giải pháp cấu hình sẵn của vSAN được cung cấp bởi các đối tác của VMware như Dell, Fujitsu, IBM,.. Giải pháp này đã bao gồm cấu hình máy chủ đã được kiểm tra và chứng nhận để triển khai vSAN

### Mô hình 4. Standard vSAN Cluster:

- Cụm vSAN tối thiểu 3 máy chủ. Các máy chủ trong cụm thường nằm ở một site duy nhất và được kết nối trên cungk mạng layer 2.
- Cấu hình All Flash yêu cầu phải có kết nối mạng 10Gb, với kiến trúc ESA yêu cầu kết nối mạng 25Gb.

![image](https://github.com/user-attachments/assets/085d2066-e323-4ce3-ac1b-6ff4ec647354)

### Mô hình 5. Two-Node vSAN Cluster:

- Gồm 2 máy chủ ở cùng một site và một máy chủ Witness. Máy chủ Witness có thể đặt ở site khác với site chính. Thông thường máy chủ Witness ở cùng site với site chính và cùng với vCenter Server.

![image](https://github.com/user-attachments/assets/06845ce1-eccb-4609-8bc0-f9e0d1bbc23e)

### Mô hình 6. vSAN Stretched Cluster:

1. Các máy chủ trên vSAN Stretched Cluster được phân bố đều ở 2 site
1. Hai site phải có độ trễ mạng không quá 5ms
1. Máy chủ Witness đặt ở site thứ 3

![image](https://github.com/user-attachments/assets/6bc6c1d0-d46a-4f88-85f9-437bf5213d21)

### 4.1.2. vVols (Virtual Volumes):
Là một công nghệ lưu trữ mạnh mẽ của VMware, cho phép quản lý dữ liệu ở mức độ đơn vị cấp VM thay vì ở mức LUN hoặc tập tin. 
vVols cho phép VMs tận dụng các tính năng lưu trữ nâng cao như luồng dữ liệu thẳng đến ổ đĩa và tích hợp mạnh mẽ với VMware vSphere.

### 4.1.3. vSAN DataStorage khác với DataStorage trên lớp vSphere 8:

Trong ngữ cảnh của VMware, vSAN Datastore và Datastore trên lớp vSphere là hai khái niệm khác nhau về việc lưu trữ dữ liệu trong môi trường ảo hóa vSphere. Dưới đây là sự khác biệt giữa chúng:

**vSAN Datastore:**

1. vSAN Datastore là một datastore đặc biệt được tạo ra từ cụm vSAN, nơi dữ liệu được lưu trữ và quản lý trên cụm máy chủ vSphere sử dụng công nghệ vSAN.
1. Dữ liệu trên vSAN Datastore được phân tán trên các ổ đĩa ổ cứng và ổ SSD trên các máy chủ thành một hệ thống lưu trữ phân tán và software-defined.
1. vSAN Datastore hỗ trợ tính năng như tích hợp với vSphere, quản lý dựa trên các storage policy, và cung cấp khả năng linh hoạt và tăng cường hiệu suất.

**Datastore trên lớp vSphere 8:**

1. Datastore trên lớp vSphere là khái niệm chung đề cập đến bất kỳ hình thức lưu trữ nào được sử dụng trong môi trường vSphere, bao gồm không chỉ vSAN Datastore mà còn cả các loại lưu trữ khác như NFS, NFC, SMB, CIFS, iSCSI, RDM, FCoE, FC, và VMFS.
1. Datastore trên lớp vSphere 8.0 bao gồm tất cả các datastore được sử dụng để lưu trữ các tập tin VM, dữ liệu và các thông tin quan trọng khác trong môi trường ảo hóa vSphere.
1. Datastore trên lớp vSphere cung cấp một phạm vi rộng lớn của các loại lưu trữ và khả năng tích hợp với nhiều giải pháp lưu trữ khác nhau.

**Tóm lại:**

1. vSAN Datastore là một datastore đặc biệt được tạo ra từ cụm vSAN, trong khi Datastore trên lớp vSphere 8.0 đề cập đến mọi loại lưu trữ được sử dụng trong môi trường vSphere.
1. vSAN Datastore là một phần của lớp lưu trữ vSAN, cung cấp lưu trữ phân tán và hiệu suất cao, trong khi Datastore trên lớp vSphere bao gồm tất cả các loại lưu trữ khác nhau được sử dụng trong môi trường ảo hóa vSphere.

## 4.2.2. Triển khai Kubernetes và Persistent Volumes trên vSAN Stretched Cluster:

_Tham khảo: https://docs.vmware.com/en/VMware-vSphere-Container-Storage-Plug-in/3.0/vmware-vsphere-csp-getting-started/GUID-982E7698-3C8D-4F88-A9EA-3A38970370EA.html_

Bạn có thể triển khai cụm Kubernetes chung và các khối liên tục trên các cụm kéo dài vSAN . Bạn có thể triển khai nhiều cụm Kubernetes với các yêu cầu lưu trữ khác nhau trong cùng một cụm vSAN .

Các cụm kéo dài vSAN hỗ trợ khối lượng tệp được hỗ trợ bởi tính năng chia sẻ tệp vSAN . Để biết thêm thông tin, hãy xem Cung cấp khối lượng tệp với Plug-in lưu trữ vùng chứa vSphere .

**Điều kiện tiên quyết:**

Khi bạn định định cấu hình cụm Kubernetes trên cụm kéo dài vSAN , hãy xem xét các mục sau:

1. Cụm Kubernetes chung không thực thi cùng một chính sách lưu trữ trên các máy ảo nút và trên các ổ đĩa liên tục. Quản trị viên vSphere chịu trách nhiệm về cấu hình, chỉ định và sử dụng chính sách lưu trữ chính xác trong các cụm Kubernetes.
1. Sử dụng chính sách lưu trữ VM với cùng cài đặt sao chép và mối quan hệ Site cho tất cả các đối tượng lưu trữ trên cụm Kubernetes. Chính sách lưu trữ tương tự nên được sử dụng cho tất cả các máy ảo nút, bao gồm mặt phẳng điều khiển và nhân viên cũng như tất cả các PV.
1. Không thể sử dụng tính năng cấu trúc liên kết để cung cấp ổ đĩa thuộc miền lỗi cụ thể trong cụm kéo dài vSAN.

### Thủ tục:

1. Thiết lập cụm kéo dài vSAN của bạn .
1.1.  Tạo một cụm kéo dài vSAN: Để biết thêm thông tin, hãy tìm kiếm cụm mở rộng vSAN trên trang Tài liệu VMware vSAN .
1.1.  Bật DRS trên cụm kéo dài.
1.1.  Bật vSphere HA: Đảm bảo thiết lập Giám sát máy chủ .
1.1.  Cho phép giám sát máy chủ và định cấu hình phản hồi lỗi máy chủ, phản hồi cách ly máy chủ và giám sát VM.

**Ghi chú:**
VMware khuyên bạn nên tắt Bảo vệ thành phần VM (VMCP) khi tất cả các máy ảo và ổ đĩa Node được triển khai trên Kho dữ liệu vSAN.
- Vô hiệu hóa kho dữ liệu bằng PDL.
- Vô hiệu hóa kho dữ liệu bằng APD.

![image](https://github.com/user-attachments/assets/f25060f5-d579-4783-9d70-2075f756a625)

2. Tạo chính sách lưu trữ VM tuân thủ các yêu cầu của cụm mở rộng vSAN.
2.1. Cấu hình khả năng chống chịu thảm họa của site: Chọn Phản chiếu Site kép để phản chiếu dữ liệu ở cả hai Site của cụm kéo dài.

![image](https://github.com/user-attachments/assets/40b7045b-3c81-4df6-8843-a82a97883ebb)

2.2. Chỉ định các lỗi có thể chịu đựng được.
- Đối với cụm bị kéo dài, cài đặt xác định số lỗi ổ đĩa hoặc máy chủ mà đối tượng lưu trữ có thể chịu đựng đối với từng site. Số miền lỗi hoặc máy chủ cần thiết trong một Site cho cụm kéo dài để chịu được n lỗi là **2n + 1** để phản chiếu.

**Phản chiếu Raid-1** cung cấp hiệu suất tốt hơn. **Raid-5** và **Raid-6** đạt được khả năng chịu lỗi bằng cách sử dụng các khối chẵn lẻ, mang lại hiệu quả không gian tốt hơn. Các tùy chọn này chỉ có sẵn trên các cụm đèn flash toàn phần.

![image](https://github.com/user-attachments/assets/bd258d95-1e06-460a-9d5c-e8d135c658d9)

2.3. Kích hoạt tính năng cung cấp bắt buộc.

![image](https://github.com/user-attachments/assets/2fbd5015-32e3-4cdd-93fb-db13cacfab6b)

3. Tạo quy tắc quan hệ VM-Host để đặt các nút Kubernetes trên Site chính hoặc phụ cụ thể, chẳng hạn như Site-A.

![image](https://github.com/user-attachments/assets/06204190-0b23-4bad-bd28-85d00fbba236)

Để biết thông tin về các quy tắc quan hệ, hãy xem _Tạo quy tắc quan hệ VM-Host trong tài liệu Quản lý tài nguyên vSphere._

4. Triển khai máy ảo Kubernetes bằng chính sách lưu trữ cụm mở rộng vSAN .
5. Tạo lớp lưu trữ bằng chính sách lưu trữ cụm mở rộng vSAN .
6. Triển khai các khối lượng liên tục bằng cách sử dụng lớp lưu trữ cụm mở rộng vSAN .

#### Phải làm gì tiếp theo:

Tùy thuộc vào nhu cầu và môi trường của bạn, bạn có thể sử dụng một trong các kịch bản triển khai sau khi triển khai cụm Kubernetes và khối lượng công việc trên cụm kéo dài vSAN .

#### Triển khai 1

Trong quá trình triển khai này, mặt phẳng điều khiển và các nút công nhân được đặt trên site chính nhưng đủ linh hoạt để chuyển đổi dự phòng trên site khác nếu site chính bị lỗi. Bạn triển khai HA Proxy trên site chính. Đây còn được gọi là triển khai Chủ động-Thụ động vì chỉ có một site của cụm vSAN kéo dài được sử dụng để triển khai máy ảo.

Nếu bạn dự định sử dụng ổ đĩa tệp (ổ đĩa RWX), bạn nên định cấu hình miền dịch vụ tệp vSAN để đặt máy chủ tệp trên site hoạt động (site ngầm định). Điều này làm giảm độ trễ lưu lượng truy cập chéo site và mang lại hiệu suất tốt hơn cho các ứng dụng sử dụng khối lượng tệp.

**Yêu cầu triển khai 1:**

    + Vị trí nút:
    + Mặt phẳng điều khiển và các nút công nhân nằm trên site chính. Chúng đủ linh hoạt để chuyển đổi dự phòng sang site khác nếu site chính bị lỗi.
    + HA Proxy trên site chính.
    + Không thể chịu được tải:	Ít nhất là FTT1
    + DRS:	Đã bật
    + Khả năng chịu được sự cố của site:	Có 2 site cluster HA
    + Chính sách lưu trữ Force Provisioning:	Đã bật
    + vSphere HA:	Đã bật.


**Các kịch bản chuyển đổi dự phòng tiềm năng khi triển khai 1:**

Bảng sau mô tả các tình huống chuyển đổi dự phòng tiềm ẩn có thể xảy ra khi bạn sử dụng mô hình triển khai 1.

**Kịch bản:**
- Một số máy chủ ESXi bị lỗi trên site chính:	
    + Máy ảo nút Kubernetes di chuyển từ các máy chủ không có sẵn sang các máy chủ có sẵn trong các site chính.
    + Nếu nút worker cần được khởi động lại, các nhóm chạy trên nút đó có thể được lên lịch lại và tạo lại trên nút khác.
    + Nếu nút mặt phẳng điều khiển cần được khởi động lại thì khối lượng công việc của ứng dụng hiện có sẽ không bị ảnh hưởng.

- Toàn bộ site chính và tất cả các máy chủ trên site đều bị lỗi:	
    + Máy ảo nút Kubernetes di chuyển từ site chính sang site phụ.
    + Bạn gặp phải thời gian ngừng hoạt động hoàn toàn cho đến khi nút VM khởi động lại trên site phụ.

- Một số máy chủ bị lỗi trên site phụ:	
    + Lỗi không ảnh hưởng đến cụm Kubernetes vì ​​toàn bộ cụm đều nằm ở site chính.

- Toàn bộ site phụ và tất cả các máy chủ trên site đó đều bị lỗi:	
    + Lỗi không ảnh hưởng đến cụm Kubernetes vì ​​toàn bộ cụm đều nằm ở site chính.
    + Sao chép các đối tượng lưu trữ dừng lại vì site phụ không sẵn dùng.

- Lỗi mạng liên site xảy ra:
    + Lỗi không ảnh hưởng đến cụm Kubernetes vì ​​toàn bộ cụm đều nằm ở site chính.
    + Sao chép các đối tượng lưu trữ dừng lại vì Site phụ không sẵn dùng.

**Triển khai 2:**

- Với mô hình này, đặt các nút mặt phẳng điều khiển trên site chính và các nút công nhân có thể được trải rộng trên site chính và phụ. Bạn triển khai HA Proxy trên site chính.

**Yêu cầu triển khai 2:**

    + Vị trí nút
    + Các nút mặt phẳng điều khiển: trên Site chính.
    + Các nút công nhân trải rộng: trên Site chính và phụ.
    + HA Proxy: trên Site chính.
    + Không thể chịu được tải:	Ít nhất là FTT1
    + DRS:	Đã bật
    + Khả năng chịu đựng sự cố của Site:	Có 2 sites HA
    + Chính sách lưu trữ Force Provisioning:	Đã bật
    + vSphere HA: Đã bật

**Các kịch bản chuyển đổi dự phòng tiềm năng khi triển khai 2:**

Bảng sau mô tả các tình huống chuyển đổi dự phòng tiềm ẩn có thể xảy ra khi bạn triển khai cụm Kubernetes bằng mô hình Triển khai 2.

- Một số máy chủ ESXi bị lỗi trên site chính:
 
    + Máy ảo nút Kubernetes di chuyển từ các máy chủ không có sẵn sang các máy chủ có sẵn trong cùng một site. Nếu tài nguyên không có sẵn, họ sẽ di chuyển đến Site bao phấn.
    + Nếu nút worker cần được khởi động lại, các nhóm chạy trên nút đó có thể được lên lịch lại và tạo lại trên nút khác.
    + Nếu nút mặt phẳng điều khiển cần được khởi động lại thì khối lượng công việc của ứng dụng hiện có sẽ không bị ảnh hưởng.

Toàn bộ Site chính và tất cả các máy chủ trên Site đều bị lỗi.	
    + Máy ảo nút mặt phẳng điều khiển Kubernetes và một số nút công nhân có trên site chính sẽ di chuyển từ site chính sang site phụ.
    + Dự kiến ​​thời gian ngừng hoạt động của mặt phẳng điều khiển cho đến khi các nút mặt phẳng điều khiển khởi động lại trên site phụ.
    + Dự kiến ​​sẽ có thời gian ngừng hoạt động một phần đối với các nhóm chạy trên nút công nhân trên Site chính.
    + Các nhóm được triển khai trên các nút công nhân trên site phụ không bị ảnh hưởng.

Một số máy chủ bị lỗi trên site phụ: Các máy ảo nút và nhóm chạy trên máy ảo nút khởi động lại trên một máy chủ khác.

Toàn bộ site phụ và tất cả các máy chủ trên site đó đều bị lỗi.	
    + Mặt phẳng điều khiển Kubernetes không bị ảnh hưởng.
    + Các nút mặt phẳng điều khiển Kubernetes di chuyển đến site chính.
    + Các nhóm được triển khai trên các nút công nhân trên site phụ bị ảnh hưởng. Họ khởi động lại cùng với các máy ảo nút.

Lỗi mạng liên site xảy ra.	
    + Mặt phẳng điều khiển Kubernetes không bị ảnh hưởng.
    + Các nút công nhân Kubernetes di chuyển đến site chính.
    + Các nhóm được triển khai trên các nút công nhân trên site phụ bị ảnh hưởng. Họ khởi động lại cùng với các máy ảo nút.

**Triển khai 3:**

- Trong mô hình triển khai này, bạn có thể đặt hai nút mặt phẳng điều khiển trên site chính và một nút mặt phẳng điều khiển trên site phụ. Triển khai HA Proxy trên site chính. Các nút công nhân có thể ở trên bất kỳ Site nào.

**Yêu cầu triển khai 3:**

Bạn có thể sử dụng mô hình triển khai này nếu bạn có các tài nguyên như nhau ở cả miền lỗi chính hoặc miền ưu tiên và miền lỗi thứ cấp, không ưu tiên và bạn muốn sử dụng phần cứng nằm ở cả hai miền lỗi. Vì cả hai miền lỗi đều có một số khối lượng công việc đang chạy nên trong trường hợp Site bị lỗi hoàn toàn, mô hình triển khai này sẽ giúp khôi phục nhanh hơn.

Mô hình này yêu cầu các quy tắc chính sách DRS cụ thể. Một quy tắc để chỉ định mối quan hệ giữa hai nút mặt phẳng điều khiển và site chính và một quy tắc khác về mối quan hệ giữa nút mặt phẳng điều khiển thứ ba và site phụ.

    - Vị trí nút:
      + Hai nút mặt phẳng điều khiển trên Site chính.
      + Một nút mặt phẳng điều khiển trên Site phụ.
      + HA Proxy trên Site chính.
      + Nút công nhân trên bất kỳ Site nào.  
    - Không thể chịu được tải: Ít nhất là FTT1
    - DRS: Đã bật
    - Khả năng chịu đựng sự cố của site: có 2 Site HA
    - Chính sách lưu trữ Force Provisioning: Đã bật
    - vSphere HA: Đã bật

**Các kịch bản chuyển đổi dự phòng tiềm năng khi triển khai 3:**

Bảng sau mô tả các tình huống chuyển đổi dự phòng tiềm ẩn có thể xảy ra khi bạn sử dụng mô hình Triển khai 3.

- Một số máy chủ ESXi bị lỗi trên site chính:	
    + Các nút bị ảnh hưởng sẽ được khởi động lại trên máy chủ có sẵn trên site chính.
    + Nếu cả hai nút mặt phẳng điều khiển đều có mặt trên máy chủ bị lỗi trên site chính, thì mặt phẳng điều khiển sẽ ngừng hoạt động cho đến khi cả hai nút mặt phẳng điều khiển phục hồi trên các máy chủ có sẵn trên site chính.
    + Trong khi các nút đang khởi động lại trên các máy chủ có sẵn, các nhóm có thể được lên lịch lại và tạo lại trên các nút có sẵn.

- Toàn bộ Site chính và tất cả các máy chủ trên Site đều bị lỗi:
    + Máy ảo nút di chuyển từ site chính sang site phụ.
    + Dự kiến ​​sẽ có thời gian ngừng hoạt động cho đến khi máy ảo nút khởi động lại trên site phụ.
      
- Một số máy chủ bị lỗi trên site phụ.:	
    + Các máy ảo nút và nhóm chạy trên máy ảo nút khởi động lại trên một máy chủ khác.
    + Nếu nút mặt phẳng điều khiển trên site phụ bị ảnh hưởng thì mặt phẳng điều khiển Kubernetes vẫn không bị ảnh hưởng. Kubernetes vẫn có thể truy cập được thông qua hai nút chính trên site chính.

- Toàn bộ site phụ và tất cả các máy chủ trên site đó đều bị lỗi:
    + Nút mặt phẳng điều khiển và nút công nhân từ site phụ được di chuyển sang site chính.
    + Các nhóm được triển khai trên các nút công nhân trên site phụ bị ảnh hưởng. Họ khởi động lại cùng với các máy ảo nút.

- Lỗi mạng liên site xảy ra:
    + Mặt phẳng điều khiển Kubernetes không bị ảnh hưởng.
    + Các nút Kubernetes di chuyển đến site chính.
    + Các nhóm được triển khai trên các nút công nhân trên site phụ bị ảnh hưởng. Họ khởi động lại cùng với các máy ảo nút.


### Nâng cấp Kubernetes và Persistent Volume trên các cụm kéo dài vSAN:

Nếu đã triển khai Kubernetes trên kho dữ liệu vSAN , bạn có thể nâng cấp hoạt động triển khai của mình sau khi bật các cụm kéo dài vSAN trên kho dữ liệu.

#### Thủ tục:
1. Chỉnh sửa chính sách lưu trữ VM hiện có được sử dụng để cung cấp ổ đĩa và VM nút trên cụm vSAN để thêm các tham số cụm kéo dài.
1. Áp dụng chính sách lưu trữ cập nhật trên tất cả các đối tượng.
1. Áp dụng chính sách lưu trữ đã cập nhật trên các ổ đĩa liên tục có trạng thái Đã lỗi thời .
