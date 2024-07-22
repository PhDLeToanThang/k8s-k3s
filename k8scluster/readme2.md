# Phần 2. Cách thiết lập cụm Kubernetes bằng Kubeadm

_Hướng dẫn từng bước để thiết lập cụm kubernetes bằng Kubeadm với một nút chính - Master và 02 (hai) nút XỬ LÝ - Worker Nodes._

Kubeadm ( https://github.com/kubernetes/kubeadm ) là một công cụ tuyệt vời để thiết lập cụm kubernetes hoạt động trong thời gian ngắn hơn. Nó thực hiện tất cả các công việc nặng nhọc trong việc thiết lập tất cả các thành phần cụm kubernetes. Ngoài ra, Nó tuân theo tất cả các phương pháp hay nhất về cấu hình cho cụm kubernetes.

## Kubeadm là gì?
Kubeadm là một công cụ để thiết lập cụm Kubernetes khả thi tối thiểu mà không cần cấu hình phức tạp. Ngoài ra, Kubeadm giúp toàn bộ quá trình trở nên dễ dàng bằng cách chạy một loạt các bước kiểm tra trước để đảm bảo rằng máy chủ có tất cả các thành phần và cấu hình cần thiết để chạy Kubernetes.

Nó được phát triển và duy trì bởi cộng đồng Kubernetes chính thức. Có các tùy chọn khác như minikube (https://devopscube.com/kubernetes-minikube-tutorial/) , kind, v.v., khá dễ cài đặt. Bạn có thể xem hướng dẫn minikube của tôi . Đó là những lựa chọn tốt với yêu cầu phần cứng tối thiểu nếu bạn đang triển khai và thử nghiệm ứng dụng trên Kubernetes.

Nhưng nếu bạn muốn thử nghiệm các thành phần cụm hoặc tiện ích thử nghiệm nằm trong quản trị cụm thì Kubeadm là lựa chọn tốt nhất. Ngoài ra, bạn có thể tạo một cụm giống như sản xuất cục bộ trên máy trạm cho mục đích phát triển và thử nghiệm.

## Điều kiện tiên quyết để thiết lập Kubeadm:

Sau đây là các điều kiện tiên quyết để **thiết lập cụm Kubeadm Kubernetes**.

1. Tối thiểu hai nút Ubuntu [Một nút chính và một nút XỬ LÝ]. Bạn có thể có nhiều nút XỬ LÝ hơn theo yêu cầu của bạn.
1. Nút chính phải có tối thiểu 2 vCPU và 2GB RAM .
1. Đối với các nút XỬ LÝ, nên sử dụng tối thiểu 1vCPU và RAM 2 GB.
1. **Phạm vi mạng 10.X.X.X/X** với IP tĩnh cho nút chính và nút XỬ LÝ. Chúng tôi sẽ sử dụng dải ipv4 **192.x.x.x làm phạm vi mạng nhóm POD** sẽ được plugin mạng Calico sử dụng. Đảm bảo dải IP nút Nodes và dải IP nhóm POD không trùng nhau.
   
**Lưu ý:** Nếu bạn đang thiết lập cụm trong mạng công ty đằng sau proxy, hãy đảm bảo đặt các biến proxy và có quyền truy cập vào sổ đăng ký vùng chứa và trung tâm docker. Hoặc nói chuyện với quản trị viên mạng của bạn để đưa danh sách trắng vào registry.k8s.io để lấy các hình ảnh được yêu cầu.

## Yêu cầu về cổng Kubeadm:
Vui lòng tham khảo hình ảnh sau đây và đảm bảo tất cả các cổng đều được phép cho bảng điều khiển (chính) và các nút XỬ LÝ. Nếu bạn đang thiết lập máy chủ đám mây cụm kubeadm, hãy đảm bảo bạn cho phép các cổng trong cấu hình tường lửa. 
![image](https://github.com/user-attachments/assets/3770f7b7-95df-4c89-92fa-9dfa9dd73823)

Nếu bạn đang sử dụng máy ảo Ubuntu vagrant-based, tường lửa sẽ bị tắt theo mặc định. Vì vậy, bạn không phải thực hiện bất kỳ cấu hình tường lửa nào.

## Kubeadm cho kỳ thi chứng chỉ Kubernetes
Nếu bạn đang chuẩn bị cho các chứng chỉ Kubernetes như **CKA, CKAD hoặc CKS**, bạn có thể sử dụng các cụm kubeadm địa phương để luyện tập cho kỳ thi chứng chỉ. Trên thực tế, bản thân kubeadm là một phần của kỳ thi CKA và CKS . Đối với CKA, bạn có thể được yêu cầu khởi động một cụm bằng Kubeadm. Đối với CKS, bạn phải nâng cấp cụm bằng kubeadm.

Nếu sử dụng máy ảo dựa trên Vagrant trên máy trạm của mình, bạn có thể khởi động và dừng cụm bất cứ khi nào bạn cần. Bằng cách có các cụm Kubeadm cục bộ, bạn có thể thử nghiệm với tất cả các cấu hình cụm và tìm hiểu cách khắc phục sự cố các thành phần khác nhau trong cụm.

> **Lưu ý quan trọng:** Nếu bạn dự định đạt chứng nhận Kubernetes, hãy sử dụng Mã phiếu giảm giá CKA/CKAD/CKS trước khi giá tăng.

## Vagrantfile, Kubeadm Tập lệnh & Bản kê khai
Ngoài ra, tất cả các lệnh được sử dụng trong hướng dẫn này dành cho cấu hình nút chính và nút XỬ LÝ đều được lưu trữ trong GitHub . Bạn có thể sao chép kho lưu trữ để tham khảo.

```sh
git clone https://github.com/techiescamp/kubeadm-scripts
```
Hướng dẫn này nhằm mục đích giúp bạn hiểu từng cấu hình cần thiết để thiết lập Kubeadm. Nếu không muốn chạy từng lệnh một, **bạn có thể chạy trực tiếp tệp script**.

Nếu bạn đang sử dụng Vagrant để thiết lập cụm Kubernetes, bạn có thể sử dụng Vagrantfile của tôi. Nó khởi chạy 3 máy ảo. Một Vagrantfile cơ bản tự giải thích. Nếu bạn chưa quen với Vagrant, hãy xem hướng dẫn của Vagrant (https://devopscube.com/vagrant-tutorial-beginners).

Nếu bạn là người dùng Terraform và AWS, bạn có thể sử dụng tập lệnh Terraform có trong thư mục Terraform để tạo các phiên bản ec2 (https://devopscube.com/use-aws-cli-create-ec2-instance) .

Ngoài ra, tôi đã tạo một **video demo về toàn bộ quá trình thiết lập kubeadm**. Bạn có thể tham khảo nó trong quá trình thiết lập.
Setup Kubernetes Cluster Using Kubeadm [Multi-node]: https://www.youtube.com/watch?v=xX52dc3u2HU 

## Thiết lập cụm Kubernetes bằng Kubeadm

Sau đây là các bước cấp cao liên quan đến việc thiết lập cụm Kubernetes dựa trên kubeadm:

1. Cài đặt thời gian chạy vùng chứa trên tất cả các nút- Chúng tôi sẽ sử dụng cri-o .
1. Cài đặt Kubeadm, Kubelet và kubectl trên tất cả các nút.
1. Bắt đầu cấu hình mặt phẳng điều khiển Kubeadm trên nút chính.
1. Lưu lệnh nối nút bằng mã thông báo.
1. Cài đặt plugin mạng Calico (nhà điều hành).
1. Nối nút XỬ LÝ với nút chính (mặt phẳng điều khiển) bằng lệnh nối.
1. Xác thực tất cả các thành phần và nút cụm.
1. Cài đặt máy chủ số liệu Kubernetes
1. Triển khai một ứng dụng mẫu và xác thực ứng dụng
1. Tất cả các bước được đưa ra trong hướng dẫn này đều được tham khảo từ tài liệu chính thức của Kubernetes và các trang dự án GitHub có liên quan.

Nếu bạn muốn hiểu chi tiết từng thành phần cụm, hãy tham khảo Kiến trúc Kubernetes toàn diện .

Bây giờ hãy bắt đầu với việc thiết lập.

### Bước 1: Kích hoạt lưu lượng cầu nối iptables trên tất cả các nút
Thực hiện các lệnh sau trên tất cả các nút cho IPtables để xem lưu lượng được bắc cầu. Ở đây chúng tôi đang điều chỉnh một số tham số kernel và thiết lập chúng bằng sysctl.
```sh
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

### Bước 2: Vô hiệu hóa trao đổi trên tất cả các Nút:

Để kubeadm hoạt động bình thường, bạn cần tắt tính năng trao đổi trên tất cả các nút bằng lệnh sau.
```sh
sudo swapoff -a
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
```
Mục này fstabsẽ đảm bảo tính năng trao đổi bị tắt khi khởi động lại hệ thống.
Bạn cũng có thể kiểm soát lỗi hoán đổi bằng tham số kubeadm mà --ignore-preflight-errors Swapchúng ta sẽ xem xét ở phần sau.

> **Lưu ý:** Từ 1.28 kubeadm đã hỗ trợ beta cho việc sử dụng trao đổi với cụm kubeadm. Đọc cái này để hiểu thêm.

### Bước 3: Cài đặt CRI-O Runtime trên tất cả các nút:

> **Lưu ý:** Thay vào đó, chúng tôi đang sử dụng cri-o nếu được chứa vì trong các kỳ thi chứng chỉ Kubernetes , cri-o được sử dụng làm thời gian chạy vùng chứa trong các cụm kỳ thi.

Yêu cầu cơ bản đối với cụm Kubernetes là thời gian chạy vùng chứa . Bạn có thể có bất kỳ thời gian chạy vùng chứa nào sau đây:
1. CRI-O
1. thùng chứa
1. Docker Engine (sử dụng cri-dockerd)
Chúng tôi sẽ sử dụng CRI-O thay vì Docker cho thiết lập này vì công cụ Docker không được dùng nữa của Kubernetes

Thực hiện các lệnh sau trên tất cả các nút để cài đặt các phần phụ thuộc cần thiết và phiên bản CRIO mới nhất:
```sh
sudo apt-get update -y
sudo apt-get install -y software-properties-common curl apt-transport-https ca-certificates
curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/ /" |
    tee /etc/apt/sources.list.d/cri-o.list
sudo apt-get update -y
sudo apt-get install -y cri-o
sudo systemctl daemon-reload
sudo systemctl enable crio --now
sudo systemctl start crio.service
```
Cài đặt crictl.
```sh
VERSION="v1.28.0"
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz
```
crictl, một tiện ích CLI để tương tác với các vùng chứa được tạo bởi thời gian chạy vùng chứa.
Khi sử dụng thời gian chạy vùng chứa không phải Docker, bạn có thể sử dụng tiện ích crictl để gỡ lỗi vùng chứa trên nút. Ngoài ra, nó rất hữu ích trong chứng nhận CKS khi bạn cần gỡ lỗi các vùng chứa.

### Bước 4: Cài đặt Kubeadm & Kubelet & Kubectl trên tất cả các Node:
Tải xuống khóa GPG cho kho lưu trữ Kubernetes APT trên tất cả các nút.
```sh
KUBERNETES_VERSION=1.29
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Cập nhật kho apt
```sh
sudo apt-get update -y
```
> **Lưu ý:** Nếu bạn đang chuẩn bị lấy chứng chỉ Kubernetes, hãy cài đặt phiên bản kubernetes cụ thể. Ví dụ: phiên bản Kubernetes hiện tại dành cho các kỳ thi CKA, CKAD và CKS là Kubernetes phiên bản 1.29

Bạn có thể sử dụng các lệnh sau để tìm phiên bản mới nhất. Cài đặt phiên bản đầu tiên trong 1.29 để bạn có thể thực hành nhiệm vụ nâng cấp cụm:
```sh
apt-cache madison kubeadm | tac
```
Chỉ định phiên bản như hiển thị bên dưới. Ở đây tôi đang sử dụng1.29.0-1.1
```sh
sudo apt-get install -y kubelet=1.29.0-1.1 kubectl=1.29.0-1.1 kubeadm=1.29.0-1.1
```
Hoặc, để cài đặt phiên bản mới nhất từ ​​repo, hãy sử dụng lệnh sau mà không chỉ định bất kỳ phiên bản nào.
```sh
sudo apt-get install -y kubelet kubeadm kubectl
```
Thêm giữ vào các gói để ngăn chặn nâng cấp.
```sh
sudo apt-mark hold kubelet kubeadm kubectl
```
Bây giờ chúng ta có tất cả các tiện ích và công cụ cần thiết để định cấu hình các thành phần Kubernetes bằng kubeadm.
Thêm IP nút vào KUBELET_EXTRA_ARGS.
```sh
sudo apt-get install -y jq
local_ip="$(ip --json addr show eth0 | jq -r '.[0].addr_info[] | select(.family == "inet") | .local')"
cat > /etc/default/kubelet << EOF
KUBELET_EXTRA_ARGS=--node-ip=$local_ip
EOF
```
### Bước 5: Khởi tạo Kubeadm trên Master Node để thiết lập Control Plane

Ở đây bạn cần xem xét hai lựa chọn:
1. **Nút chính có IP riêng:** Nếu bạn có các nút chỉ có địa chỉ IP riêng thì máy chủ API sẽ được truy cập qua IP riêng của nút chính.
1. **Nút chính có IP công cộng:** Nếu bạn đang thiết lập cụm Kubeadm trên nền tảng Đám mây và bạn cần quyền truy cập vào máy chủ Api chính qua IP công cộng của máy chủ nút chính.
Chỉ có lệnh khởi tạo Kubeadm khác nhau đối với IP Công khai và Riêng tư.
Chỉ thực hiện các lệnh trong phần này trên nút chính.
Nếu bạn đang sử dụng IP riêng cho Nút chính,

Đặt các biến môi trường sau. Thay thế 10.0.0.10 bằng IP của nút chính của bạn:
```sh
IPADDR="10.0.0.10"
NODENAME=$(hostname -s)
POD_CIDR="192.168.0.0/16"
```
Nếu bạn muốn sử dụng IP công cộng của nút chính,
Đặt các biến môi trường sau. Biến IPADDR sẽ được tự động đặt thành IP công cộng của máy chủ bằng ifconfig.melệnh gọi cuộn. Bạn cũng có thể thay thế nó bằng địa chỉ IP công cộng
```sh
IPADDR=$(curl ifconfig.me && echo "")
NODENAME=$(hostname -s)
POD_CIDR="192.168.0.0/16"
```
Bây giờ, hãy khởi tạo cấu hình mặt phẳng điều khiển nút chính bằng lệnh kubeadm.
Để thiết lập dựa trên địa chỉ IP riêng, hãy sử dụng lệnh init sau:
```sh
sudo kubeadm init --apiserver-advertise-address=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=$POD_CIDR --node-name $NODENAME --ignore-preflight-errors Swap
--ignore-preflight-errors Swapthực sự không bắt buộc vì ban đầu chúng tôi đã vô hiệu hóa tính năng hoán đổi.
```

Để thiết lập dựa trên địa chỉ IP công cộng, hãy sử dụng lệnh init sau.
Ở đây thay vì --apiserver-advertise-addresschúng tôi sử dụng --control-plane-endpointtham số cho điểm cuối của máy chủ API.
```sh
sudo kubeadm init --control-plane-endpoint=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=$POD_CIDR --node-name $NODENAME --ignore-preflight-errors Swap
```
Tất cả các bước khác cũng giống như định cấu hình nút chính bằng IP riêng.
Khi khởi tạo kubeadm thành công, bạn sẽ nhận được đầu ra có vị trí tệp kubeconfig và lệnh nối với mã thông báo như hiển thị bên dưới. Sao chép nó và lưu nó vào tập tin. chúng ta sẽ cần nó để nối nút XỬ LÝ với nút chính.

![image](https://github.com/user-attachments/assets/10af7e21-805d-4bcb-9fbe-7a749734a7d8)

Sử dụng các lệnh sau **từ đầu ra** để tạo kubeconfigbản chính để bạn có thể sử dụng kubectlđể tương tác với API cụm.
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Bây giờ, hãy xác minh kubeconfig bằng cách thực hiện lệnh kubectl sau để liệt kê tất cả các nhóm trong kube-systemkhông gian tên.
```sh
kubectl get po -n kube-system
```
Bạn sẽ thấy đầu ra sau đây. Bạn sẽ thấy hai **nhóm Coredns pods ở trạng thái chờ xử lý**. Đó là hành vi mong đợi. Khi chúng tôi **cài đặt plugin mạng**, nó sẽ ở trạng thái chạy.

![image](https://github.com/user-attachments/assets/0fa0e062-57e3-4a16-a962-ae8ec1779973)

Bạn xác minh tất cả các trạng thái sức khỏe của thành phần cụm bằng lệnh sau.
```sh
kubectl get --raw='/readyz?verbose'
```
Bạn có thể lấy thông tin cụm bằng lệnh sau.
```sh
kubectl cluster-info
```
Theo mặc định, các ứng dụng sẽ không được lên lịch trên nút chính. Nếu bạn muốn sử dụng nút chính để lên lịch ứng dụng , hãy làm hỏng nút chính.
```sh
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```
**Lưu ý:** Bạn cũng có thể chuyển cấu hình kubeadm dưới dạng tệp khi khởi tạo cụm. Xem Kubeadm Init với file config

### Bước 6: Tham gia các nút XỬ LÝ vào nút chính Kubernetes

Chúng tôi cũng đã thiết lập các tiện ích cri-o, kubelet và kubeadm trên các nút XỬ LÝ.
Bây giờ, hãy kết nối nút XỬ LÝ với nút chính bằng lệnh nối Kubeadm mà bạn có ở đầu ra khi thiết lập nút chính.
Nếu bạn lỡ sao chép lệnh nối, hãy thực hiện lệnh sau trong nút chính để tạo lại mã thông báo bằng lệnh nối.
```sh
kubeadm token create --print-join-command
```
Đây là những gì lệnh trông như thế nào. Sử dụng sudonếu bạn chạy như một người dùng bình thường. Lệnh này thực hiện khởi động TLS cho các nút ( https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/ ).
```sh
sudo kubeadm join 10.128.0.37:6443 --token j4eice.33vgvgyf5cxw4u8i \
    --discovery-token-ca-cert-hash sha256:37f94469b58bcc8f26a4aa44441fb17196a585b37288f85e22475b00c36f1c61
```
Khi thực hiện thành công, bạn sẽ thấy kết quả có nội dung: “Nút này đã tham gia vào cụm”.

![image](https://github.com/user-attachments/assets/3616731b-f36a-4c76-8eaf-39ba16087d9f)

Bây giờ thực thi lệnh kubectl từ nút chính để kiểm tra xem nút đó có được thêm vào nút chính hay không.
```sh
kubectl get nodes
```
Đầu ra ví dụ,
```sh
root@controlplane:~# kubectl get nodes
```
```table
NAME           STATUS   ROLES           AGE     VERSION
controlplane   Ready    control-plane   8m42s   v1.29.0
node01         Ready    worker          2m6s    v1.29.0
```
Trong lệnh trên, ROLE dành <none>cho các nút XỬ LÝ. Bạn có thể thêm nhãn vào nút worker bằng lệnh sau. Thay thế worker-node01bằng tên máy chủ của nút XỬ LÝ mà bạn muốn gắn nhãn.
```sh
kubectl label node node01  node-role.kubernetes.io/worker=worker
```
Bạn có thể thêm nhiều nút hơn bằng lệnh nối tương tự.

### Bước 7: Cài đặt Calico Network Plugin cho Pod Networking:

Kubeadm không cấu hình bất kỳ plugin mạng nào. Bạn cần cài đặt plugin mạng mà bạn chọn cho mạng kubernetes pod và kích hoạt chính sách mạng.
Tôi đang sử dụng plugin mạng Calico cho thiết lập này.

> **Lưu ý:** Đảm bảo bạn thực thi lệnh kubectl từ nơi bạn đã định cấu hình kubeconfigtệp. Từ máy trạm chính của bạn có khả năng kết nối với API kubernetes.

Thực hiện các lệnh sau để cài đặt toán tử plugin mạng Calico trên cụm:
```sh
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```
Sau vài phút, nếu bạn kiểm tra các nhóm trong kube-systemkhông gian tên, bạn sẽ thấy các nhóm calico và các nhóm CoreDNS đang chạy.
```sh
kubectl get po -n kube-system
```
![image](https://github.com/user-attachments/assets/1f30d356-ba12-4004-9212-cf23a4cd9942)

### Bước 8: Thiết lập máy chủ số liệu Kubernetes

Kubeadm không cài đặt thành phần máy chủ số liệu trong quá trình khởi tạo. Chúng ta phải cài đặt nó một cách riêng biệt.
Để xác minh điều này, nếu chạy lệnh top, bạn sẽ thấy lỗi Metrics API not available.
```sh
root@controlplane:~# kubectl top nodes

error: Metrics API not available
```
Để cài đặt máy chủ số liệu, hãy thực thi tệp kê khai máy chủ số liệu sau. Nó triển khai phiên bản máy chủ số liệuv0.6.2:
```sh
kubectl apply -f https://raw.githubusercontent.com/techiescamp/kubeadm-scripts/main/manifests/metrics-server.yaml
```
Bảng kê khai này được lấy từ repo r phục vụ số liệu chính thức . Tôi đã thêm --kubelet-insecure-tlscờ vào vùng chứa để làm cho nó hoạt động trong thiết lập cục bộ và lưu trữ riêng biệt. Hoặc nếu không, bạn sẽ gặp lỗi sau.
```sh
 because it doesn't contain any IP SANs" node=""
```
Sau khi triển khai các đối tượng máy chủ số liệu, bạn sẽ mất một phút để xem các số liệu nút và nhóm bằng lệnh trên cùng.
```sh
kubectl top nodes
```
Bạn sẽ có thể xem số liệu của nút như hiển thị bên dưới.
```sh
root@controlplane:~# kubectl top nodes
```
```table
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
controlplane   142m         7%     1317Mi          34%
node01         36m          1%     915Mi           23%
```
Bạn cũng có thể xem số liệu CPU và bộ nhớ của nhóm bằng lệnh sau.
```sh
kubectl top pod -n kube-system
```
### Bước 9: Triển khai ứng dụng Nginx mẫu

Bây giờ chúng ta đã có tất cả các thành phần để làm cho cụm và ứng dụng hoạt động, hãy triển khai một ứng dụng Nginx mẫu và xem liệu chúng ta có thể truy cập nó qua NodePort không
Tạo triển khai Nginx . Thực hiện các thao tác sau trực tiếp trên dòng lệnh. Nó triển khai nhóm trong không gian tên mặc định.
```sh
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80      
EOF
```
Hiển thị triển khai Nginx trên NodePort 32000
```sh
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector: 
    app: nginx
  type: NodePort  
  ports:
    - port: 80
      targetPort: 80
      nodePort: 32000
EOF
```
Kiểm tra trạng thái nhóm bằng lệnh sau:
```sh
kubectl get pods
```
Sau khi quá trình triển khai hoàn tất, bạn sẽ có thể truy cập trang chủ Nginx trên NodePort được phân bổ.

Ví dụ:

![image](https://github.com/user-attachments/assets/0d91b521-a323-41f8-9a91-f79e34f9051b)

### Bước 10: Thêm cấu hình Kubeadm vào máy trạm
Nếu muốn kết nối cụm Kubeadm bằng kubectl từ máy trạm của mình, bạn có thể hợp nhất kubeadm admin.conf với tệp kubeconfig hiện có của mình.
Thực hiện theo các bước dưới đây để cấu hình:

Bước 10.1: Sao chép nội dung admin.conftừ nút mặt phẳng điều khiển và lưu nó vào tệp có tên kubeadm-config.yamltrong máy trạm của bạn.

Bước 10.2: Sao lưu kubeconfig hiện có:
```sh
cp ~/.kube/config ~/.kube/config.bak
```
Bước 10.3: Hợp nhất cấu hình mặc định với kubeadm-config.yaml và xuất nó sang biến KUBECONFIG
```sh
export KUBECONFIG=~/.kube/config:/path/to/kubeadm-config.yaml
```
Bước 10.4: Hợp nhất các cấu hình thành một tập tin
```sh
kubectl config view --flatten > ~/.kube/merged_config.yaml
```
Bước 10.5: Thay config cũ bằng config mới
```sh
mv ~/.kube/merged_config.yaml ~/.kube/config
```
Bước 10.6: Liệt kê tất cả các bối cảnh
```sh
kubectl config get-contexts -o name
```
Bước 10.7: Đặt bối cảnh hiện tại cho cụm kubeadm.
```sh
kubectl config use-context <cluster-name-here>
```
Bây giờ, bạn sẽ có thể kết nối với cụm Kubeadm từ tiện ích kubectl trên máy trạm cục bộ của mình.

### Các vấn đề Kubeadm có thể xảy ra:

Sau đây là những vấn đề có thể xảy ra mà bạn có thể gặp phải trong quá trình thiết lập kubeadm:

1. **Pod Hết bộ nhớ và CPU:** Nút chính phải có tối thiểu 2vCPU và bộ nhớ 2 GB.
1. **Các nút không thể kết nối với Master:** Kiểm tra tường lửa giữa các nút và đảm bảo tất cả các nút có thể giao tiếp với nhau trên các cổng kubernetes cần thiết.
1. **Calico Pod Khởi động lại:** Đôi khi, nếu bạn sử dụng cùng một dải IP cho mạng nút và nhóm, nhóm Calico có thể không hoạt động như mong đợi.
Vì vậy, hãy đảm bảo phạm vi IP của nút và nhóm không trùng nhau. Địa chỉ IP chồng chéo (https://devopscube.com/ip-address-tutorial/ )cũng có thể gây ra sự cố cho các ứng dụng khác đang chạy trên cụm.
Đối với các lỗi nhóm khác, hãy xem hướng dẫn khắc phục sự cố nhóm kubernetes ( https://devopscubecom/troubleshoot-kubernetes-pods/ ).

Nếu máy chủ của bạn không có tối thiểu 2 vCPU, bạn sẽ gặp lỗi sau:
```error
[ERROR NumCPU]: the number of available CPUs 1 is less than the required 2
Nếu bạn sử dụng IP công cộng có --apiserver-advertise-addresstham số, bạn sẽ gặp lỗi các thành phần nút chính với lỗi sau. Để khắc phục lỗi này, hãy sử dụng --control-plane-endpointtham số có địa chỉ IP công cộng.

kubelet-check] Initial timeout of 40s passed.

Unfortunately, an error has occurred:
        timed out waiting for the condition

This error is likely caused by:
        - The kubelet is not running
        - The kubelet is unhealthy due to a misconfiguration of the node in some way (required cgroups disabled)

If you are on a systemd-powered system, you can try to troubleshoot the error with the following commands:
        - 'systemctl status kubelet'
        - 'journalctl -xeu kubelet'
```
Bạn sẽ gặp lỗi sau trong các nút XỬ LÝ khi cố gắng tham gia nút XỬ LÝ bằng mã thông báo mới sau khi đặt lại nút chính. 
Để khắc phục lỗi này, hãy đặt lại nút worker bằng lệnh
```sh
kubeadm reset.
```
```error
[ERROR FileAvailable--etc-kubernetes-kubelet.conf]: /etc/kubernetes/kubelet.conf already exists
        [ERROR Port-10250]: Port 10250 is in use
        [ERROR FileAvailable--etc-kubernetes-pki-ca.crt]: /etc/kubernetes/pki/ca.crt already exists
```

### Cấu hình quan trọng của cụm Kubernetes

Sau đây là các cấu hình cụm Kubernetes (https://devopscube.com/kubernetes-cluster-configurations/ ) quan trọng mà bạn nên biết:

Cấu hình	         Vị trí
Vị trí nhóm tĩnh (etcd, api-server, trình quản lý bộ điều khiển và bộ lập lịch)	/etc/kubernetes/tệp kê khai
Vị trí chứng chỉ TLS (kubernetes-ca, etcd-ca và kubernetes-front-proxy-ca)	/etc/kubernetes/pki
Tệp quản trị Kubeconfig	/etc/kubernetes/admin.conf
Cấu hình Kubelet	/var/lib/kubelet/config.yaml

![image](https://github.com/user-attachments/assets/ff916181-4ce3-4081-a523-c71deaf1039a)

Có các cấu hình là một phần của cổng tính năng Kubernetes. Nếu muốn sử dụng các tính năng là một phần của cổng tính năng, bạn cần bật chúng trong quá trình khởi tạo Kubeadm bằng tệp cấu hình kubeadm.

Bạn có thể tham khảo cách bật cổng tính năng trong blog Kubeadm để hiểu rõ hơn.

### Nâng cấp cụm Kubeadm
Sử dụng Kubeadm, bạn có thể nâng cấp cụm kubernetes cho cùng bản vá phiên bản hoặc phiên bản mới.
Nâng cấp Kubeadm không gây ra bất kỳ thời gian ngừng hoạt động nào nếu bạn nâng cấp từng nút một.
Để thực hành, vui lòng tham khảo hướng dẫn từng bước của tôi về nâng cấp cụm Kubeadm

### Sao lưu dữ liệu ETCD
Sao lưu etcd là một trong những nhiệm vụ quan trọng trong các dự án thực tế và chứng nhận CKA.
Bạn có thể làm theo hướng dẫn sao lưu etcd để tìm hiểu cách thực hiện sao lưu và khôi phục etcd.

### Thiết lập giám sát Prometheus
Bước tiếp theo, bạn có thể thử thiết lập ngăn xếp giám sát Prometheus trên cụm Kubeadm.
Tôi đã xuất bản một hướng dẫn chi tiết để thiết lập. Tham khảo prometheus trên hướng dẫn Kubernetes để biết hướng dẫn từng bước. Ngăn xếp chứa prometheus, trình quản lý cảnh báo, số liệu trạng thái kube và Grafana.

### Kubeadm hoạt động như thế nào?
Đây là cách thiết lập Kubeadm hoạt động.

Khi bạn khởi tạo cụm Kubernetes bằng Kubeadm, nó sẽ thực hiện như sau:

1. Khi bạn khởi tạo kubeadm, trước tiên, kubeadm sẽ chạy tất cả các bước kiểm tra trước để xác thực trạng thái hệ thống và tải xuống tất cả các hình ảnh vùng chứa cụm cần thiết từ sổ đăng ký vùng chứa **register.k8s.io**.
1. Sau đó, nó tạo các chứng chỉ TLS cần thiết và lưu trữ chúng trong thư mục **/etc/kubernetes/pki**.
1. Tiếp theo, nó tạo tất cả các tệp kubeconfig cho các thành phần cụm trong thư mục **/etc/kubernetes**.
1. Sau đó, dịch vụ kubelet khởi động tạo các tệp kê khai nhóm tĩnh cho tất cả các thành phần cụm và lưu nó vào thư mục **/etc/kubernetes/manifests**.
1. Tiếp theo, nó khởi động tất cả các thành phần mặt phẳng điều khiển từ các bảng kê khai nhóm tĩnh.
1. Sau đó, nó cài đặt các thành phần DNS và Kubeproxy cốt lõi
1. Cuối cùng, nó tạo ra mã thông báo khởi động nút.
1. Các nút XỬ LÝ sử dụng mã thông báo này để tham gia mặt phẳng điều khiển.

![image](https://github.com/user-attachments/assets/3d03300b-b7f9-489e-b273-e96ba9034185)
_Quy trình làm việc Kubeadm_

Như bạn có thể thấy, tất cả các cấu hình cụm khóa sẽ có trong thư mục /etc/kubernetes.

## Câu hỏi thường gặp về Kubeadm

**1. Làm cách nào để sử dụng Chứng chỉ CA tùy chỉnh với Kubeadm?**

Theo mặc định, kubeadm tạo chứng chỉ CA của riêng mình. Tuy nhiên, nếu bạn muốn sử dụng chứng chỉ CA tùy chỉnh, chúng phải được đặt trong **/etc/kubernetes/pki** thư mục. Khi kubeadm được chạy, nó sẽ sử dụng các chứng chỉ hiện có nếu chúng được tìm thấy và sẽ không ghi đè lên chúng.

**2. Làm cách nào để tạo lệnh Kubeadm Join?**

Bạn có thể sử dụng kubeadm token create --print-join-commandlệnh để tạo lệnh nối.

## Phần kết luận
Trong bài đăng này, chúng ta đã tìm hiểu cách cài đặt Kubernetes từng bước bằng kubeadm.

- Là một kỹ sư DevOps, việc hiểu biết về các thành phần của cụm Kubernetes là điều cần thiết. Với các công ty sử dụng dịch vụ Kubernetes được quản lý , chúng tôi bỏ lỡ việc tìm hiểu các khối xây dựng cơ bản của kubernetes.

- Thiết lập Kubeadm này rất phù hợp để học và triển khai với kubernetes.

- Ngoài ra, còn có nhiều cấu hình Kubeadm khác mà tôi không đề cập trong hướng dẫn này vì nó nằm ngoài phạm vi của hướng dẫn này. Vui lòng tham khảo tài liệu chính thức của Kubeadm . Bằng cách thiết lập toàn bộ cụm trong máy ảo, bạn có thể tìm hiểu tất cả các cấu hình thành phần cụm và khắc phục sự cố của cụm về lỗi thành phần.
