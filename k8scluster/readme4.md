# Phần 4. Triển khai Cụm Kubernetes trong Ubuntu 24.04 LTS server trên vSphere vSAN Cluster hoặc vSphere vVOLUME phiên bản 8x:
Tham khảo: https://docs.vmware.com/en/VMware-vSphere-Container-Storage-Plug-in/3.0/vmware-vsphere-csp-getting-started/GUID-982E7698-3C8D-4F88-A9EA-3A38970370EA.html

## 4.1. vSphere vSAN và VMware vVOLUME là gì ?

**vSAN (Virtual SAN):** Là một giải pháp lưu trữ phân tán và software-defined của VMware, cho phép tổ chức lưu trữ dữ liệu trên cụm máy chủ vSphere. 
vSAN kết hợp các ổ đĩa ổ cứng và ổ SSD từ các máy chủ thành một hệ thống lưu trữ hiệu suất cao, tự động phân phối và quản lý dữ liệu.

**vVols (Virtual Volumes):** Là một công nghệ lưu trữ mạnh mẽ của VMware, cho phép quản lý dữ liệu ở mức độ đơn vị cấp VM thay vì ở mức LUN hoặc tập tin. 
vVols cho phép VMs tận dụng các tính năng lưu trữ nâng cao như luồng dữ liệu thẳng đến ổ đĩa và tích hợp mạnh mẽ với VMware vSphere.
