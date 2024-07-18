# kiến ​​trúc Kubernetes Cluster Full Outline:
_nếu bạn đang tìm kiếm:_

**Mục lục:**
1. Hiểu kiến ​​trúc của Kubernetes
2. Nắm bắt các khái niệm cơ bản về Kubernetes
3. Tìm hiểu về các thành phần kiến ​​trúc Kubernetes
4. Khám phá quy trình làm việc kết nối các thành phần này
Sau đó, bạn sẽ thấy hướng dẫn về kiến ​​trúc Kubernetes này là quan trọng.

# Kiến trúc Kubernetes là gì?

_Sơ đồ kiến ​​trúc Kubernetes sau đây hiển thị tất cả các thành phần của cụm Kubernetes và cách các hệ thống bên ngoài kết nối với cụm Kubernetes._

![image](https://github.com/user-attachments/assets/9c3d47f7-e16a-47d7-a356-11d8289be6ba)

Điều đầu tiên và quan trọng nhất bạn nên hiểu về Kubernetes là nó là một hệ thống phân tán. 
Có nghĩa là nó có nhiều thành phần trải rộng trên các máy chủ khác nhau qua mạng. 
Những máy chủ này có thể là máy ảo hoặc máy chủ vật lý. Chúng ta gọi nó là **cụm Kubernetes - viết tắt: K8s Cluster**.

**Cụm Kubernetes** bao gồm các nút có trong **Bảng điều khiển - Control panel: viết tắt: cc** và các nút Xử lý **Nút Xử lý - Node Workers**.

## Bảng điều khiển - cc

Bảng điều khiển chịu trách nhiệm điều phối vùng chứa và duy trì trạng thái mong muốn của cụm K8s. Nó có các thành phần sau:

1. kube-apiserver (máy chủ kube)
1. etcd
1. kube-scheduler (lập lịch kube)
1. kube-controller-manager (quản lý bộ điều khiển kube)
1. cloud-controller-manager (trình quản lý bộ điều khiển đám mây)

Một cụm có thể có một hoặc nhiều nút Bảng điều khiển "control plane nodes".

## Nút Xử lý - node workers

Các nút Worker chịu trách nhiệm chạy các ứng dụng được chứa trong container. Nút Xử lý có các thành phần sau:

1. kubelet
1. kube-proxy
1. Container runtime

## Các thành phần có trong Bảng điều khiển Kubernetes:

