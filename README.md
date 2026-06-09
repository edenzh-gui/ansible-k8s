# Ansible Kubernetes 集群一键部署项目

本项目提供了一套完整的 Ansible Playbook，用于在 Linux 服务器上自动化部署 Kubernetes 集群，支持**多发行版**及**多 Master 高可用（HA）**模式。

---

## ✨ 功能特性

| 特性 | 说明 |
|------|------|
| 🐧 多发行版支持 | Ubuntu 20.04/22.04/24.04、Debian 11/12、CentOS Stream 9、Rocky/AlmaLinux 9 |
| ☸️ 高可用 HA 模式 | Nginx + Keepalived 实现 VIP 浮动，支持多 Master 节点，单 Master 宕机不影响集群 |
| 🔄 自动模式检测 | 根据 `hosts.ini` 中的 Master 数量自动选择单 Master 或 HA 模式，无需手动配置 |
| 🔒 版本锁定 | 防止意外升级 kubelet/kubeadm/kubectl |
| 📦 Containerd | 集成 SystemdCgroup，符合 K8s 最佳实践 |
| 🕸️ Flannel CNI | 自动部署 Pod 网络（`10.244.0.0/16`） |
| ⚡ IPVS 代理模式 | kube-proxy 使用 IPVS 模式，内核级哈希查找，性能优于 iptables |

---

## 🛠️ 环境要求

- **操作系统**: Ubuntu 20.04/22.04/24.04、Debian 11/12、CentOS Stream 9、Rocky Linux 9、AlmaLinux 9
- **硬件配置**: 建议至少 2 Core CPU, 4GB 内存（Master 节点推荐 4 Core, 8GB）
- **网络条件**: 所有节点需处于同一局域网互通，并能访问公网（用于下载依赖包）
- **控制节点**: 运行此脚本的机器需安装 Ansible 和 `sshpass`（若使用密码登录）

---

## 📁 目录结构

```text
ansible-k8s/
├── ansible.cfg                      # Ansible 主配置文件
├── inventory/
│   ├── group_vars/
│   │   └── all.yml                  # 全局变量（K8s 版本、网络段、HA 配置等）
│   └── hosts.ini                    # ⭐ 重点！主机清单及登录信息配置
├── playbooks/
│   └── cluster.yml                  # 入口 Playbook
└── roles/
    ├── common/                      # 基础环境准备（多发行版）+ IPVS 内核模块
    ├── containerd/                  # 安装容器运行时（多发行版）
    ├── kubernetes-base/             # 安装 kubeadm/kubelet/kubectl（多发行版）
    ├── nginx/                       # [HA] 安装并配置 Nginx 负载均衡（stream 模块）
    ├── keepalived/                  # [HA] 安装并配置 Keepalived 虚拟 IP
    ├── master/                      # Master 节点初始化（支持单 Master 和 HA）
    └── worker/                      # Worker 节点加入集群
```

---

## 🚀 快速开始

### 1. 安装 Ansible 控制端依赖

```bash
# Ubuntu / Debian
sudo apt update && sudo apt install -y ansible sshpass

# CentOS / RHEL / Rocky
sudo yum install -y epel-release && sudo yum install -y ansible sshpass
```

### 2. 选择部署模式

#### 模式一：单 Master（最简单）

编辑 `inventory/hosts.ini`，保留一个 `kube_master` 节点：

```ini
[kube_master]
master1 ansible_host=192.168.9.41 keepalived_priority=100

[kube_node]
worker1 ansible_host=192.168.9.42
worker2 ansible_host=192.168.9.43

[all:vars]
ansible_user=root
ansible_ssh_pass=您的登录密码
```

#### 模式二：多 Master 高可用（HA）

**步骤 1**：编辑 `inventory/hosts.ini`，配置 3 个 Master 节点：

```ini
[kube_master]
master1 ansible_host=192.168.9.41 keepalived_priority=100
master2 ansible_host=192.168.9.42 keepalived_priority=90
master3 ansible_host=192.168.9.43 keepalived_priority=80

[kube_node]
worker1 ansible_host=192.168.9.44
worker2 ansible_host=192.168.9.45
worker2 ansible_host=192.168.9.46

[all:vars]
ansible_user=root
ansible_ssh_pass=您的登录密码
```

**步骤 2**：编辑 `inventory/group_vars/all.yml`，配置 VIP：

```yaml
# 取消注释，填入同网段内一个未被使用的空闲 IP
ha_vip_ip: "192.168.9.100"

# 服务器网卡名（用 ip addr 命令查看，通常是 eth0 或 ens18）
ha_vip_interface: "eth0"
```

> **HA 架构说明**：Nginx 和 Keepalived 会自动部署在所有 Master 节点上。Nginx 使用 stream 模块做 TCP 四层负载均衡，将流量轮询转发到各 Master 的 APIServer。Keepalived 通过 VRRP 协议竞争 VIP，优先级（`keepalived_priority`）最高的节点持有 VIP。当该节点的 Nginx 异常时，VIP 自动漂移到下一个节点。

### 3. 测试连通性

```bash
ansible -i inventory/hosts.ini all -m ping
```

### 4. 执行一键部署

```bash
ansible-playbook -i inventory/hosts.ini playbooks/cluster.yml
```

单 Master 模式约 **3-5 分钟**，HA 模式约 **8-12 分钟** 完成部署。

### 5. 验证集群状态

登录到 Master 节点（HA 模式可登录任意一台）：

```bash
kubectl get nodes
```

所有节点状态为 `Ready` 即表示部署成功。

验证 IPVS 模式是否生效：

```bash
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode
ipvsadm -Ln
```

---

## 🏗️ HA 集群架构

```
用户请求 / Worker 节点
        │
        ▼
┌───────────────────┐
│  VIP: 192.168.x.x │  ← Keepalived 浮动 IP（自动漂移）
│  端口: 16443       │
└─────────┬─────────┘
          │
  ┌───────┴────────┐
  │  Nginx (每台    │  ← stream 模块轮询转发到后端 APIServer :6443
  │  Master 都有)   │
  └───────┬────────┘
          │
 ┌────────┼────────┐
 ▼        ▼        ▼
master1  master2  master3   ← etcd 集群，控制平面高可用
```

---

## ⚠️ 常见问题

- **网络拉取失败**：K8s 镜像位于 `registry.k8s.io`，国内服务器可能拉取失败。建议配置服务器代理或使用能访问外网的环境。
- **SSH 报错**：请确认 `hosts.ini` 中的用户名、密码正确，且目标机器的 sshd 已开放 root 密码登录。
- **HA 模式 VIP 冲突**：确保 `ha_vip_ip` 在局域网内没有其他设备在使用该 IP。
- **网卡名不对**：不同系统的网卡名不同（`eth0`、`ens18`、`ens3` 等），在 `all.yml` 中正确填写 `ha_vip_interface`，可用 `ip addr` 命令查看。
- **Rocky/CentOS 首次运行报错**：如果 SELinux 没有自动处理，可以手动先在所有节点执行 `setenforce 0`。
