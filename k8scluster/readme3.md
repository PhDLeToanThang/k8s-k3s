# Phần 3. Triển khai Cụm Kubernetes trên vSphere với CSI và CPI (chi dung cho K8s dung vSphere CSI)
https://cloud-provider-vsphere.sigs.k8s.io/tutorials/kubernetes-on-vsphere-with-kubeadm.html

> Mục đích của hướng dẫn này là cung cấp cho người đọc hướng dẫn từng bước về cách triển khai Kubernetes trên cơ sở hạ tầng vSphere . 
Hướng dẫn sử dụng kubeadm, một công cụ được xây dựng để cung cấp “đường dẫn nhanh” có phương pháp thực hành tốt nhất để tạo cụm Kubernetes . 
Người đọc cũng sẽ tìm hiểu cách triển khai các plugin Giao diện lưu trữ vùng chứa và Giao diện nhà cung cấp đám mây cho các hoạt động cụ thể của vSphere . 
Ở cuối hướng dẫn này, bạn sẽ có một K8 chạy đầy đủ trên môi trường vSphere , cho phép cung cấp khối lượng động.

## Điều kiện tiên quyết
Phần này sẽ bao gồm các điều kiện tiên quyết cần phải có trước khi thử triển khai.

## Yêu cầu vSphere
vSphere 6.7U3 (hoặc mới hơn) là điều kiện tiên quyết để sử dụng CSI và CPI tại thời điểm viết bài. Điều này có thể thay đổi trong tương lai và tài liệu sẽ được cập nhật để phản ánh bất kỳ thay đổi nào trong tuyên bố hỗ trợ này. Nếu bạn đang sử dụng phiên bản vSphere dưới 6.7 U3, bạn có thể nâng cấp vSphere lên 6.7U3 hoặc làm theo một trong các hướng dẫn dành cho các phiên bản vSphere cũ hơn . Dưới đây là hướng dẫn triển khai Kubernetes với kubeadm, sử dụng VCP - Triển khai Kubernetes bằng kubeadm với vSphere Cloud Provider (in-tree) .

## Yêu cầu về tường lửa
Việc cung cấp quyền truy cập (các) nút chính của K8 vào giao diện quản lý vCenter là đủ, do các nhóm CPI và CSI được triển khai trên (các) nút chính. Nếu các thành phần này được triển khai trên các nút XỬ LÝ hay nói cách khác - các nút đó cũng sẽ cần quyền truy cập vào giao diện quản lý vCenter.

Nếu bạn muốn sử dụng tính năng cung cấp khối lượng nhận biết cấu trúc liên kết và tính năng liên kết muộn bằng cách sử dụng zone/ region, nút cần khám phá cấu trúc liên kết của nó bằng cách kết nối với vCenter, để làm được điều này, mọi nút đều có thể giao tiếp với vCenter. Bạn có thể tắt tính năng tùy chọn này nếu bạn chỉ muốn mở nút chính vào giao diện quản lý vCenter.

## Hệ điều hành khách được đề xuất
VMware khuyên bạn nên tạo mẫu máy ảo bằng Máy chủ PC 64-bit (AMD64) Guest OS Ubuntu 18.04.1 LTS (Bionic Beaver). Hãy kiểm tra nó trên VMware PartnerWeb . Mẫu này được sao chép để hoạt động như hình ảnh cơ sở cho cụm Kubernetes của bạn . Để biết hướng dẫn về cách thực hiện việc này, vui lòng tham khảo hướng dẫn được cung cấp trong bài đăng trên blog này của Myles Gray của VMware . Đảm bảo rằng quyền truy cập SSH được bật trên tất cả các nút. Điều này phải được thực hiện để chạy các lệnh trên cả nút chính và nút XỬ LÝ Kubernetes trong hướng dẫn này.

## Yêu cầu phần cứng máy ảo
Phần cứng máy ảo phải là phiên bản 15 trở lên. Đối với các yêu cầu về CPU và bộ nhớ của Máy ảo, kích thước phù hợp dựa trên yêu cầu về khối lượng công việc. VMware cũng khuyến nghị các máy ảo nên sử dụng bộ điều khiển VMware Paravirtual SCSI cho Đĩa chính trên Node VM . Đây phải là mặc định nhưng bạn nên kiểm tra. Cuối cùng, tham số disk.EnableUUID phải được đặt cho mỗi máy ảo nút. Bước này là cần thiết để VMDK luôn hiển thị UUID nhất quán cho VM , do đó cho phép đĩa được gắn đúng cách. Bạn không nên chụp ảnh nhanh các máy ảo nút CNS để tránh lỗi và hành vi không thể đoán trước.

## Hình ảnh Docker
Sau đây là danh sách docker image cần thiết để cài đặt CSI và CPI trên Kubernetes . Những hình ảnh này được tự động kéo vào khi các bảng kê khai CSI và CPI được triển khai. VMware phân phối và đề xuất các image sau:
```images
gcr.io/cloud-provider-vsphere/csi/release/driver:v3.2.0
gcr.io/cloud-provider-vsphere/csi/release/syncer:v3.2.0
registry.k8s.io/cloud-pv-vsphere/cloud-provider-vsphere:v1.30.1
```
Ngoài ra, bạn có thể sử dụng các hình ảnh sau hoặc bất kỳ hình ảnh vùng chứa nguồn mở hoặc thương mại nào có sẵn phù hợp cho việc triển khai CSI.
**Lưu ý:**  các thẻ tham chiếu phiên bản của các thành phần khác nhau. Điều này sẽ thay đổi với các phiên bản trong tương lai:
```images
quay.io/k8scsi/csi-provisioner:v1.2.2
quay.io/k8scsi/csi-attacher:v1.1.1
quay.io/k8scsi/csi-node-driver-registrar:v1.1.0
quay.io/k8scsi/livenessprobe:v1.1.0
registry.k8s.io/kube-apiserver:v1.14.2
registry.k8s.io/kube-controller-manager:v1.14.2
registry.k8s.io/kube-scheduler:v1.14.2
registry.k8s.io/kube-proxy:v1.14.2
registry.k8s.io/pause:3.1
registry.k8s.io/etcd:3.3.10
registry.k8s.io/coredns:1.3.1
```

