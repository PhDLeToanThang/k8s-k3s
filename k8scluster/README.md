# Kiến ​​trúc Kubernetes Cluster Full Outline:
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
1. api-server có proxy apiserver tích hợp sẵn . Nó là một phần của quy trình máy chủ API.
Nó chủ yếu được sử dụng để cho phép truy cập vào các dịch vụ ClusterIP từ bên ngoài cụm, mặc dù các dịch vụ này thường chỉ có thể truy cập được trong chính cụm đó.
1. Máy chủ API cũng có một lớp tổng hợp cho phép bạn mở rộng API Kubernetes để tạo các tài nguyên và bộ điều khiển API tùy chỉnh.
1. Máy chủ API cũng hỗ trợ xem tài nguyên để biết các thay đổi. Ví dụ: khách hàng có thể thiết lập chế độ theo dõi trên các tài nguyên cụ thể và nhận thông báo theo thời gian thực khi các tài nguyên đó được tạo, sửa đổi hoặc xóa

### 2. etcd:
Kubernetes là một hệ thống phân tán và nó cần một cơ sở dữ liệu phân tán hiệu quả như etcd để hỗ trợ tính chất phân tán của nó.
Nó hoạt động như một khám phá dịch vụ phụ trợ và cơ sở dữ liệu. Bạn có thể gọi nó là bộ não của cụm Kubernetes.

etcd là một kho lưu trữ khóa-giá trị phân tán, nhất quán mã nguồn mở. Vì vậy, nó có nghĩa gì?

**1. Tính nhất quán cao:** Nếu một bản cập nhật được thực hiện cho một nút, tính nhất quán cao sẽ đảm bảo nó được cập nhật cho tất cả các nút khác trong cụm ngay lập tức. 
Ngoài ra, nếu bạn nhìn vào định lý CAP , việc đạt được tính khả dụng 100% với tính nhất quán cao và & Dung sai phân vùng là không thể.
**1. Phân phối:** etcd được thiết kế để chạy trên nhiều nút dưới dạng cụm mà không làm mất đi tính nhất quán.
**1. Kho lưu trữ giá trị khóa:** Cơ sở dữ liệu phi quan hệ lưu trữ dữ liệu dưới dạng khóa và giá trị.
Nó cũng hiển thị API khóa-giá trị. 
Kho dữ liệu được xây dựng dựa trên BboltDB , một nhánh của BoltDB.

_etcd sử dụng thuật toán đồng thuận bè để có tính nhất quán và tính sẵn sàng cao. 
Nó hoạt động theo kiểu thành viên dẫn đầu để có tính sẵn sàng cao và chống lại các lỗi nút._

## Vậy etcd hoạt động như thế nào với Kubernetes?

Nói một cách đơn giản, khi bạn sử dụng kubectl để lấy thông tin chi tiết về đối tượng kubernetes, bạn sẽ lấy nó từ etcd. 
Ngoài ra, khi bạn triển khai một đối tượng như nhóm, một mục nhập sẽ được tạo trong etcd.

Tóm lại, đây là những gì bạn cần biết về etcd.

etcd lưu trữ tất cả các cấu hình, trạng thái và siêu dữ liệu của các đối tượng Kubernetes (nhóm, bí mật, daemonsets, triển khai , configmaps, statefulsets, v.v.).
etcd cho phép khách hàng đăng ký các sự kiện bằng Watch() API. 
Kubernetes api-server sử dụng chức năng theo dõi của etcd để theo dõi sự thay đổi trạng thái của một đối tượng.
etcd hiển thị API khóa-giá trị bằng cách sử dụng gRPC . 
Ngoài ra, cổng gRPC là một proxy RESTful chuyển tất cả lệnh gọi API HTTP thành thông báo gRPC. 
Điều này làm cho nó trở thành cơ sở dữ liệu lý tưởng cho Kubernetes.
etcd lưu trữ tất cả các đối tượng trong khóa thư mục /registry ở định dạng khóa-giá trị. 

_Ví dụ: thông tin về nhóm có tên Nginx trong không gian tên mặc định có thể được tìm thấy trong /registry/pod/default/nginx_

