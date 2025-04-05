# AWS DEA

## 1. Data Engineering Fundamentals
- **Structured Data**
  - Definition
    - Data that is organized in a
defined manner or schema, typically
found in relational databases

- **Unstructured Data**
     - **apiVersion**: Phiên bản API.
     - **kind**: Loại tài nguyên (Pod).
     - **metadata**: Thông tin (tên, nhãn).
     - **spec**:
        - Containers
        - Ports
        - Volumes

  - Kiểu Pod
     - **Single-container Pod**: Chạy 1 container chính.
     - **Multi-container Pod**:
       - Mô hình: Sidecar, Ambassador, Adapter, init container
       - Chia sẻ IP, Volume.

  - ## 🚀 Vòng đời Pod
     - **Pending**: Đang chờ tài nguyên.
     - **Running**: Container đang hoạt động.
     - **Succeeded**: Hoàn tất thành công.
     - **Failed**: Container gặp lỗi.
     - **CrashLoopBackOff**: Liên tục khởi động lại do lỗi.

  - ## 🔍 Quản lý với kubectl
     - **Tạo Pod:** `kubectl apply -f pod.yaml`
     - **Xem Pod:** `kubectl get pods`
     - **Chi tiết Pod:** `kubectl describe pod <name>`
     - **Xem log:** `kubectl logs <pod> -c <container>`
     - **Truy cập Pod:** `kubectl exec -it <pod> -- /bin/bash`

  - ## 🗂️ Pod và Tính Hồi Phục
     - Pod KHÔNG tự hồi phục khi lỗi.
     - Sử dụng Controller để quản lý:
     - Deployment (Stateless)
     - StatefulSet (Stateful)
     - DaemonSet (Chạy trên mọi node)

  - ## ⚡ Ưu - Nhược điểm
    - ✅ Đơn giản, dễ triển khai.
    - ✅ Hiệu quả khi chia sẻ tài nguyên.
    - ❌ Không tự phục hồi khi lỗi.
    - ❌ Không tối ưu cho ứng dụng lớn.
- **ReplicaSet**: Đảm bảo số Pod mong muốn.
- **Deployment**: Quản lý ReplicaSet, hỗ trợ rolling updates.
- **StatefulSet**: Quản lý ứng dụng có trạng thái.
- **DaemonSet**: Chạy Pod trên tất cả các node.
- **Job**: Thực thi tác vụ một lần rồi dừng.
- **CronJob**: Chạy tác vụ định kỳ.

## 2. Networking Resources
- **Service**: Expose Pod (ClusterIP, NodePort, LoadBalancer).
- **Ingress**: Quản lý HTTP/HTTPS traffic.
- **NetworkPolicy**: Kiểm soát lưu lượng mạng giữa các Pod.

## 3. Storage Resources
- **PersistentVolume (PV)**: Định nghĩa lưu trữ vật lý.
- **PersistentVolumeClaim (PVC)**: Yêu cầu sử dụng PV.
- **StorageClass**: Dynamic provisioning cho lưu trữ.
- **Volume**: Gắn vào Pod để chia sẻ dữ liệu.

## 4. Configuration & Secret Management
- **ConfigMap**: Lưu cấu hình dạng key-value.
- **Secret**: Lưu dữ liệu nhạy cảm (token, mật khẩu).
- **ResourceQuota**: Giới hạn tài nguyên theo namespace.
- **LimitRange**: Đặt giới hạn CPU, RAM cho Pod.

## 5. Cluster Resources
- **Namespace**: Phân chia tài nguyên logic.
- **Node**: Máy chủ vật lý/ảo trong cluster.
- **Role/RoleBinding**: Quản lý quyền trong namespace.
- **ClusterRole/ClusterRoleBinding**: Quản lý quyền toàn cụm.
- **ServiceAccount**: Cấp quyền cho Pod truy cập API.

## 6. Custom Resources
- **CustomResourceDefinition (CRD)**: Định nghĩa tài nguyên tùy chỉnh.
- **Operator**: Tự động hóa quản lý ứng dụng phức tạp.