## Công cụ
Nếu bạn dự định triển khai Kubernetes trên vSphere từ môi trường MacOS, trình brewquản lý gói có thể được sử dụng để cài đặt và quản lý các công cụ cần thiết. 
Nếu sử dụng môi trường Linux hoặc Windows để bắt đầu triển khai, các liên kết đến các công cụ sẽ được bao gồm. 
Làm theo hướng dẫn cụ thể của công cụ để cài đặt công cụ trên các hệ điều hành khác nhau.
Đối với mỗi công cụ, lệnh cài đặt bia cho MacOS được hiển thị ở đây.

- brew- https://brew.sh.
- govc- brew tap govmomi /tap/govc && brew install govmomi /tap/govc.
- kubectl- cài đặt kubernetes -cli.
- tmux(tùy chọn) - cài đặt tmux.

Dưới đây là link các công cụ và hướng dẫn cài đặt cho các hệ điều hành khác:
- govc dành cho các Hệ điều hành khác - nên dùng phiên bản 0.20.0 trở lên.
- kubectl cho các Hệ điều hành khác.
- tmux cho các Hệ điều hành khác.
  
## Thiết lập máy ảo và hệ điều hành khách:
Bước tiếp theo là cài đặt các thành phần Kubernetes cần thiết trên máy ảo Ubuntu OS. Một số thành phần phải được cài đặt trên tất cả các nút. 
Trong các trường hợp khác, một số thành phần chỉ cần được cài đặt trên máy chủ và trong các trường hợp khác, chỉ cần cài đặt trên máy XỬ LÝ. 
Trong mỗi trường hợp, vị trí cài đặt các thành phần sẽ được đánh dấu. 
Tất cả các lệnh cài đặt và cấu hình phải được thực thi với quyền root. 
Bạn có thể chuyển sang môi trường root bằng lệnh "sudo su". 
Các bước thiết lập cần thiết trên tất cả các nút Phần sau đây nêu chi tiết các bước cần thiết trên cả nút chính và nút XỬ LÝ.

## Cài đặt VMTools (nếu cần)
Để biết thêm thông tin về VMTools bao gồm cả cài đặt, vui lòng truy cập tài liệu chính thức.