![image](https://github.com/user-attachments/assets/48c0ec6b-4900-44a2-86cc-1046031b5791)

_Ngoài ra, etcd nó là thành phần Statefulset duy nhất trong mặt phẳng điều khiển._

### 3. kube-scheduler:

Bộ lập lịch kube chịu trách nhiệm lập lịch cho các nhóm Kubernetes trên các nút công nhân.

Khi triển khai một nhóm, bạn chỉ định các yêu cầu về nhóm như CPU, bộ nhớ, mối quan hệ, vết bẩn hoặc dung sai, mức độ ưu tiên, khối lượng liên tục (PV), v.v. Nhiệm vụ chính của bộ lập lịch là xác định yêu cầu tạo và chọn nút tốt nhất cho nhóm nhóm đáp ứng được yêu cầu.

Hình ảnh sau đây hiển thị tổng quan cấp cao về cách hoạt động của bộ lập lịch.

![image](https://github.com/user-attachments/assets/184dde4f-239b-4d82-83a3-28b32a8d2dd6)

_sơ đồ quy trình làm việc của bộ lập lịch kubernetes cấp cao._

Trong cụm Kubernetes, sẽ có nhiều nút công nhân. Vậy làm thế nào để bộ lập lịch chọn nút trong số tất cả các nút Workers?

Đây là cách bộ lập lịch hoạt động.

Để chọn nút tốt nhất, bộ lập lịch Kube sử dụng các hoạt động lọc và tính điểm.
Trong quá trình lọc , bộ lập lịch sẽ tìm các nút phù hợp nhất để nhóm có thể được lên lịch. Ví dụ: nếu có năm nút công nhân có sẵn tài nguyên để chạy nhóm, thì nó sẽ chọn tất cả năm nút. Nếu không có nút nào thì nhóm đó sẽ không thể lập lịch trình và được chuyển đến hàng đợi lập lịch trình. Nếu Đó là một cụm lớn, giả sử có 100 nút công nhân và bộ lập lịch không lặp lại trên tất cả các nút. Có một tham số cấu hình bộ lập lịch được gọi là percentageOfNodesToScore. Giá trị mặc định thường là 50% . Vì vậy, nó cố gắng lặp lại hơn 50% số nút theo kiểu vòng tròn. Nếu các nút công nhân được trải rộng trên nhiều vùng thì bộ lập lịch sẽ lặp lại các nút ở các vùng khác nhau. Đối với các cụm rất lớn, mặc định percentageOfNodesToScorelà 5%.
Trong giai đoạn tính điểm , bộ lập lịch xếp hạng các nút bằng cách gán điểm cho các nút công nhân được lọc. Bộ lập lịch thực hiện việc chấm điểm bằng cách gọi nhiều plugin lập lịch . Cuối cùng, nút công nhân có thứ hạng cao nhất sẽ được chọn để lên lịch cho nhóm. Nếu tất cả các nút có cùng thứ hạng thì một nút sẽ được chọn ngẫu nhiên.
Sau khi nút được chọn, bộ lập lịch sẽ tạo một sự kiện ràng buộc trong máy chủ API. Có nghĩa là một sự kiện để liên kết một nhóm và nút.
Đây là những điều bạn cần biết về một công cụ lập lịch trình.

Nó là bộ điều khiển lắng nghe các sự kiện tạo nhóm trong máy chủ API.
Bộ lập lịch có hai giai đoạn. Chu kỳ lập kế hoạch  và  chu trình Ràng buộc . Cùng nhau nó được gọi là bối cảnh lập kế hoạch. Chu trình lập lịch sẽ chọn một nút công nhân và chu trình liên kết sẽ áp dụng thay đổi đó cho cụm.
Bộ lập lịch luôn đặt các nhóm có mức độ ưu tiên cao trước các nhóm có mức độ ưu tiên thấp để lập lịch. Ngoài ra, trong một số trường hợp, sau khi nhóm bắt đầu chạy trong nút đã chọn, nhóm có thể bị trục xuất hoặc chuyển sang các nút khác. Nếu bạn muốn hiểu thêm, hãy đọc hướng dẫn ưu tiên nhóm Kubernetes
Bạn có thể tạo bộ lập lịch tùy chỉnh và chạy nhiều bộ lập lịch trong một cụm cùng với bộ lập lịch gốc. Khi triển khai một nhóm, bạn có thể chỉ định bộ lập lịch tùy chỉnh trong bảng kê khai nhóm. Vì vậy, các quyết định lập lịch sẽ được đưa ra dựa trên logic của bộ lập lịch tùy chỉnh.
Bộ lập lịch có một khung lập lịch có thể cắm được . Có nghĩa là bạn có thể thêm plugin tùy chỉnh của mình vào quy trình lập lịch trình.

4. Trình quản lý bộ điều khiển Kube
Bộ điều khiển là gì? Bộ điều khiển là các chương trình chạy các vòng điều khiển vô hạn. Có nghĩa là nó chạy liên tục và theo dõi trạng thái thực tế và mong muốn của các đối tượng. Nếu có sự khác biệt giữa trạng thái thực tế và trạng thái mong muốn, nó đảm bảo rằng tài nguyên/đối tượng kubernetes ở trạng thái mong muốn.

Theo tài liệu chính thức,

Trong Kubernetes, bộ điều khiển là các vòng điều khiển theo dõi trạng thái cụm của bạn, sau đó thực hiện hoặc yêu cầu thay đổi nếu cần. Mỗi bộ điều khiển cố gắng di chuyển trạng thái cụm hiện tại đến gần trạng thái mong muốn hơn.

Giả sử bạn muốn tạo một bản triển khai, bạn chỉ định trạng thái mong muốn trong tệp YAML kê khai (phương pháp khai báo). Ví dụ: 2 bản sao, một ổ đĩa gắn kết, sơ đồ cấu hình, v.v. Bộ điều khiển triển khai tích hợp sẵn đảm bảo rằng quá trình triển khai luôn ở trạng thái mong muốn. Nếu người dùng cập nhật quá trình triển khai với 5 bản sao thì bộ điều khiển triển khai sẽ nhận ra nó và đảm bảo trạng thái mong muốn là 5 bản sao.

Trình quản lý bộ điều khiển Kube là một thành phần quản lý tất cả các bộ điều khiển Kubernetes. Các tài nguyên/đối tượng Kubernetes như nhóm, không gian tên, công việc, bản sao được quản lý bởi bộ điều khiển tương ứng. Ngoài ra, bộ lập lịch Kube cũng là bộ điều khiển được quản lý bởi trình quản lý bộ điều khiển Kube.


![image](https://github.com/user-attachments/assets/f3d71097-50b6-4785-bcb7-9cc71f64543f)

_Quy trình làm việc của trình quản lý bộ điều khiển Kubernetes_

Sau đây là danh sách các bộ điều khiển Kubernetes tích hợp quan trọng.

1. Bộ điều khiển triển khai
1. Bộ điều khiển bản sao
1. Bộ điều khiển DaemonSet 
1. Người kiểm soát công việc ( Công việc Kubernetes )
1. Bộ điều khiển CronJob
1. bộ điều khiển điểm cuối
1. bộ điều khiển không gian tên
1. người điều khiển tài khoản dịch vụ .
1. Bộ điều khiển nút
Đây là những điều bạn nên biết về trình quản lý bộ điều khiển Kube.

Nó quản lý tất cả các bộ điều khiển và các bộ điều khiển cố gắng giữ cụm ở trạng thái mong muốn.
Bạn có thể mở rộng kubernetes bằng bộ điều khiển tùy chỉnh được liên kết với định nghĩa tài nguyên tùy chỉnh.

### 5. Trình quản lý bộ điều khiển đám mây (CCM)
Khi kubernetes được triển khai trong môi trường đám mây, trình quản lý bộ điều khiển đám mây đóng vai trò là cầu nối giữa API nền tảng đám mây và cụm Kubernetes.

Bằng cách này, các thành phần cốt lõi của kubernetes có thể hoạt động độc lập và cho phép các nhà cung cấp đám mây tích hợp với kubernetes bằng cách sử dụng plugin. (Ví dụ: giao diện giữa cụm kubernetes và API đám mây AWS)

Tích hợp bộ điều khiển đám mây cho phép cụm Kubernetes cung cấp các tài nguyên đám mây như phiên bản (đối với nút), Cân bằng tải (đối với dịch vụ) và Khối lượng lưu trữ (đối với khối lượng liên tục).

![image](https://github.com/user-attachments/assets/c0c72054-405a-4680-ad3d-770270516866)

_Quy trình làm việc của kiến ​​trúc Trình quản lý bộ điều khiển đám mây_

Trình quản lý bộ điều khiển đám mây chứa một tập hợp các bộ điều khiển dành riêng cho nền tảng đám mây để đảm bảo trạng thái mong muốn của các thành phần dành riêng cho đám mây (nút, Bộ cân bằng tải , bộ lưu trữ, v.v.). Sau đây là ba bộ điều khiển chính là một phần của trình quản lý bộ điều khiển đám mây.

Bộ điều khiển nút: Bộ điều khiển này cập nhật thông tin liên quan đến nút bằng cách giao tiếp với API của nhà cung cấp đám mây. Ví dụ: ghi nhãn và chú thích nút, nhận tên máy chủ, tính khả dụng của CPU và bộ nhớ, tình trạng của nút, v.v.
Bộ điều khiển tuyến đường: Nó chịu trách nhiệm định cấu hình các tuyến mạng trên nền tảng đám mây. Vì vậy, các nhóm ở các nút khác nhau có thể nói chuyện với nhau.
Bộ điều khiển dịch vụ : Nó đảm nhiệm việc triển khai các bộ cân bằng tải cho các dịch vụ kubernetes, gán địa chỉ IP, v.v.
Sau đây là một số ví dụ cổ điển về trình quản lý bộ điều khiển đám mây.

Triển khai Kubernetes Service thuộc loại Load Balancer. Tại đây Kubernetes cung cấp Loadbalancer dành riêng cho Đám mây và tích hợp với Dịch vụ Kubernetes.
Cung cấp khối lượng lưu trữ (PV) cho các nhóm được hỗ trợ bởi các giải pháp lưu trữ đám mây.
Trình quản lý bộ điều khiển đám mây tổng thể quản lý vòng đời của các tài nguyên dành riêng cho đám mây được kubernetes sử dụng.

# Các thành phần nút Worker Kubernetes:

## Bây giờ chúng ta hãy xem xét từng thành phần của nút công nhân.

### 1. Kubelet
Kubelet là thành phần tác nhân chạy trên mọi nút trong cụm. t không chạy dưới dạng vùng chứa thay vào đó chạy dưới dạng daemon, được quản lý bởi systemd.

Nó chịu trách nhiệm đăng ký các nút công nhân với máy chủ API và làm việc với podSpec (Đặc tả Pod – YAML hoặc JSON) chủ yếu từ máy chủ API. podSpec xác định các vùng chứa sẽ chạy bên trong nhóm, tài nguyên của chúng (ví dụ: giới hạn CPU và bộ nhớ) cũng như các cài đặt khác như biến môi trường, ổ đĩa và nhãn.

Sau đó, nó đưa podSpec về trạng thái mong muốn bằng cách tạo các thùng chứa.

Nói một cách đơn giản, kubelet chịu trách nhiệm về những điều sau.

Tạo, sửa đổi và xóa vùng chứa cho nhóm.
Chịu trách nhiệm xử lý các cuộc thăm dò về mức độ hoạt động, mức độ sẵn sàng và khởi động.
Chịu trách nhiệm Gắn các ổ đĩa bằng cách đọc cấu hình nhóm và tạo các thư mục tương ứng trên máy chủ để gắn ổ đĩa.
Thu thập và báo cáo trạng thái Nút và nhóm thông qua các cuộc gọi đến máy chủ API với các triển khai như cAdvisor và  CRI.
Kubelet cũng là bộ điều khiển theo dõi các thay đổi của nhóm và sử dụng thời gian chạy vùng chứa của nút để kéo hình ảnh, chạy vùng chứa, v.v.

Ngoài PodSpec từ máy chủ API, kubelet có thể chấp nhận podSpec từ một tệp, điểm cuối HTTP và máy chủ HTTP. Một ví dụ điển hình về “podSpec từ một tệp” là nhóm tĩnh Kubernetes.

Nhóm tĩnh được điều khiển bởi kubelet chứ không phải máy chủ API.

Điều này có nghĩa là bạn có thể tạo nhóm bằng cách cung cấp vị trí YAML của nhóm cho thành phần Kubelet. Tuy nhiên, các nhóm tĩnh do Kubelet tạo không được máy chủ API quản lý.

Dưới đây là trường hợp sử dụng ví dụ thực tế của nhóm tĩnh.

Trong khi khởi động mặt phẳng điều khiển, kubelet khởi động máy chủ api, bộ lập lịch và trình quản lý bộ điều khiển dưới dạng các nhóm tĩnh từ podSpecs nằm ở/etc/kubernetes/manifests

Sau đây là một số điều quan trọng về kubelet.

Kubelet sử dụng giao diện gRPC CRI (giao diện thời gian chạy vùng chứa) để giao tiếp với thời gian chạy vùng chứa.
Nó cũng hiển thị điểm cuối HTTP để truyền nhật ký và cung cấp các phiên thực thi cho khách hàng.
Sử dụng gRPC CSI (giao diện lưu trữ vùng chứa) để định cấu hình khối lượng khối.
Nó sử dụng plugin CNI được định cấu hình trong cụm để phân bổ địa chỉ IP của nhóm và thiết lập mọi tuyến mạng và quy tắc tường lửa cần thiết cho nhóm.

![image](https://github.com/user-attachments/assets/c423dda9-2d90-49e0-b390-0f0718de0299)

_Giải thích kubelet thành phần Kubernetes_

## 2. Proxy Kube
Để hiểu proxy Kube, bạn cần có kiến ​​thức cơ bản về Dịch vụ Kubernetes và các đối tượng điểm cuối.

Dịch vụ trong Kubernetes là một cách để hiển thị một tập hợp các nhóm bên trong hoặc cho lưu lượng truy cập bên ngoài. Khi bạn tạo đối tượng dịch vụ, nó sẽ nhận được một IP ảo được gán cho nó. Nó được gọi là clusterIP. Nó chỉ có thể truy cập được trong cụm Kubernetes.

Đối tượng Endpoint chứa tất cả các địa chỉ IP và cổng của các nhóm nhóm trong đối tượng Dịch vụ. Bộ điều khiển điểm cuối chịu trách nhiệm duy trì danh sách các địa chỉ IP nhóm (điểm cuối). Bộ điều khiển dịch vụ chịu trách nhiệm định cấu hình điểm cuối cho dịch vụ.

Bạn không thể ping ClusterIP vì nó chỉ được sử dụng để khám phá dịch vụ, không giống như các IP nhóm có thể ping được.

Bây giờ hãy hiểu Kube Proxy.

Kube-proxy là một daemon chạy trên mọi nút dưới dạng một daemonset . Nó là một thành phần proxy triển khai khái niệm Dịch vụ Kubernetes cho các nhóm. (DNS duy nhất cho một tập hợp các nhóm có cân bằng tải). Nó chủ yếu ủy quyền UDP, TCP và SCTP và không hiểu HTTP.

Khi bạn hiển thị các nhóm bằng Dịch vụ (ClusterIP), Kube-proxy sẽ tạo các quy tắc mạng để gửi lưu lượng truy cập đến các nhóm phụ trợ (điểm cuối) được nhóm trong đối tượng Dịch vụ. Có nghĩa là tất cả việc cân bằng tải và khám phá dịch vụ đều được xử lý bởi proxy Kube.

Vậy Kube-proxy hoạt động như thế nào?

Proxy Kube giao tiếp với máy chủ API để nhận thông tin chi tiết về Dịch vụ (ClusterIP) cũng như các IP và cổng nhóm tương ứng (điểm cuối). Nó cũng giám sát những thay đổi trong dịch vụ và điểm cuối.

Sau đó, Kube-proxy sử dụng bất kỳ chế độ nào sau đây để tạo/cập nhật quy tắc định tuyến lưu lượng truy cập đến các nhóm phía sau Dịch vụ

1. IPTables : Đây là chế độ mặc định. Trong chế độ IPTables, lưu lượng được xử lý theo quy tắc IPtable. Điều này có nghĩa là đối với mỗi dịch vụ, các quy tắc IPtable sẽ được tạo. Các quy tắc này nắm bắt lưu lượng truy cập đến ClusterIP và sau đó chuyển tiếp nó đến nhóm phụ trợ. Ngoài ra, ở chế độ này, kube-proxy chọn ngẫu nhiên nhóm phụ trợ để cân bằng tải. Sau khi kết nối được thiết lập, các yêu cầu sẽ chuyển đến cùng một nhóm cho đến khi kết nối bị ngắt.
1. IPVS : Đối với các cụm có dịch vụ vượt quá 1000, IPVS cung cấp cải thiện hiệu suất. Nó hỗ trợ các thuật toán cân bằng tải sau cho phần phụ trợ.
- rr: round-robin : Đây là chế độ mặc định.
- lc: ít kết nối nhất (số lượng kết nối mở nhỏ nhất)
- dh: băm đích
- sh: băm nguồn
- sed: độ trễ dự kiến ​​ngắn nhất
- nq: không bao giờ xếp hàng
- Không gian người dùng (cũ & không được đề xuất)
- Kernelspace : Chế độ này chỉ dành cho hệ thống Windows.

![image](https://github.com/user-attachments/assets/dfd8ff8b-fbe1-4366-b9dc-b6c034d1a6c1)

Nếu bạn muốn hiểu sự khác biệt về hiệu suất giữa chế độ IPtables kube-proxy và IPVS, hãy đọc bài viết này .

Ngoài ra, bạn có thể chạy cụm Kubernetes mà không cần kube-proxy bằng cách thay thế nó bằng Cilium .

Tính năng Alpha 1.29 : Kubeproxy có chương trình phụ trợ mới dựa trên 𝗻𝗳𝘁𝗮𝗯𝗹𝗲𝘀. nftables là sự kế thừa của IPtables được thiết kế đơn giản và hiệu quả hơn

### 3. Thời gian chạy vùng chứa
Chắc hẳn bạn đã biết về Java Runtime (JRE) . Đây là phần mềm cần thiết để chạy các chương trình Java trên máy chủ. Theo cách tương tự, thời gian chạy vùng chứa là một thành phần phần mềm cần thiết để chạy vùng chứa.

Thời gian chạy vùng chứa chạy trên tất cả các nút trong cụm Kubernetes. Nó chịu trách nhiệm lấy hình ảnh từ cơ quan đăng ký vùng chứa, chạy vùng chứa, phân bổ và cách ly tài nguyên cho vùng chứa và quản lý toàn bộ vòng đời của vùng chứa trên máy chủ.

Để hiểu rõ hơn về điều này, chúng ta hãy xem xét hai khái niệm chính:

Giao diện thời gian chạy vùng chứa (CRI): Đây là một bộ API cho phép Kubernetes tương tác với các thời gian chạy vùng chứa khác nhau. Nó cho phép sử dụng các thời gian chạy vùng chứa khác nhau có thể thay thế cho Kubernetes. CRI xác định API để tạo, bắt đầu, dừng và xóa vùng chứa cũng như để quản lý hình ảnh và mạng vùng chứa.
Sáng kiến ​​vùng chứa mở (OCI): Đây là bộ tiêu chuẩn cho các định dạng vùng chứa và thời gian chạy
Kubernetes hỗ trợ nhiều thời gian chạy container (CRI-O, Docker Engine, containerd, v.v.) tuân thủ Giao diện thời gian chạy container (CRI) . Điều này có nghĩa là tất cả các thời gian chạy vùng chứa này đều triển khai giao diện CRI và hiển thị API gRPC CRI (điểm cuối dịch vụ hình ảnh và thời gian chạy).

Vậy Kubernetes tận dụng thời gian chạy của container như thế nào?

Như chúng ta đã tìm hiểu trong phần Kubelet, tác nhân kubelet chịu trách nhiệm tương tác với thời gian chạy vùng chứa bằng API CRI để quản lý vòng đời của vùng chứa. Nó cũng lấy tất cả thông tin về vùng chứa từ thời gian chạy của vùng chứa và cung cấp thông tin đó cho mặt phẳng điều khiển.

Hãy lấy một ví dụ về giao diện thời gian chạy của vùng chứa CRI-O . Dưới đây là thông tin tổng quan cấp cao về cách hoạt động của thời gian chạy vùng chứa với kubernetes.

![image](https://github.com/user-attachments/assets/912fd3d1-484b-4185-8659-5c630f406b2b)

### Tổng quan về CRI-O thời gian chạy vùng chứa Kubernetes

1. Khi có yêu cầu mới cho một nhóm từ máy chủ API, kubelet sẽ giao tiếp với trình nền CRI-O để khởi chạy các vùng chứa được yêu cầu thông qua Giao diện thời gian chạy vùng chứa Kubernetes.
1. CRI-O kiểm tra và lấy hình ảnh vùng chứa được yêu cầu từ sổ đăng ký vùng chứa được định cấu hình bằng containers/imagethư viện.
1. Sau đó, CRI-O tạo đặc tả thời gian chạy OCI (JSON) cho vùng chứa.
1. Sau đó, CRI-O khởi chạy thời gian chạy tương thích OCI (runc) để bắt đầu quy trình vùng chứa theo thông số kỹ thuật thời gian chạy.

## Các thành phần bổ trợ cụm Kubernetes
Ngoài các thành phần cốt lõi, cụm kubernetes cần có các thành phần bổ trợ để hoạt động đầy đủ. Việc chọn một addon phụ thuộc vào yêu cầu của dự án và trường hợp sử dụng.

Sau đây là một số thành phần bổ trợ phổ biến mà bạn có thể cần trên một cụm.

### Plugin CNI (Giao diện mạng container)
CoreDNS (Dành cho máy chủ DNS): CoreDNS hoạt động như một máy chủ DNS trong cụm Kubernetes. Bằng cách bật tiện ích bổ sung này, bạn có thể kích hoạt tính năng khám phá dịch vụ dựa trên DNS.
Máy chủ số liệu (Dành cho số liệu tài nguyên): Tiện ích bổ sung này giúp bạn thu thập dữ liệu hiệu suất và mức sử dụng tài nguyên của Nút và nhóm trong cụm.
Giao diện người dùng web (Bảng điều khiển Kubernetes): Tiện ích bổ sung này cho phép bảng điều khiển Kubernetes quản lý đối tượng thông qua giao diện người dùng web.
1. Plugin CNI
Trước tiên, bạn cần hiểu Giao diện mạng container (CNI)

Đây là kiến ​​trúc dựa trên plugin với các thông số kỹ thuật và thư viện trung lập với nhà cung cấp để tạo giao diện mạng cho Vùng chứa.

Nó không dành riêng cho Kubernetes. Với mạng container CNI có thể được chuẩn hóa trên các công cụ điều phối container như Kubernetes, Mesos, CloudFoundry, Podman, Docker, v.v.

Khi nói đến mạng container, các công ty có thể có các yêu cầu khác nhau như cách ly mạng, bảo mật, mã hóa, v.v. Khi công nghệ container tiên tiến, nhiều nhà cung cấp mạng đã tạo ra các giải pháp dựa trên CNI cho các container có nhiều khả năng kết nối mạng. Bạn có thể gọi nó là CNI-Plugin

Điều này cho phép người dùng chọn giải pháp mạng phù hợp nhất với nhu cầu của họ từ các nhà cung cấp khác nhau.

Plugin CNI hoạt động như thế nào với Kubernetes?

Trình quản lý bộ điều khiển Kube chịu trách nhiệm gán CIDR nhóm cho mỗi nút. Mỗi nhóm nhận được một địa chỉ IP duy nhất từ ​​CIDR nhóm.
Kubelet tương tác với thời gian chạy vùng chứa để khởi chạy nhóm đã lên lịch. Plugin CRI là một phần của thời gian chạy Container tương tác với plugin CNI để định cấu hình mạng nhóm.
Plugin CNI cho phép kết nối mạng giữa các nhóm trải rộng trên cùng một nút hoặc các nút khác nhau bằng cách sử dụng mạng lớp phủ.

![image](https://github.com/user-attachments/assets/3467be9a-7800-421a-b2e3-162a4022fa76)

Sau đây là các chức năng cấp cao được cung cấp bởi các plugin CNI.

Mạng nhóm
Bảo mật và cách ly mạng nhóm bằng cách sử dụng Chính sách mạng để kiểm soát luồng lưu lượng giữa các nhóm và giữa các không gian tên.
Một số plugin CNI phổ biến bao gồm:

Calico
vải nỉ
Dệt lưới
Cilium (Sử dụng eBPF)
Amazon VPC CNI (Dành cho AWS VPC)
Azure CNI (Dành cho mạng ảo Azure)Mạng Kubernetes là một chủ đề lớn và nó khác nhau tùy theo nền tảng lưu trữ.
Đối tượng gốc Kubernetes
Cho đến bây giờ chúng ta đã tìm hiểu về các thành phần kubernetes cốt lõi và cách hoạt động của từng thành phần.

Tất cả các thành phần này đều hướng tới việc quản lý các đối tượng Kubernetes chính sau đây.

Nhóm
Không gian tên
Bản sao
Triển khai
Daemonset
Bộ trạng thái
Việc làm & Cronjob
Bản đồ cấu hình và bí mật
Khi nói đến kết nối mạng, các đối tượng Kubernetes sau đây đóng vai trò quan trọng.

Dịch vụ
Xâm nhập
Chính sách mạng.
Ngoài ra, Kubernetes có thể mở rộng bằng CRD và Bộ điều khiển tùy chỉnh. Vì vậy, các thành phần cụm cũng quản lý các đối tượng được tạo bằng bộ điều khiển tùy chỉnh và định nghĩa tài nguyên tùy chỉnh.

# Câu hỏi thường gặp về kiến ​​trúc Kubernetes

Mục đích chính của mặt phẳng điều khiển Kubernetes là gì?
Mặt phẳng điều khiển chịu trách nhiệm duy trì trạng thái mong muốn của cụm và các ứng dụng đang chạy trên đó. Nó bao gồm các thành phần như máy chủ API, etcd, Trình lập lịch biểu và trình quản lý bộ điều khiển.

Mục đích của các nút công nhân trong cụm Kubernetes là gì?
Các nút công nhân là các máy chủ (kim loại trần hoặc ảo) chạy vùng chứa trong cụm. Chúng được quản lý bởi mặt phẳng điều khiển và nhận hướng dẫn từ nó về cách chạy các container là một phần của nhóm.

Giao tiếp giữa mặt phẳng điều khiển và nút công nhân được bảo mật như thế nào trong Kubernetes?
Giao tiếp giữa mặt phẳng điều khiển và các nút công nhân được bảo mật bằng chứng chỉ PKI và giao tiếp giữa các thành phần khác nhau diễn ra qua TLS. Bằng cách này, chỉ những thành phần đáng tin cậy mới có thể giao tiếp với nhau.

Mục đích của kho lưu trữ khóa-giá trị etcd trong Kubernetes là gì?
Etcd chủ yếu lưu trữ các đối tượng kubernetes, thông tin cụm, thông tin nút và dữ liệu cấu hình của cụm, chẳng hạn như trạng thái mong muốn của các ứng dụng chạy trên cụm.

Điều gì xảy ra với các ứng dụng Kubernetes nếu etcd bị hỏng?
Mặc dù các ứng dụng đang chạy sẽ không bị ảnh hưởng nếu etcd bị ngừng hoạt động, nhưng sẽ không thể tạo hoặc cập nhật bất kỳ đối tượng nào nếu không có etcd hoạt động.

# Phần kết luận
Hiểu kiến ​​trúc Kubernetes giúp bạn triển khai và vận hành Kubernetes hàng ngày.

Khi triển khai thiết lập cụm cấp sản xuất, việc có kiến ​​thức đúng đắn về các thành phần Kubernetes sẽ giúp bạn chạy và khắc phục sự cố ứng dụng
