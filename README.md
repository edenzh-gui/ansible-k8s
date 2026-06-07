# Ansible Kubernetes 集群一键部署项目

本项目提供了一套完整的 Ansible Playbook，用于在 Ubuntu 服务器上自动化部署单 Master、多 Worker 节点的 Kubernetes (v1.36) 集群。

## 🛠️ 环境要求

- **操作系统**: Ubuntu 20.04 / 22.04 / 24.04
- **硬件配置**: 建议至少 2 Core CPU, 4GB 内存
- **网络条件**: 所有节点需处于同一局域网互通，并能正常访问公网下载依赖包（国内服务器可自行在系统中配置全局代理或镜像加速）
- **控制节点**: 运行此脚本的机器需安装 Ansible 和 `sshpass`（若使用密码登录）。

## 📦 集群组件与架构

- **容器运行时**: Containerd (集成 SystemdCgroup)
- **Kubernetes 版本**: `v1.36.*`
- **网络插件 (CNI)**: Flannel (Pod 网络: `10.244.0.0/16`)
- **系统优化**: 自动关闭 Swap 分区，自动调整 K8s 所需的 sysctl 网络转发参数，加载 `br_netfilter` 等内核模块。

## 📁 目录结构

```text
ansible-k8s/
├── ansible.cfg                # Ansible 主配置文件
├── inventory/
│   ├── group_vars/
│   │   └── all.yml            # 全局变量定义（k8s 版本，网络段等）
│   └── hosts.ini              # 重点！主机清单及登录信息配置
├── playbooks/
│   └── cluster.yml            # 入口 Playbook 执行文件
└── roles/                     # 任务角色拆分目录
    ├── common/                # 基础环境准备与系统配置
    ├── containerd/            # 安装并配置容器运行时
    ├── kubernetes-base/       # 配置源并安装 kubeadm, kubelet, kubectl
    ├── master/                # Master 节点初始化与安装 Flannel
    └── worker/                # Worker 节点通过 token 加入集群
```

## 🚀 快速开始

### 1. 准备 Ansible 运行环境
在您的控制端机器（运行脚本的机器）上安装 Ansible 和密码传递工具：
```bash
# Ubuntu / Debian
sudo apt update && sudo apt install -y ansible sshpass

# CentOS / RHEL
sudo yum install -y epel-release && sudo yum install -y ansible sshpass
```

### 2. 配置您的服务器信息
编辑 `inventory/hosts.ini` 文件，填入您真实的服务器 IP 地址以及登录账号和密码。

```ini
[kube_master]
master1 ansible_host=192.168.9.41

[kube_node]
worker1 ansible_host=192.168.9.42
worker2 ansible_host=192.168.9.43

[all:vars]
ansible_user=root
ansible_ssh_pass=您的登录密码
```
*提示：如果使用免密登录，可以注释掉 `ansible_ssh_pass`。*

### 3. 测试连通性
在本项目根目录下，使用 Ansible 进行 Ping 测试，确保可以联通所有目标节点：
```bash
ansible -i inventory/hosts.ini all -m ping
```

### 4. 执行一键部署
连通性确认无误后，直接执行主控 Playbook：
```bash
ansible-playbook -i inventory/hosts.ini playbooks/cluster.yml
```
等待大约 3-5 分钟即可完成整个部署流程！

### 5. 验证集群状态
通过 SSH 登录到您的 Master 主机（如：192.168.9.41），执行：
```bash
kubectl get nodes
```
如果看到所有节点的状态都变为 `Ready`，说明集群搭建大功告成！

## ⚠️ 常见问题

- **网络不通卡住**：因为 K8s 1.36 的镜像位于 `registry.k8s.io`，国内可能会拉取失败。建议确保您的服务器处于能够流畅访问外网的状态。
- **SSH 报错**：如果在执行时遇到权限拒绝问题，请再次确认 `hosts.ini` 中的用户、密码是否正确，或者目标机的 sshd 服务是否开放了 root 密码登录。