## disk.EnableUUID=1
Các lệnh govc sau đây sẽ đặt disk.EnableUUID=1 trên tất cả các nút.
```vmx
# export GOVC_INSECURE=1
# export GOVC_URL='https://<VC_IP>'
# export GOVC_USERNAME=VC_Admin_User
# export GOVC_PASSWORD=VC_Admin_Passwd

# govc ls
/datacenter/vm
/datacenter/network
/datacenter/host
/datacenter/datastore
```
Để truy xuất tất cả các máy ảo Node, hãy sử dụng lệnh sau:
```vmx
# govc ls /<datacenter-name>/vm
/datacenter/vm/k8s-node3
/datacenter/vm/k8s-node4
/datacenter/vm/k8s-node1
/datacenter/vm/k8s-node2
/datacenter/vm/k8s-master
```
Để sử dụng govc để kích hoạt Disk UUID, hãy sử dụng lệnh sau:
```vmx
# govc vm.change -vm '/datacenter/vm/k8s-node1' -e="disk.enableUUID=1"
# govc vm.change -vm '/datacenter/vm/k8s-node2' -e="disk.enableUUID=1"
# govc vm.change -vm '/datacenter/vm/k8s-node3' -e="disk.enableUUID=1"
# govc vm.change -vm '/datacenter/vm/k8s-node4' -e="disk.enableUUID=1"
# govc vm.change -vm '/datacenter/vm/k8s-master' -e="disk.enableUUID=1"
```
Thông tin thêm về disk.enableUUID có thể được tìm thấy trong VMware Knowledgebase Article 52815 (https://kb.vmware.com/s/article/52815).

## Nâng cấp phần cứng máy ảo
Phần cứng HW version VM phải ở phiên bản 15 trở lên.
```vmx
# govc vm.upgrade -version=15 -vm '/datacenter/vm/k8s-node1'
# govc vm.upgrade -version=15 -vm '/datacenter/vm/k8s-node2'
# govc vm.upgrade -version=15 -vm '/datacenter/vm/k8s-node3'
# govc vm.upgrade -version=15 -vm '/datacenter/vm/k8s-node4'
# govc vm.upgrade -version=15 -vm '/datacenter/vm/k8s-master'
```
Kiểm tra phiên bản Phần cứng VM sau khi chạy lệnh trên:
```vmx
# govc vm.option.info '/datacenter/vm/k8s-node1' | grep HwVersion
HwVersion:           15
```
## Tắt Hoán đổi
SSH vào tất cả các nút XỬ LÝ của K8 và vô hiệu hóa trao đổi trên tất cả các nút bao gồm cả nút chính. 
Đây là điều kiện tiên quyết cho kubeadm. 
NẾU bạn đã làm theo hướng dẫn trước đó về cách tạo hình ảnh mẫu hệ điều hành thì bước này sẽ được triển khai.
```sh
# swapoff -a
# vi /etc/fstab
... remove any swap entry from this file ...
```
## Cài đặt Docker CE
Nên sử dụng các bước sau để cài đặt thời gian chạy vùng chứa trên tất cả các nút. Docker CE 18.06 phải được sử dụng. 
Kubernetes có các phiên bản được hỗ trợ rõ ràng, vì vậy nó phải là phiên bản này

Đầu tiên, cập nhật chỉ mục gói apt:
```sh
# apt update
```
Bước tiếp theo là cài đặt các gói để cho phép apt sử dụng kho lưu trữ qua HTTPS:
```sh
# apt install ca-certificates software-properties-common \
apt-transport-https curl -y
```
Bây giờ hãy thêm khóa GPG chính thức của Docker:
```sh
# curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
```
Để hoàn tất quá trình cài đặt, hãy thêm kho lưu trữ docker apt:
```sh
# add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
```
Bây giờ chúng ta có thể cài đặt Docker CE. Để cài đặt một phiên bản cụ thể, hãy thay thế chuỗi phiên bản bằng số phiên bản mong muốn:
```sh
# apt update
# apt install docker-ce=18.06.0~ce~3-0~ubuntu -y
```
Cuối cùng, thiết lập các tham số daemon, như xoay vòng nhật ký và nhóm:
```sh
# tee /etc/docker/daemon.json >/dev/null <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
# mkdir -p /etc/systemd/system/docker.service.d
```
Và để hoàn tất, hãy khởi động lại docker để nhận các thông số mới:
```sh
# systemctl daemon-reload
# systemctl restart docker
```
Docker hiện đã được cài đặt. Xác minh trạng thái của docker thông qua lệnh sau:
```status
#systemctl status docker
 docker.service - Docker Application Container Engine
   Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2019-09-06 12:37:27 UTC; 4min 15s ago

#docker info | egrep "Server Version|Cgroup Driver"
Server Version: 18.06.0-ce
Cgroup Driver: systemd
```

## Cài đặt Kubelet, Kubectl, Kubeadm
Bước tiếp theo là cài đặt các thành phần Kubernetes chính trên mỗi nút.

kubeadmlà lệnh khởi động lại cụm. Nó chạy trên nút chính và tất cả các nút XỬ LÝ. 
kubeletlà thành phần chạy trên tất cả các nút trong cụm và thực hiện các tác vụ như khởi động nhóm và vùng chứa. 
kubectllà tiện ích dòng lệnh để giao tiếp với cụm của bạn. Nó chỉ chạy trên nút chính. 
Đối với các bản phân phối Ubuntu, một phiên bản cụ thể có thể được cài đặt với phiên bản chỉ định của tên gói, 
ví dụ apt install -qy kubeadm=1.14.2-00 kubelet=1.14.2-00 kubectl=1.14.2-00: . Đây phải là phiên bản tối thiểu được cài đặt.

Đầu tiên, kho lưu trữ Kubernetes cần được thêm vào apt.
```sh
# curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
# cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
Tiếp theo cài đặt kubelet , kubectl và kubeadm.
```sh
# apt update
# apt install -qy kubeadm=1.14.2-00 kubelet=1.14.2-00 kubectl=1.14.2-00
```
Cuối cùng, giữ các gói Kubernetes ở phiên bản đã cài đặt của chúng để không nâng cấp đột ngột khi nâng cấp apt.
```sh
# apt-mark hold kubelet kubeadm kubectl
```

## Bước thiết lập cho flannel (Pod Networking)
Chúng tôi sẽ sử dụng flannel cho mạng nhóm trong ví dụ này, vì vậy phần dưới đây cần được chạy trên tất cả các nút để chuyển lưu lượng IPv4 được bắc cầu sang chuỗi iptables:
```sh
# sysctl net.bridge.bridge-nf-call-iptables=1
```
Điều đó hoàn thành các bước thiết lập chung trên cả nút chính và nút XỬ LÝ. 
Bây giờ chúng ta sẽ xem xét các bước liên quan đến việc kích hoạt Giao diện nhà cung cấp đám mây vSphere (CPI) và Giao diện lưu trữ vùng chứa (CSI) trước khi sẵn sàng triển khai cụm Kubernetes của mình . Hãy chú ý đến nơi các bước được thực hiện, sẽ là trên nút chính hoặc nút XỬ LÝ.

## Cài đặt (các) nút chính Kubernetes
Một lần nữa, các bước này chỉ được thực hiện trên bản gốc. 
Sử dụng kubeadminit để khởi tạo nút chính. 
Để khởi tạo nút chính, trước hết chúng ta cần tạo một kubeadminit.yaml tệp kê khai cần được chuyển tới kubeadmlệnh. 

**Lưu ý:** tham chiếu đến nhà cung cấp đám mây bên ngoài trong nodeRegistrationphần tệp kê khai.
```sh
# tee /etc/kubernetes/kubeadminit.yaml >/dev/null <<EOF
apiVersion: kubeadm.k8s.io/v1beta1
kind: InitConfiguration
bootstrapTokens:
       - groups:
         - system:bootstrappers:kubeadm:default-node-token
         token: y7yaev.9dvwxx6ny4ef8vlq
         ttl: 0s
         usages:
         - signing
         - authentication
nodeRegistration:
  kubeletExtraArgs:
    cloud-provider: external
---
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
useHyperKubeImage: false
kubernetesVersion: v1.14.2
networking:
  serviceSubnet: "10.96.0.0/12"
  podSubnet: "10.244.0.0/16"
etcd:
  local:
    imageRepository: "registry.k8s.io"
    imageTag: "3.3.10"
dns:
  type: "CoreDNS"
  imageRepository: "registry.k8s.io"
  imageTag: "1.5.0"
EOF
```
Khởi động nút chính Kubernetes bằng tệp cấu hình cụm được tạo ở bước trên.
```sh
# kubeadm init --config /etc/kubernetes/kubeadminit.yaml
[init] Using Kubernetes version: v1.14.2
. .
. .
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
. .
. .
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.192.116.47:6443 --token y7yaev.9dvwxx6ny4ef8vlq \
    --discovery-token-ca-cert-hash sha256:[sha sum from above output]
```
> **Lưu ý:** phần cuối cùng của đầu ra cung cấp lệnh nối các nút XỬ LÝ với nút chính trong cụm Kubernetes này.
> Chúng tôi sẽ sớm quay lại bước đó. Tiếp theo, thiết lập tệp kubeconfig trên tệp chính để các lệnh Kubernetes CLI như kubectl có thể được sử dụng trên cụm Kubernetes mới tạo của bạn.
```sh
# mkdir -p $HOME/.kube
# cp /etc/kubernetes/admin.conf $HOME/.kube/config
```
Bạn cũng có thể sử dụng kubectltrên các hệ thống bên ngoài (không phải hệ thống chính) bằng cách sao chép nội dung của hệ thống chính vào tệp /etc/kubernetes/admin.conftrên máy tính cục bộ của bạn .~/.kube/config

Ở giai đoạn này, bạn có thể nhận thấy các nhóm coredns vẫn ở trạng thái chờ xử lý kèm theo FailedSchedulingtrạng thái. Điều này là do nút chính có những vết bẩn mà nhóm coredns không thể chịu đựng được. Điều này được mong đợi, vì chúng tôi đã bắt đầu kubeletvới cloud-provider: external. Sau khi cài đặt Giao diện nhà cung cấp đám mây vSphere và các nút được khởi tạo, các vết bẩn sẽ tự động bị xóa khỏi nút và điều đó sẽ cho phép lập lịch cho các nhóm coreDNS.
```sh
# kubectl get pods --namespace=kube-system
NAME                                 READY   STATUS    RESTARTS   AGE
coredns-fb8b8dccf-q57f9              0/1       Pending   0          87s
coredns-fb8b8dccf-scgp2              0/1       Pending   0          87s
etcd-k8s-master                      1/1       Running   0          54s
kube-apiserver-k8s-master            1/1       Running   0          39s
kube-controller-manager-k8s-master  1/1       Running   0          54s
kube-proxy-rljk8                     1/1       Running   0          87s
kube-scheduler-k8s-master            1/1       Running   0          37s

# kubectl describe pod coredns-fb8b8dccf-q57f9 --namespace=kube-system
.
.
Events:
  Type     Reason            Age                 From               Message
  ----     ------            ----                ----               -------
  Warning  FailedScheduling  7s (x21 over 2m1s)  default-scheduler  0/1 nodes are available: 1 node(s) had taints that the pod didn't tolerate.
```
## Cài đặt mạng lớp phủ flannel pod:
Bước tiếp theo cần được thực hiện trên nút chính là mạng lớp phủ nhóm flannel phải được cài đặt để các nhóm có thể giao tiếp với nhau. Lệnh cài đặt flannel trên bản gốc như sau:
```sh
# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/62e44c867a2846fefb68bd5f178daf4da3095ccb/Documentation/kube-flannel.yml
```
Vui lòng làm theo các hướng dẫn thay thế này để cài đặt mạng lớp phủ nhóm ngoài mạng flannel.

Tại thời điểm này, bạn có thể kiểm tra xem mạng lớp phủ đã được triển khai chưa.
```sh
# kubectl get pods --namespace=kube-system
NAME                               READY   STATUS    RESTARTS   AGE
coredns-6557d7f7d6-9s7sm           0/1     Pending   0          107s
coredns-6557d7f7d6-wgxtq           0/1     Pending   0          107s
etcd-k8s-mstr                      1/1     Running   0          70s
kube-apiserver-k8s-mstr            1/1     Running   0          54s
kube-controller-manager-k8s-mstr   1/1     Running   0          53s
kube-flannel-ds-amd64-pm9m9        1/1     Running   0          11s
kube-proxy-8dfm9                   1/1     Running   0          107s
kube-scheduler-k8s-mstr            1/1     Running   0          49s
```
## Xuất cấu hình nút chính:
Cuối cùng, cấu hình nút chính cần phải được xuất vì nó được sử dụng bởi các nút XỬ LÝ muốn tham gia vào nút chính.
```sh
# kubectl -n kube-public get configmap cluster-info -o jsonpath='{.data.kubeconfig}' > discovery.yaml
```
Tệp discovery.yamlsẽ cần được sao chép vào /etc/kubernetes/discovery.yamlmỗi nút XỬ LÝ.

## Cài đặt (các) nút XỬ LÝ Kubernetes:
Thực hiện nhiệm vụ này trên các nút XỬ LÝ. Xác minh rằng bạn đã cài đặt Docker CE, kubeadm, v.v. trên các nút XỬ LÝ trước khi thử thêm chúng vào nút chính.

Để (các) nút XỬ LÝ tham gia vào nút chính, bạn phải tạo một tệp yaml cấu hình kubeadm của nút XỬ LÝ. 
> **Lưu ý:** nó đang được sử dụng /etc/kubernetes/discovery.yamllàm đầu vào cho khám phá chính. 
> Chúng tôi sẽ hướng dẫn cách sao chép tệp từ trình chạy sang tệp chính trong bước tiếp theo.
> Ngoài ra, hãy lưu ý rằng mã thông báo được sử dụng trong cấu hình nút XỬ LÝ giống như mã thông báo chúng tôi đặt trong kubeadminitmaster.yaml
> cấu hình chính ở trên. Cuối cùng, một lần nữa chúng tôi xác định rõ rằng nhà cung cấp đám mây là bên ngoài đối với nhân viên vì chúng tôi sẽ sử dụng CPI mới.
```sh
# tee /etc/kubernetes/kubeadminitworker.yaml >/dev/null <<EOF
apiVersion: kubeadm.k8s.io/v1beta1
caCertPath: /etc/kubernetes/pki/ca.crt
discovery:
  file:
    kubeConfigPath: /etc/kubernetes/discovery.yaml
  timeout: 5m0s
  tlsBootstrapToken: y7yaev.9dvwxx6ny4ef8vlq
kind: JoinConfiguration
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  kubeletExtraArgs:
    cloud-provider: external
EOF
```
Bạn có thể sao chép tệp discovery.yaml vào máy cục bộ của mình bằng scp.

Đầu tiên, với tư cách là siêu người dùng, hãy sử dụng scp để sao chép /etc/kubernetes/discovery.yaml từ bản gốc /home/ubuntu/discovery.yaml sang tất cả các nút.

Bây giờ bạn sẽ cần phải đăng nhập vào từng nút và sao chép tệp discovery.yamltừ /home/ubuntusang /etc/kubernetes. 
Tệp discovery.yaml phải tồn tại /etc/kubernetes trên các nút.

Sau khi hoàn thành bước đó, hãy chạy lệnh sau trên mỗi nút XỬ LÝ để nó tham gia vào nút chính (và các nút XỬ LÝ khác đã được tham gia) trong cụm:
```sh
# kubeadm join --config /etc/kubernetes/kubeadminitworker.yaml
```
Bạn có thể kiểm tra xem nút đã tham gia hay chưa bằng cách chạy # kubectl get nodestrên bản gốc.

## Cài đặt Giao diện nhà cung cấp đám mây vSphere

Các bước sau đây chỉ được thực hiện trên bản gốc.

> **Lưu ý:** trình điều khiển CSI yêu cầu có ProviderID nhãn trên mỗi nút trong cụm K8s.
> Điều này có thể được thực hiện bằng bất kỳ phương tiện nào thuận tiện nhất - Lý tưởng nhất là nên sử dụng Giao diện nhà cung cấp đám mây (CPI)
> vì nó được duy trì và cập nhật tích cực, nhưng nếu phải sử dụng Nhà cung cấp đám mây vSphere (VCP)
> do các yêu cầu của trường nâu hoặc hạn chế về kiến ​​trúc phân phối của bạn - điều đó cũng có thể chấp nhận được.
> Miễn là ProviderID được điền bằng một số phương tiện - trình điều khiển vSphere CSI sẽ hoạt động.

## Tạo bản đồ cấu hình CPI:
Tệp sơ đồ cấu hình đám mây này, được chuyển tới CPI khi khởi tạo, chứa thông tin chi tiết về cấu hình vSphere .
Tệp này mà chúng tôi gọi ở đây vsphere.conf đã được điền một số giá trị mẫu. Rõ ràng, bạn sẽ cần sửa đổi tệp này để phản ánh cấu hình vSphere của riêng bạn .

> **LƯU Ý:** Kể từ phiên bản CPI 1.2.0 trở lên, định dạng cấu hình đám mây ưu tiên sẽ YAMLdựa trên.
> Định INI dạng dựa trên sẽ không được dùng nữa nhưng được hỗ trợ cho đến khi quá trình chuyển đổi sang YAML cấu hình dựa trên ưu tiên hoàn tất.
> Thông báo ngừng sử dụng này sẽ được đặt trong nhật ký CPI khi sử dụng INI định dạng cấu hình dựa trên.
```sh
# tee /etc/kubernetes/vsphere.conf >/dev/null <<EOF

# Global properties in this section will be used for all specified vCenters unless overriden in VirtualCenter section.
global:
  port: 443
  # set insecureFlag to true if the vCenter uses a self-signed cert
  insecureFlag: true
  # settings for using k8s secret
  secretName: cpi-global-secret
  secretNamespace: kube-system

# vcenter section
vcenter:
  tenant-finance:
    server: 1.1.1.1
    datacenters:
      - finance
  tenant-hr:
    server: 192.168.0.1
    datacenters:
      - hrwest
      - hreast
  tenant-engineering:
    server: 10.0.0.1
    datacenters:
      - engineering
    secretName: cpi-engineering-secret
    secretNamespace: kube-system

# labels for regions and zones
labels:
  region: k8s-region
  zone: k8s-zone

EOF
```

Dưới đây là mô tả về các trường được sử dụng trong sơ đồ cấu hình vsphere .conf:

- insecureFlag phải được đặt thành true để sử dụng chứng chỉ tự ký để đăng nhập.
- server phần được xác định để giữ thuộc tính của địa chỉ IP vcenter hoặc FQDN.
- secretName giữ (các) thông tin xác thực cho một hoặc danh sách Máy chủ vCenter.
- secretNamespace được đặt thành không gian tên nơi bí mật đã được tạo.
- port là Cổng máy chủ vCenter. Mặc định là 443 nếu không được chỉ định.
- datacenters phải là danh sách các trung tâm dữ liệu nơi có máy ảo nút kubernetes.
  
Đối với những người dùng triển khai CPI phiên bản 1.1.0 trở về trước, INI cấu hình dựa trên tương ứng phản ánh cấu hình trên sẽ xuất hiện như sau:
```sh
# tee /etc/kubernetes/vsphere.conf >/dev/null <<EOF

[Global]
port = "443"
insecure-flag = "true"
secret-name = "cpi-global-secret"
secret-namespace = "kube-system"

[VirtualCenter "1.1.1.1"]
datacenters = "finance"

[VirtualCenter "192.168.0.1"]
datacenters = "hrwest,hreast"

[VirtualCenter "10.0.0.1"]
datacenters = "engineering"
secret-name = "cpi-engineering-secret"
secret-namespace = "kube-system"

EOF
```
Tạo sơ đồ cấu hình bằng cách chạy lệnh sau:
```sh
# cd /etc/kubernetes
# kubectl create configmap cloud-config --from-file=vsphere.conf --namespace=kube-system
```
Xác minh rằng sơ đồ cấu hình đã được tạo thành công trong không gian tên kube-system.
```sh
# kubectl get configmap cloud-config --namespace=kube-system
NAME              DATA     AGE
cloud-config      1        82s
```

## Tạo bí mật CPI

CPI hỗ trợ lưu trữ thông tin xác thực vCenter trong:

- Bí mật toàn cầu được chia sẻ chứa tất cả thông tin đăng nhập vCenter hoặc
- Một bí mật dành riêng cho một cấu hình vCenter cụ thể được ưu tiên hơn bất kỳ thứ gì có thể được định cấu hình trong bí mật chung
Trong ví dụ vsphere.conf trên, có hai Kubernetes secret được cấu hình .
vCenter tại 10.0.0.1 chứa thông tin xác thực trong bí mật có tên cpi-engineering-secret trong không gian tên kube-system và vCenter tại 1.1.1.1 và 192.168.0.1 chứa
thông tin xác thực trong bí mật có tên cpi-global-secret trong không gian tên kube-system được xác định trong global:phần.

Một ví dụ về Secrets YAML có thể được sử dụng để tham khảo khi tạo tệp secrets. 
Nếu ví dụ bí mật YAML được sử dụng, hãy cập nhật tên bí mật để sử dụng a <unique secret name>, 
địa chỉ IP vCenter trong các khóa của stringData, và username và password cho mỗi khóa.

Bí mật dành cho vCenter tại 10.0.0.1có thể trông giống như sau:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cpi-engineering-secret
  namespace: kube-system
stringData:
  10.0.0.1.username: "administrator@vsphere.local"
  10.0.0.1.password: "password"
```
Đây là ví dụ Bí mật thứ hai, lần này hiển thị một định dạng thay thế. 
Định dạng thay thế này cho phép sử dụng địa chỉ máy chủ IPv6. 
Định dạng này yêu cầu các mục nhập máy chủ {id}, tên người dùng {id} và mật khẩu_{id}, trong đó các mục nhập có hậu tố chung cho mỗi máy chủ:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cpi-engineering-secret
  namespace: kube-system
stringData:
  server_prod: fd01::102
  username_prod: "administrator@vsphere.local"
  password_prod: "password"
  server_test: 10.0.0.2
  username_test: "developer@vsphere.local"
  password_test: "sekret"
```
Sau đó, để tạo bí mật, hãy chạy lệnh sau thay thế tên của tệp YAML bằng tên bạn đã sử dụng:
```sh
# kubectl create -f cpi-engineering-secret.yaml
```
Xác minh rằng bí mật thông tin xác thực được tạo thành công trong không gian tên kube-system.
```sh
# kubectl get secret cpi-engineering-secret --namespace=kube-system
NAME                    TYPE     DATA   AGE
cpi-engineering-secret   Opaque   1      43s
```
Nếu bạn có nhiều vCenter như trong ví dụ vsphere .conf ở trên, 
Kubernetes Secret YAML của bạn có thể trông giống như sau để lưu trữ thông tin xác thực vCenter cho vCenters tại 1.1.1.1 và 192.168.0.1:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cpi-global-secret
  namespace: kube-system
stringData:
  1.1.1.1.username: "administrator@vsphere.local"
  1.1.1.1.password: "password"
  192.168.0.1.username: "administrator@vsphere.local"
  192.168.0.1.password: "password"
```

## Vùng và khu vực dành cho vị trí nhóm và khối lượng - CPI

Kubernetes cho phép bạn đặt các Pod và Khối liên tục trên các phần cụ thể của cơ sở hạ tầng cơ bản, 
ví dụ: các Trung tâm dữ liệu khác nhau hoặc các vCenter khác nhau, sử dụng khái niệm Vùng và Khu vực. 
Tuy nhiên, để sử dụng các biện pháp kiểm soát vị trí, các bước cấu hình bắt buộc cần phải được thực hiện tại thời điểm triển khai Kubernetes 
và yêu cầu cài đặt bổ sung trong vSphere .conf của cả CPI và CSI. 
Để biết thêm thông tin về cách triển khai hỗ trợ vùng/khu vực, hãy xem hướng dẫn về cách thực hiện tại đây .

Nếu bạn không quan tâm đến vị trí đặt đối tượng của K8 thì có thể bỏ qua phần này và có thể tiến hành các bước thiết lập CPI còn lại.

## Kiểm tra xem tất cả các nút có bị nhiễm độc không
Trước khi cài đặt vSphere Cloud Controller Manager, hãy đảm bảo tất cả các nút đều bị nhiễm node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule. 
Khi kubelet được khởi động với nhà cung cấp đám mây “bên ngoài”, vết bẩn này được đặt trên một nút để đánh dấu nó là không sử dụng được. 
Sau khi bộ điều khiển từ nhà cung cấp đám mây khởi tạo nút này, kubelet sẽ loại bỏ vết bẩn này.
```sh
# kubectl describe nodes | egrep "Taints:|Name:"
Name:               k8s-master
Taints:             node-role.kubernetes.io/master:NoSchedule
Name:               k8s-node1
Taints:             node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule
Name:               k8s-node2
Taints:             node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule
Name:               k8s-node3
Taints:             node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule
Name:               k8s-node4
Taints:             node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule
```

## Triển khai bảng kê khai CPI
Có 3 bảng kê khai phải được triển khai để cài đặt Giao diện nhà cung cấp đám mây vSphere . 
Ví dụ sau áp dụng các vai trò RBAC và liên kết RBAC cho cụm Kubernetes của bạn . 
Nó cũng triển khai Trình quản lý bộ điều khiển đám mây trong DaemonSet.
```sh
# kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-vsphere/master/manifests/controller-manager/cloud-controller-manager-roles.yaml
clusterrole.rbac.authorization.k8s.io/system:cloud-controller-manager created

# kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-vsphere/master/manifests/controller-manager/cloud-controller-manager-role-bindings.yaml
clusterrolebinding.rbac.authorization.k8s.io/system:cloud-controller-manager created

# kubectl apply -f https://github.com/kubernetes/cloud-provider-vsphere/raw/master/manifests/controller-manager/vsphere-cloud-controller-manager-ds.yaml
serviceaccount/cloud-controller-manager created
daemonset.extensions/vsphere-cloud-controller-manager created
service/vsphere-cloud-controller-manager created
```
## Xác minh rằng CPI đã được triển khai thành công
Xác minh vsphere - cloud-controller-manager đang chạy và tất cả các nhóm hệ thống khác đều đang hoạt động 
(lưu ý rằng các nhóm coredns trước đây không chạy - bây giờ chúng sẽ chạy vì các vết bẩn đã được loại bỏ bằng cách cài đặt CPI):
```sh
# kubectl get pods --namespace=kube-system
NAME                                     READY   STATUS    RESTARTS   AGE
coredns-fb8b8dccf-bq7qq                  1/1     Running   0          71m
coredns-fb8b8dccf-r47q2                  1/1     Running   0          71m
etcd-k8s-master                          1/1     Running   0          69m
kube-apiserver-k8s-master                1/1     Running   0          70m
kube-controller-manager-k8s-master       1/1     Running   0          69m
kube-flannel-ds-amd64-7kmk9              1/1     Running   0          38m
kube-flannel-ds-amd64-dtvbg              1/1     Running   0          63m
kube-flannel-ds-amd64-hq57c              1/1     Running   0          30m
kube-flannel-ds-amd64-j7g4s              1/1     Running   0          22m
kube-flannel-ds-amd64-q4zsn              1/1     Running   0          21m
kube-proxy-6jcng                         1/1     Running   0          30m
kube-proxy-bh8kh                         1/1     Running   0          21m
kube-proxy-rb9xp                         1/1     Running   0          22m
kube-proxy-srhpj                         1/1     Running   0          71m
kube-proxy-vh4lg                         1/1     Running   0          38m
kube-scheduler-k8s-master                1/1     Running   0          70m
vsphere-cloud-controller-manager-549hb   1/1     Running   0          25s
```

## Kiểm tra xem tất cả các nút có sạch không
Xác minh node.cloudprovider. kubernetes .io/uninitialized taint bị xóa khỏi tất cả các nút.
```sh
# kubectl describe nodes | egrep "Taints:|Name:"
Name:               k8s-master
Taints:             node-role.kubernetes.io/master:NoSchedule
Name:               k8s-node1
Taints:             <none>
Name:               k8s-node2
Taints:             <none>
Name:               k8s-node3
Taints:             <none>
Name:               k8s-node4
Taints:             <none>
```
> **Lưu ý:** Nếu bạn vô tình mắc lỗi với vsphere.conf, chỉ cần xóa các thành phần CPI và configMap,
> thực hiện mọi chỉnh sửa cần thiết đối với vSphere.conftệp configMap và áp dụng lại các bước trên.

Bây giờ bạn có thể xóa vsphere.conf tệp được tạo tại /etc/kubernetes/.

## Cài đặt trình điều khiển giao diện lưu trữ vùng chứa vSphere

Bây giờ CPI đã được cài đặt, chúng ta có thể tập trung vào CSI. 
Vui lòng truy cập https://vsphere-csi-driver.sigs.k8s.io/driver-deployment/installation.html để cài đặt Trình điều khiển vSphere CSI.

Tệp kê khai mẫu để kiểm tra chức năng của trình điều khiển CSI
Sau đây là một số bảng kê khai mẫu có thể được sử dụng để xác minh rằng một số quy trình cung cấp bằng trình điều khiển vSphere CSI đang hoạt động như mong đợi.

Ví dụ được cung cấp ở đây sẽ chỉ ra cách tạo một ứng dụng có trạng thái được chứa trong bộ chứa và sử dụng vSphere Client để truy cập vào các ổ đĩa hỗ trợ ứng dụng của bạn. Quy trình làm việc mẫu sau đây cho biết cách triển khai ứng dụng MongoDB với một bản sao.

Trong khi thực hiện các tác vụ của quy trình làm việc, bạn luân phiên vai trò của người dùng vSphere và người dùng Kubernetes . Nhiệm vụ sử dụng các mục sau:

- Tệp YAML lớp lưu trữ
- Tệp YAML của dịch vụ MongoDB
- Tệp YAML StatefulSet của MongoDB

## Tạo chính sách lưu trữ
Đĩa ảo (VMDK) sẽ hỗ trợ ứng dụng được đóng gói của bạn cần phải đáp ứng các yêu cầu lưu trữ cụ thể. 
Là người dùng vSphere , bạn tạo chính sách lưu trữ VM dựa trên các yêu cầu do người dùng Kubernetes cung cấp cho bạn . 
Chính sách lưu trữ sẽ được liên kết với VMDK hỗ trợ ứng dụng của bạn. 
Nếu bạn có nhiều phiên bản Máy chủ vCenter trong môi trường của mình, hãy tạo chính sách lưu trữ VM trên từng phiên bản.
Sử dụng cùng một tên chính sách trên tất cả các phiên bản.

- Trong vSphere Client, trên trang đích chính, chọn VM Storage Policies.
- Dưới Policies and Profiles, chọn VM Storage Policies.
- Nhấp chuột Create VM Storage Policy.
- Nhập tên và mô tả chính sách rồi nhấp vào Tiếp theo. Vì mục đích của cuộc trình diễn này, chúng tôi sẽ đặt tên cho nó Space-Efficient.
- Trên trang Cấu trúc chính sách trong các quy tắc dành riêng cho Kho dữ liệu, chọn Enable rules for "vSAN" storagevà nhấp vào Tiếp theo.
- Trên trang vSAN, chúng ta sẽ giữ nguyên giá trị mặc định cho chính sách này là standard clustervà RAID-1 (Mirroring).
- Trên trang Tương thích bộ nhớ, xem lại danh sách kho dữ liệu vSAN phù hợp với chính sách này và nhấp vào Tiếp theo.
- Trên trang Xem lại và kết thúc, xem lại cài đặt chính sách và nhấp vào Kết thúc.

![image](https://github.com/user-attachments/assets/caee6f44-2c9e-43eb-b971-22341181e469)

Bây giờ bạn có thể thông báo cho người dùng Kubernetes về tên chính sách lưu trữ. 
Chính sách lưu trữ VM mà bạn đã tạo sẽ được sử dụng như một phần của định nghĩa lớp lưu trữ để cung cấp ổ đĩa động.

## Tạo một lớp lưu trữ
Với tư cách là người dùng Kubernetes , hãy xác định và triển khai lớp lưu trữ tham chiếu chính sách lưu trữ VM đã tạo trước đó . 
Chúng ta sẽ sử dụng kubectl để thực hiện các bước sau. Nói chung, bạn cung cấp thông tin cho kubectl trong tệp YAML. 
kubectl chuyển đổi thông tin thành JSON khi thực hiện yêu cầu API. 
Bây giờ chúng ta sẽ tạo tệp YAML StorageClass mô tả các yêu cầu lưu trữ cho vùng chứa và tham chiếu chính sách lưu trữ VM sẽ được sử dụng. 
Đây csi.vsphere.vmware.comlà tên của nhà cung cấp vSphere CSI và là tên được đặt trong trường nhà cung cấp trong StorageClass yaml. 
Tệp YAML mẫu sau đây bao gồm chính sách lưu trữ Tiết kiệm không gian mà bạn đã tạo trước đó bằng vSphere Client. 
Khối lượng liên tục VMDK thu được được đặt trên kho dữ liệu tương thích với dung lượng trống tối đa đáp ứng các yêu cầu về chính sách lưu trữ Hiệu quả về Không gian.
```yaml
# cat mongodb-storageclass.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: mongodb-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: csi.vsphere.vmware.com
parameters:
  storagepolicyname: "Space-Efficient"
  fstype: ext4
# kubectl create -f mongodb-storageclass.yaml
storageclass.storage.k8s.io/mongodb-sc created
# kubectl get storageclass mongodb-sc
NAME         PROVISIONER                    AGE
mongodb-sc   csi.vsphere.vmware.com   5s
```
## Tạo một dịch vụ
Với tư cách là người dùng Kubernetes , hãy xác định và triển khai Dịch vụ Kubernetes . 
Dịch vụ cung cấp điểm cuối mạng cho ứng dụng. Sau đây là tệp YAML mẫu xác định dịch vụ cho ứng dụng MongoDB.

```yaml
# cat mongodb-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
  labels:
    name: mongodb-service
spec:
  ports:
  - port: 27017
    targetPort: 27017
  clusterIP: None
  selector:
    role: mongo
# kubectl create -f mongodb-service.yaml
service/mongodb-service created
# kubectl get svc mongodb-service
NAME              TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE
mongodb-service   ClusterIP   None         <none>        27017/TCP   15s
```
## Tạo và triển khai StatefulSet
Với tư cách là người dùng Kubernetes , hãy xác định và triển khai StatefulSet chỉ định số lượng bản sao sẽ được sử dụng cho ứng dụng của bạn. 
Đầu tiên, tạo bí mật cho file key. MongoDB sẽ sử dụng khóa này để liên lạc với cụm nội bộ.
```sh
# openssl rand -base64 741 > key.txt
# kubectl create secret generic shared-bootstrap-data --from-file=internal-auth-mongodb-keyfile=key.txt
secret/shared-bootstrap-data created
```
Tiếp theo chúng ta cần xác định thông số kỹ thuật cho ứng dụng được chứa trong tệp StatefulSet YAML.
Đặc tả mẫu sau đây yêu cầu một phiên bản của ứng dụng MongoDB, 
chỉ định hình ảnh bên ngoài sẽ được sử dụng và tham chiếu lớp lưu trữ mongodb-sc mà bạn đã tạo trước đó.
Lớp lưu trữ này ánh xạ tới chính sách lưu trữ VM tiết kiệm không gian mà bạn đã xác định trước đó ở phía vSphere Client.

> **Lưu ý: tệp kê khai này yêu cầu nút Kubernetes có thể tiếp cận hình ảnh có tên mongo:3.4. 
> Nếu các nút Kubernetes của bạn không thể truy cập vào kho lưu trữ bên ngoài thì tệp YAML này cần được sửa đổi để truy cập vào kho lưu trữ nội bộ cục bộ của bạn. 
> Tất nhiên, repo này cũng cần chứa hình ảnh Mongo.
> Chúng tôi đã đặt số lượng bản sao thành 3, cho biết rằng sẽ có 3 Pod, 3 PVC và 3 PV được khởi tạo như một phần của StatefulSet này.

```yaml
# cat mongodb-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongod
spec:
  serviceName: mongodb-service
  replicas: 3
  selector:
    matchLabels:
      role: mongo
      environment: test
      replicaset: MainRepSet
  template:
    metadata:
      labels:
        role: mongo
        environment: test
        replicaset: MainRepSet
    spec:
      containers:
      - name: mongod-container
        image: mongo:3.4
        command:
        - "numactl"
        - "--interleave=all"
        - "mongod"
        - "--bind_ip"
        - "0.0.0.0"
        - "--replSet"
        - "MainRepSet"
        - "--auth"
        - "--clusterAuthMode"
        - "keyFile"
        - "--keyFile"
        - "/etc/secrets-volume/internal-auth-mongodb-keyfile"
        - "--setParameter"
        - "authenticationMechanisms=SCRAM-SHA-1"
        resources:
          requests:
            cpu: 0.2
            memory: 200Mi
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: secrets-volume
          readOnly: true
          mountPath: /etc/secrets-volume
        - name: mongodb-persistent-storage-claim
          mountPath: /data/db
      volumes:
      - name: secrets-volume
        secret:
          secretName: shared-bootstrap-data
          defaultMode: 256
  volumeClaimTemplates:
  - metadata:
      name: mongodb-persistent-storage-claim
      annotations:
        volume.beta.kubernetes.io/storage-class: "mongodb-sc"
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
# kubectl create -f mongodb-statefulset.yaml
statefulset.apps/mongo created

Xác minh rằng ứng dụng MongoDB đã được triển khai. Đợi các nhóm bắt đầu chạy và PVC được tạo cho mỗi bản sao.
```sh
# kubectl get statefulset mongod
NAME     READY   AGE
mongod   3/3     96s
# kubectl get pod -l role=mongo
NAME       READY   STATUS    RESTARTS   AGE
mongod-0   1/1     Running   0          13h
mongod-1   1/1     Running   0          13h
mongod-2   1/1     Running   0          13h
# kubectl get pvc
NAME                                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mongodb-persistent-storage-claim-mongod-0   Bound    pvc-ea98b22a-b8cf-11e9-b1d3-005056a0e4f0   1Gi        RWO            mongodb-sc     13h
mongodb-persistent-storage-claim-mongod-1   Bound    pvc-0267fa7d-b8d0-11e9-b1d3-005056a0e4f0   1Gi        RWO            mongodb-sc     13h
mongodb-persistent-storage-claim-mongod-2   Bound    pvc-24d86a37-b8d0-11e9-b1d3-005056a0e4f0   1Gi        RWO            mongodb-sc     13h
```
## Thiết lập cấu hình bộ bản sao Mongo

Để thiết lập cấu hình bộ bản sao Mongo, chúng ta cần kết nối với một trong các quy trình vùng chứa mongod để định cấu hình bộ bản sao.
Chạy lệnh sau để kết nối với vùng chứa đầu tiên. 
Trong shell, khởi tạo bộ bản sao. Bạn có thể dựa vào tên máy chủ giống nhau do đã sử dụng StatefulSet.
```sh
# kubectl exec -it mongod-0 -c mongod-container bash
root@mongod-0:/# mongo
MongoDB shell version v3.4.22
connecting to: mongodb://127.0.0.1:27017
MongoDB server version: 3.4.22
Welcome to the MongoDB shell.
For interactive help, type "help".
For more comprehensive documentation, see
        http://docs.mongodb.org/
Questions? Try the support group
        http://groups.google.com/group/mongodb-user
> rs.initiate({_id: "MainRepSet", version: 1, members: [
... { _id: 0, host : "mongod-0.mongodb-service.default.svc.cluster.local:27017" },
... { _id: 1, host : "mongod-1.mongodb-service.default.svc.cluster.local:27017" },
... { _id: 2, host : "mongod-2.mongodb-service.default.svc.cluster.local:27017" }
...  ]});
{ "ok" : 1 }
```
Điều này làm cho mongodb-0 trở thành nút chính và hai nút còn lại là nút phụ.

## Xác minh chức năng Cloud Native Storage đang hoạt động trong vSphere
Sau khi ứng dụng của bạn được triển khai, trạng thái của nó được hỗ trợ bởi tệp VMDK được liên kết với chính sách lưu trữ được chỉ định. 
Với tư cách là quản trị viên vSphere , bạn có thể xem lại VMDK được tạo cho khối lượng vùng chứa của mình. 
Trong bước này, chúng tôi sẽ xác minh rằng tính năng Cloud Native Storage được phát hành cùng với vSphere 6.7U3 có hoạt động hay không. 
Để truy cập giao diện người dùng CNS, hãy đăng nhập vào ứng dụng khách vSphere ,
sau đó điều hướng đến Trung tâm dữ liệu → Giám sát → Lưu trữ gốc trên nền tảng đám mây → Khối lượng vùng chứa và quan sát xem các khối lượng liên tục mới được tạo
có hiện diện hay không. Những thứ này phải khớp với kubectl get pvcđầu ra từ trước đó.
Bạn cũng có thể theo dõi trạng thái tuân thủ chính sách lưu trữ của họ.

![image](https://github.com/user-attachments/assets/dec3d5a5-6160-46da-956c-69f655c7ac89)

Thế là hoàn tất việc kiểm tra. CSI, CPI và CNS đều đang hoạt động.
