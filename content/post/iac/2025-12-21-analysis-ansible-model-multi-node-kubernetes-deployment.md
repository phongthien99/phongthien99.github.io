---
title: "Analysis of the Ansible Model in Multi-Node Kubernetes Deployment"
author: phongthien99
date: 2025-12-21 21:40:00 +0700
categories: [devops, iac]
tags: [infrastructure-as-code, terraform, ansible, automation, devops]
math: true
media_subpath: '/posts/iac'
---
# Analysis of the Ansible Model in Multi-Node Kubernetes Deployment

## 1.ĐẶT VẤN ĐỀ

Trong các hệ thống hạ tầng hiện đại, việc tự động hóa triển khai và quản lý cấu hình đóng vai trò quan trọng nhằm đảm bảo tính nhất quán và giảm thiểu lỗi do thao tác thủ công. **Ansible** là một công cụ tự động hóa không cần agent, cho phép quản lý và điều phối nhiều máy chủ thông qua các playbook và role có cấu trúc rõ ràng.

Nghiên cứu này tập trung phân tích cách **Ansible tổ chức và điều phối quy trình triển khai hạ tầng đa node**, thông qua một case study triển khai Kubernetes bằng kubeadm. Kubernetes được sử dụng như một ví dụ thực tế nhằm làm rõ khả năng tự động hóa, tái sử dụng và mở rộng của Ansible trong triển khai hệ thống phức tạp

## 2. KIẾN THỨC NỀN TẢNG

## 2.1 Tổng quan mô hình hoạt động của Ansible

Trong Ansible, **Playbook** đóng vai trò trung tâm, chịu trách nhiệm điều phối toàn bộ quá trình triển khai. Playbook sử dụng **Inventory** để xác định các node mục tiêu và gọi các **Role** tương ứng với từng chức năng triển khai.

Sơ đồ lớp (class diagram) ở phần kiến thức mô tả rõ mối quan hệ giữa các thành phần:

- Playbook gọi (include/import) các Role.
- Mỗi Role bao gồm các Task cụ thể và có thể sử dụng Template để sinh file cấu hình động.
- Inventory kết hợp với `group_vars` và `host_vars` để cung cấp biến cấu hình cho toàn bộ hệ thống.

Cách tổ chức này giúp tách biệt **logic triển khai**, **cấu hình hạ tầng** và **tham số môi trường**, từ đó tăng khả năng tái sử dụng và giảm sự phụ thuộc cứng vào môi trường cụ thể.

{{< mermaid >}}
classDiagram
    class Playbook {
        +includes(Role)
        +uses(Inventory)
    }

    class Role {
        +contains(Tasks)
        +uses(Templates)
    }

    class Tasks {
        +executes()
    }

    class Templates {
        +generatesConfig()
    }

    class Inventory {
        +has(group_vars)
        +has(host_vars)
    }

    class group_vars {
        +providesVariables()
    }

    class host_vars {
        +providesVariables()
    }

    Playbook --> Role : includes
    Playbook --> Inventory : uses
    Role --> Tasks : contains
    Role --> Templates : uses
    Inventory --> group_vars : has
    Inventory --> host_vars : has
    group_vars --> Role : providesVariables
    host_vars --> Role : providesVariables

{{</ mermaid >}}

## 2.2 Kubernetes và kubeadm

Kubeadm là công cụ chính thức do Kubernetes cung cấp nhằm đơn giản hóa việc khởi tạo cluster. Kubeadm đảm nhiệm các nhiệm vụ chính như:

- Khởi tạo control plane
- Sinh chứng chỉ
- Thiết lập cấu hình kubelet
- Cung cấp lệnh `kubeadm join` để thêm worker node

Tuy nhiên, kubeadm **không tự động hóa các bước chuẩn bị hệ thống** (cài container runtime, cấu hình kernel, network, v.v.). Đây chính là phần mà Ansible bổ sung để hoàn thiện quy trình triển khai end-to-end

### 2. Case study triển khai k8s bằng kubeadmin

Sơ đồ thiết kế triển khai cho thấy **cấu trúc phân lớp rõ ràng của project Ansible**:

- **ansible.cfg** và **Makefile** đóng vai trò lớp điều khiển, giúp tiêu chuẩn hóa cách chạy playbook.
- **Inventory** và **group_vars** định nghĩa hạ tầng và tham số cấu hình cluster.
- Các **Playbook** đảm nhiệm điều phối theo từng giai đoạn: chuẩn bị node, khởi tạo control plane và thêm worker.
- Các **Role** được thiết kế theo đúng trình tự logic của kubeadm:
    - `common`: chuẩn bị hệ thống
    - `containerd`: cài runtime
    - `kubernetes`: cài thành phần K8s
    - `control_plane`: khởi tạo cluster
    - `cni`: triển khai mạng
    - `workers`: join node

{{< mermaid >}}
classDiagram
    %% Root Files
    class AnsibleConfig["ansible.cfg"] {
        <<config>>
        +inventory = inventory/hosts.ini
        +roles_path = ./roles
        +host_key_checking = False
    }

    class Makefile["Makefile"] {
        <<commands>>
        +ping: Test connectivity
        +check: Check prerequisites
        +deploy: Full deployment
        +deploy-nodes: Setup nodes
        +deploy-control-plane: Setup CP
        +deploy-workers: Setup workers
        +verify: Check cluster
        +reset: Reset cluster
    }

    %% Inventory Package
    namespace inventory {
        class HostsIni["hosts.ini"] {
            <<inventory>>
            +[control_plane]
            +k8s-dev-cp: 192.168.100.10
            +[workers]
            +k8s-dev-worker-1: 192.168.100.11
            +k8s-dev-worker-2: 192.168.100.12
            +[k8s_cluster:vars]
            +k8s_version: v1.29.0
            +pod_network_cidr: 10.244.0.0/16
        }
    }

    %% Group Vars Package
    namespace group_vars {
        class AllYml["all.yml"] {
            <<variables>>
            +k8s_version: "1.29"
            +pod_network_cidr: "10.244.0.0/16"
            +service_cidr: "10.96.0.0/12"
            +control_plane_endpoint: "192.168.100.10"
            +cni_plugin: "calico"
            +pod_ready_timeout: 300
        }

        class K8sClusterYml["k8s_cluster.yml"] {
            <<variables>>
            +cluster specific vars
        }
    }

    %% Playbooks Package
    namespace playbooks {
        class DeployK8sCluster["deploy-k8s-cluster.yml"] {
            <<playbook>>
            +import_playbook: setup-nodes.yml
            +import_playbook: setup-control-plane.yml
            +import_playbook: setup-workers.yml
            +task: Get all pods status
            +task: Display cluster summary
        }

        class CheckPrereq["check-prerequisites.yml"] {
            <<playbook>>
            +task: Check OS compatibility
            +task: Check memory >= 2GB
            +task: Check CPU >= 2 cores
            +task: Check required ports
            +task: Validate inventory
        }

        class SetupNodes["setup-nodes.yml"] {
            <<playbook>>
            +hosts: k8s_cluster
            +role: common
            +role: containerd
            +role: kubernetes
        }

        class SetupControlPlane["setup-control-plane.yml"] {
            <<playbook>>
            +hosts: control_plane
            +role: control_plane
            +role: cni
        }

        class SetupWorkers["setup-workers.yml"] {
            <<playbook>>
            +hosts: workers
            +role: workers
        }
    }

    %% Roles Package - Common
    namespace roles.common {
        class CommonTasks["tasks/main.yml"] {
            <<tasks>>
            +Disable swap permanently
            +Load kernel module: overlay
            +Load kernel module: br_netfilter
            +Set sysctl: net.ipv4.ip_forward=1
            +Set sysctl: net.bridge.bridge-nf-call-iptables=1
            +Update apt cache
        }
    }

    %% Roles Package - Containerd
    namespace roles.containerd {
        class ContainerdDefaults["defaults/main.yml"] {
            <<defaults>>
            +containerd_version: "latest"
        }

        class ContainerdTasks["tasks/main.yml"] {
            <<tasks>>
            +Install containerd package
            +Create /etc/containerd directory
            +Generate default config
            +Set SystemdCgroup = true
            +Restart containerd service
            +Enable containerd on boot
        }
    }

    %% Roles Package - Kubernetes
    namespace roles.kubernetes {
        class K8sTemplates["templates/kubeadm-config.yaml.j2"] {
            <<template>>
            +apiVersion: kubeadm.k8s.io/v1beta3
            +kind: ClusterConfiguration
            +kubernetesVersion: k8s_version
            +networking.podSubnet:  pod_network_cidr
        }

        class K8sHandlers["handlers/main.yml"] {
            <<handlers>>
            +restart kubelet
        }

        class K8sTasks["tasks/main.yml"] {
            <<tasks>>
            +Add K8s apt key
            +Add K8s repository
            +Install kubeadm= k8s_version
            +Install kubelet= k8s_version
            +Install kubectl= k8s_version
            +Hold kubeadm package
            +Hold kubelet package
            +Hold kubectl package
            +Enable kubelet service
        }
    }

    %% Roles Package - Control Plane
    namespace roles.control_plane {
        class ControlPlaneTasks["tasks/main.yml"] {
            <<tasks>>
            +Check if cluster initialized
            +Run kubeadm init with config
            +Create .kube directory
            +Copy admin.conf to .kube/config
            +Set kubeconfig ownership
            +Generate join command
            +Save join command to /tmp/
            +Wait for control plane pods
        }
    }

    %% Roles Package - CNI
    namespace roles.cni {
        class CNIDefaults["defaults/main.yml"] {
            <<defaults>>
            +cni_plugin: "calico"
            +calico_version: "v3.26.1"
            +flannel_version: "v0.22.0"
        }

        class CNITasks["tasks/main.yml"] {
            <<tasks>>
            +Download Calico manifest
            +Apply Calico to cluster
            +Wait for calico-node DaemonSet
            +Wait for calico-kube-controllers
        }
    }

    %% Roles Package - Workers
    namespace roles.workers {
        class WorkersTasks["tasks/main.yml"] {
            <<tasks>>
            +Check if kubelet.conf exists
            +Generate join command from CP
            +Execute join command
            +Wait for kubelet to start
            +Verify node joined successfully
        }
    }

    %% Relationships - Config to Inventory
    AnsibleConfig --> HostsIni : uses

    %% Relationships - Makefile to Playbooks
    Makefile --> DeployK8sCluster : executes
    Makefile --> CheckPrereq : executes

    %% Relationships - Playbooks Imports
    DeployK8sCluster ..> SetupNodes : imports
    DeployK8sCluster ..> SetupControlPlane : imports
    DeployK8sCluster ..> SetupWorkers : imports

    %% Relationships - Playbooks use Variables
    AllYml ..> DeployK8sCluster : configures
    AllYml ..> SetupNodes : configures
    AllYml ..> SetupControlPlane : configures
    K8sClusterYml ..> DeployK8sCluster : configures

    %% Relationships - Playbooks to Roles
    SetupNodes --> CommonTasks : includes role
    SetupNodes --> ContainerdTasks : includes role
    SetupNodes --> K8sTasks : includes role

    SetupControlPlane --> ControlPlaneTasks : includes role
    SetupControlPlane --> CNITasks : includes role

    SetupWorkers --> WorkersTasks : includes role

    %% Relationships - Roles internal
    ContainerdTasks ..> ContainerdDefaults : uses
    K8sTasks ..> K8sTemplates : uses
    K8sTasks ..> K8sHandlers : notifies
    CNITasks ..> CNIDefaults : uses


{{</ mermaid >}}

### Kết luận

Kết quả nghiên cứu cho thấy việc triển khai Kubernetes bằng **Ansible kết hợp kubeadm** mang lại nhiều lợi ích rõ rệt so với phương pháp thủ công.

Trước hết, quá trình triển khai được **chuẩn hóa và tự động hóa hoàn toàn**, giúp giảm thời gian cài đặt và hạn chế lỗi cấu hình. Thứ hai, cấu trúc project theo hướng role-based giúp hệ thống **dễ bảo trì, dễ mở rộng và dễ tái sử dụng** trong các môi trường khác nhau