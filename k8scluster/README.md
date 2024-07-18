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

Trước tiên, chúng ta hãy xem xét từng thành phần của mặt phẳng điều khiển và các khái niệm quan trọng đằng sau mỗi thành phần:

### 1. kube-apiserver:

Máy chủ kube-apiserver là trung tâm của cụm Kubernetes hiển thị API Kubernetes. Nó có khả năng mở rộng cao và có thể xử lý số lượng lớn yêu cầu đồng thời.
Người dùng cuối và các thành phần cụm khác giao tiếp với cụm thông qua máy chủ API. Rất hiếm khi hệ thống giám sát và dịch vụ của bên thứ ba có thể giao tiếp với máy chủ API để tương tác với cụm.

Vì vậy, khi bạn sử dụng kubectl để quản lý cụm, ở phần phụ trợ, bạn thực sự đang giao tiếp với máy chủ API thông qua API HTTP REST . Tuy nhiên, các thành phần cụm bên trong như bộ lập lịch 'scheduler', bộ điều khiển 'controller', v.v. sẽ giao tiếp với máy chủ API bằng gRPC (https://grpc.io/docs/what-is-grpc/introduction/) .

Giao tiếp giữa máy chủ API và các thành phần khác trong cụm diễn ra qua TLS để ngăn chặn truy cập trái phép vào cụm.

![image](https://github.com/user-attachments/assets/924ca4a5-3698-4211-a21a-fbbd10863b97)

_Kiến trúc Kubernetes - giải thích về kube-apiserver_

#### Kubernetes api-server chịu trách nhiệm về những điều sau:

1. Quản lý API : Hiển thị điểm cuối API cụm và xử lý tất cả các yêu cầu API. API là phiên bản và nó hỗ trợ nhiều phiên bản API cùng một lúc.
1. Xác thực (Sử dụng chứng chỉ ứng dụng khách, mã thông báo mang và Xác thực cơ bản HTTP) và Ủy quyền (đánh giá ABAC và RBAC).
1. Xử lý các yêu cầu API và xác thực dữ liệu cho các đối tượng API như nhóm, dịch vụ, v.v. (Bộ điều khiển nhập học xác thực và đột biến).
1. Nó là thành phần duy nhất giao tiếp với etcd.
1. api-server điều phối tất cả các quy trình giữa mặt phẳng điều khiển và các thành phần nút workers.
1. api-server có proxy apiserver tích hợp sẵn . Nó là một phần của quy trình máy chủ API. Nó chủ yếu được sử dụng để cho phép truy cập vào các dịch vụ ClusterIP từ bên ngoài cụm, mặc dù các dịch vụ này thường chỉ có thể truy cập được trong chính cụm đó.
1. Máy chủ API cũng có một lớp tổng hợp cho phép bạn mở rộng API Kubernetes để tạo các tài nguyên và bộ điều khiển API tùy chỉnh.
1. Máy chủ API cũng hỗ trợ xem tài nguyên để biết các thay đổi. Ví dụ: khách hàng có thể thiết lập chế độ theo dõi trên các tài nguyên cụ thể và nhận thông báo theo thời gian thực khi các tài nguyên đó được tạo, sửa đổi hoặc xóa
