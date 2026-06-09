# Kubernetes 集群手动部署 SOP

> 本文档与 Ansible 自动化脚本完全对应，帮助你理解每一步在做什么。
> 适用系统：Ubuntu 22.04 / 24.04（Debian 系）

---

## 集群规划

| 角色 | 主机名 | IP | 说明 |
|------|--------|-----|------|
| Master1 | master01 | 192.168.9.41 | 控制平面（第一个初始化） |
| Master2 | master02 | 192.168.9.42 | 控制平面 |
| Master3 | master03 | 192.168.9.43 | 控制平面 |
| Worker1 | worker01 | 192.168.9.44 | 工作节点 |
| Worker2 | worker02 | 192.168.9.45 | 工作节点 |
| Worker3 | worker03 | 192.168.9.46 | 工作节点 |
| VIP | — | 192.168.9.100 | Keepalived 浮动 IP |

---

## 阶段一：所有节点基础准备

> 对应 Ansible role: `common`
> 在**所有 6 台**机器上执行

### 1.1 关闭 Swap

Kubernetes 要求关闭 Swap，否则 kubelet 无法启动。

```bash
# 立即关闭
swapoff -a

# 永久关闭：注释掉 /etc/fstab 中的 swap 行
sed -i '/\sswap\s/s/^/#/' /etc/fstab
```

**为什么？** K8s 的调度器需要精确管理内存，Swap 会导致内存统计不准确，影响 Pod 的资源限制和 OOM Kill 判断。

### 1.2 加载内核模块

```bash
# 写入开机自动加载的模块列表
cat > /etc/modules-load.d/k8s.conf << EOF
overlay
br_netfilter
EOF

# 立即加载
modprobe overlay
modprobe br_netfilter
```

**为什么？**
- `overlay`：容器存储驱动（OverlayFS）需要
- `br_netfilter`：让 Linux 网桥上的流量经过 iptables 处理，K8s 网络插件需要

### 1.3 配置内核网络参数

```bash
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 立即生效
sysctl --system
```

**为什么？**
- `bridge-nf-call-iptables`：让网桥流量走 iptables，Service 转发才能正常工作
- `ip_forward`：允许 IP 转发，Pod 跨节点通信必需

### 1.4 加载 IPVS 内核模块

```bash
# 写入 IPVS 模块列表
cat > /etc/modules-load.d/ipvs.conf << EOF
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF

# 立即加载
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack

# 安装管理工具
apt update && apt install -y ipvsadm ipset
```

**为什么？** kube-proxy 默认使用 iptables 模式，IPVS 模式性能更好（内核级哈希查找 vs 逐条匹配规则），适合生产环境。

### 1.5 验证

```bash
# 确认 swap 已关闭
free -h | grep Swap    # Swap 行应该全是 0

# 确认内核模块已加载
lsmod | grep br_netfilter
lsmod | grep overlay
lsmod | grep ip_vs

# 确认 sysctl 参数
sysctl net.bridge.bridge-nf-call-iptables   # 应为 1
sysctl net.ipv4.ip_forward                  # 应为 1
```

---

## 阶段二：安装容器运行时 Containerd

> 对应 Ansible role: `containerd`
> 在**所有 6 台**机器上执行

### 2.1 安装 Containerd

```bash
apt update
apt install -y containerd
```

### 2.2 配置 Containerd

```bash
# 创建配置目录
mkdir -p /etc/containerd

# 生成默认配置
containerd config default > /etc/containerd/config.toml

# 关键修改：启用 SystemdCgroup（K8s 强烈推荐）
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# 重启使配置生效
systemctl restart containerd
systemctl enable containerd
```

**为什么要改 SystemdCgroup？** K8s 使用 systemd 作为 cgroup 驱动，如果 containerd 使用不同的 cgroup 驱动（cgroupfs），会导致资源管理混乱。两者必须一致。

### 2.3 验证

```bash
systemctl status containerd    # 应为 active (running)
```

---

## 阶段三：安装 Kubernetes 组件

> 对应 Ansible role: `kubernetes-base`
> 在**所有 6 台**机器上执行

### 3.1 添加 Kubernetes APT 仓库

```bash
# 安装依赖
apt install -y curl tar

# 添加 K8s 官方 GPG 密钥
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | \
    gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# 添加 APT 源
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
    https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /" | \
    tee /etc/apt/sources.list.d/kubernetes.list
```

### 3.2 安装三件套

```bash
apt update
apt install -y kubelet kubeadm kubectl

# 锁定版本，防止意外升级
apt-mark hold kubelet kubeadm kubectl

# 启动 kubelet（它会自动重启直到被 kubeadm 配置）
systemctl enable kubelet
systemctl start kubelet
```

**三件套是什么？**
- `kubelet`：运行在每个节点上的"工人"，负责管理容器和 Pod
- `kubeadm`：集群初始化/加入的"安装向导"
- `kubectl`：你用来与集群交互的"命令行客户端"

### 3.3 验证

```bash
kubeadm version
kubelet --version
kubectl version --client
```

---

## 阶段四：部署高可用组件（仅 Master 节点）

> 对应 Ansible roles: `nginx` + `keepalived`
> 仅在 **3 台 Master** 上执行
> ⚠️ 单 Master 模式可跳过此阶段

### 4.1 安装 Nginx（负载均衡）

```bash
apt install -y nginx libnginx-mod-stream
```

**为什么需要 `libnginx-mod-stream`？** Ubuntu 的 nginx 默认不包含 stream（TCP 四层代理）模块。没有它，nginx 无法做 TCP 转发。

### 4.2 配置 Nginx Stream 负载均衡

```bash
# 创建 stream 配置目录
mkdir -p /etc/nginx/stream.d

# 在 nginx.conf 末尾添加 stream 引入（注意：不能放在 http{} 块内部）
cat >> /etc/nginx/nginx.conf << 'EOF'
# BEGIN ANSIBLE MANAGED - K8S STREAM LB
stream {
    include /etc/nginx/stream.d/*.conf;
}
# END ANSIBLE MANAGED - K8S STREAM LB
EOF
```

### 4.3 创建 APIServer 负载均衡配置

```bash
cat > /etc/nginx/stream.d/k8s-apiserver-lb.conf << 'EOF'
upstream kubernetes_apiserver {
    least_conn;
    server 192.168.9.41:6443 max_fails=3 fail_timeout=10s;
    server 192.168.9.42:6443 max_fails=3 fail_timeout=10s;
    server 192.168.9.43:6443 max_fails=3 fail_timeout=10s;
}

server {
    listen 16443;
    proxy_pass kubernetes_apiserver;
    proxy_connect_timeout 10s;
    proxy_timeout 300s;
}
EOF

# 检查配置语法 & 重启
nginx -t
systemctl restart nginx
systemctl enable nginx
```

**端口为什么是 16443 而不是 6443？** 因为 kube-apiserver 自己要用 6443，Nginx 和它跑在同一台机器上，端口不能冲突。所以 Nginx 用 16443 接收流量，再转发到后端的 6443。

### 4.4 验证 Nginx

```bash
ss -tnlp | grep 16443    # 应能看到 nginx 在监听
```

### 4.5 安装 Keepalived（VIP 漂移）

```bash
apt install -y keepalived
```

### 4.6 创建健康检查脚本

```bash
cat > /etc/keepalived/check_nginx.sh << 'EOF'
#!/bin/bash
# 检查 Nginx 是否正常监听 16443 端口
ss -tnlp | grep -q ":16443" || exit 1
EOF

chmod +x /etc/keepalived/check_nginx.sh
```

### 4.7 配置 Keepalived

> ⚠️ 每台 Master 的 `state` 和 `priority` 不同！

**Master1 (192.168.9.41)：**

```bash
cat > /etc/keepalived/keepalived.conf << 'EOF'
global_defs {
    enable_script_security
    script_user root
}

vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 3
    weight  -20
    fall    2
    rise    2
}

vrrp_instance VI_1 {
    state MASTER          # ← Master1 是 MASTER
    interface ens33       # ← 改成你的实际网卡名
    virtual_router_id 51
    priority 100          # ← Master1 优先级最高
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass k8sha
    }

    virtual_ipaddress {
        192.168.9.100     # ← VIP 地址
    }

    track_script {
        check_nginx
    }
}
EOF
```

**Master2 (192.168.9.42)** — 改两处：

```
    state BACKUP          # ← 改为 BACKUP
    priority 90           # ← 优先级降低
```

**Master3 (192.168.9.43)** — 改两处：

```
    state BACKUP          # ← 改为 BACKUP
    priority 80           # ← 优先级最低
```

启动：

```bash
systemctl restart keepalived
systemctl enable keepalived
```

### 4.8 验证 HA 组件

```bash
# 查看 VIP 是否在 Master1 上
ip addr show ens33 | grep 192.168.9.100

# 测试 VIP 连通性（从任意节点 ping）
ping -c 3 192.168.9.100
```

---

## 阶段五：初始化第一个 Master

> 对应 Ansible role: `master`（第一个节点部分）
> 仅在 **Master1** 上执行

### 5.1 生成 kubeadm 默认配置文件

```bash
# 生成完整的默认配置文件
kubeadm config print init-defaults > /tmp/kubeadm-config.yaml
```

然后修改以下标注了 `# ← 修改` 的字段：

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 0.0.0.0       # ← 修改：改为 0.0.0.0 或本机 IP（如 192.168.9.41）
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: master01                   # ← 修改：改为你的主机名
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta4
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
controlPlaneEndpoint: "192.168.9.100:16443"  # ← 新增：VIP:端口（HA 必须）
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.k8s.io
kind: ClusterConfiguration
kubernetesVersion: 1.36.0         # ← 修改：改为你的 K8s 版本
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16        # ← 新增：Flannel 要求此网段
scheduler: {}
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd              # ← 新增：与 containerd 的 cgroup 驱动保持一致
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs                         # ← 新增：使用 IPVS 代替默认的 iptables
```

> **💡 两种写法的区别：**
> - `kubeadm config print init-defaults` 生成完整配置 → 适合**手动部署**，所有字段一目了然
> - 只写需要改的字段，其余用默认值 → 适合 **Ansible 自动化**（模板简洁、跨版本兼容）
> - 两种方式效果完全一致，kubeadm 都会自动补全未指定的字段

**关键参数说明：**
- `controlPlaneEndpoint`：所有节点都通过这个地址访问 API Server。填的是 VIP:端口，而不是某台 Master 的 IP
- `mode: ipvs`：让 kube-proxy 使用 IPVS 而不是 iptables
- `podSubnet`：给 Flannel 用的，必须是 `10.244.0.0/16`
- `cgroupDriver: systemd`：必须与 containerd 的 cgroup 驱动一致。（注：自 K8s v1.22 起，kubeadm 默认已改用 systemd 作为 cgroup 驱动，若你使用的是较新版本，此项可省略，kubeadm 会自动应用默认值）

### 5.2 执行初始化

```bash
kubeadm init --config=/tmp/kubeadm-config.yaml --upload-certs
```

> 🕐 这一步大约需要 1-3 分钟，它会：
> 1. 拉取控制平面镜像（etcd, apiserver, scheduler, controller-manager）
> 2. 生成 CA 证书和各组件证书
> 3. 启动 etcd 和控制平面
> 4. 生成 bootstrap token

**⚠️ 重要：记下输出中的两条 join 命令！**

```
# Worker 加入命令（类似这样）：
kubeadm join 192.168.9.100:16443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:xxxxxxx

# Master 加入命令（多了 --control-plane 和 --certificate-key）：
kubeadm join 192.168.9.100:16443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:xxxxxxx \
    --control-plane --certificate-key xxxxxxx
```

### 5.3 配置 kubectl

```bash
mkdir -p /root/.kube
cp /etc/kubernetes/admin.conf /root/.kube/config
chmod 600 /root/.kube/config
```

### 5.4 部署 Flannel 网络插件

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

**为什么需要 CNI 插件？** kubeadm init 只启动了控制平面，但 Pod 之间还不能通信。CNI（容器网络接口）插件负责给每个 Pod 分配 IP 并建立跨节点的虚拟网络。没有它，节点状态会一直是 `NotReady`。

### 5.5 验证

```bash
kubectl get nodes          # Master1 应为 Ready
kubectl get pods -A        # 所有 Pod 应为 Running
```

---

## 阶段六：其余 Master 加入控制平面

> 对应 Ansible role: `master`（额外 Master 部分）
> 在 **Master2** 和 **Master3** 上分别执行

### 6.1 加入集群

```bash
# 使用第 5.2 步输出的 Master join 命令
kubeadm join 192.168.9.100:16443 --token <你的token> \
    --discovery-token-ca-cert-hash sha256:<你的hash> \
    --control-plane --certificate-key <你的certificate-key>
```

> 如果 token 过期了（默认 24 小时），在 Master1 上重新生成：
>
> ```bash
> # 生成新 token + join 命令
> kubeadm token create --print-join-command
>
> # 重新上传证书并获取 certificate-key
> kubeadm init phase upload-certs --upload-certs
> ```

### 6.2 配置 kubectl（每台额外 Master 都要做）

```bash
mkdir -p /root/.kube
cp /etc/kubernetes/admin.conf /root/.kube/config
chmod 600 /root/.kube/config
```

### 6.3 验证

```bash
# 在任意 Master 上执行
kubectl get nodes
# 应该能看到 3 个 Master 都是 Ready
```

---

## 阶段七：Worker 节点加入集群

> 对应 Ansible role: `worker`
> 在**所有 Worker** 节点上执行

### 7.1 加入集群

```bash
# 使用第 5.2 步输出的 Worker join 命令（不带 --control-plane）
kubeadm join 192.168.9.100:16443 --token <你的token> \
    --discovery-token-ca-cert-hash sha256:<你的hash>
```

### 7.2 验证（在任意 Master 上执行）

```bash
kubectl get nodes
```

预期输出：

```
NAME       STATUS   ROLES           AGE   VERSION
master01   Ready    control-plane   10m   v1.36.x
master02   Ready    control-plane   5m    v1.36.x
master03   Ready    control-plane   4m    v1.36.x
worker01   Ready    <none>          2m    v1.36.x
worker02   Ready    <none>          2m    v1.36.x
worker03   Ready    <none>          2m    v1.36.x
```

---

## 阶段八：最终验证

### 8.1 集群状态

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl cluster-info
```

### 8.2 IPVS 模式验证

```bash
# 确认 kube-proxy 使用 IPVS 模式
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode

# 查看 IPVS 规则
ipvsadm -Ln
```

### 8.3 HA 高可用验证

```bash
# 确认 VIP 在线
ping -c 3 192.168.9.100

# 确认 Nginx 负载均衡正常
ss -tnlp | grep 16443

# 模拟故障：在持有 VIP 的 Master 上停止 Nginx
systemctl stop nginx

# 等待几秒后，VIP 会自动漂移到另一台 Master
# 在另一台 Master 上验证：
ip addr show ens33 | grep 192.168.9.100

# 恢复 Nginx
systemctl start nginx
```

### 8.4 部署测试应用

```bash
# 部署一个 nginx 测试
kubectl create deployment nginx-test --image=nginx --replicas=3
kubectl expose deployment nginx-test --port=80 --type=NodePort
kubectl get svc nginx-test

# 访问测试（NodePort 端口从上面命令输出获取）
curl http://192.168.9.44:<NodePort>

# 清理
kubectl delete deployment nginx-test
kubectl delete svc nginx-test
```

---

## 附录：Ansible 自动化 vs 手动操作对照表

| 阶段 | Ansible Role | 手动步骤 | 执行节点 |
|------|-------------|---------|---------|
| 基础准备 | `common` | 关 swap、加载内核模块、sysctl、IPVS | 全部 6 台 |
| 容器运行时 | `containerd` | 安装+配置 containerd | 全部 6 台 |
| K8s 组件 | `kubernetes-base` | 添加 APT 源、安装三件套 | 全部 6 台 |
| 负载均衡 | `nginx` | 安装 nginx + stream 模块 + 配置 | 3 台 Master |
| VIP 漂移 | `keepalived` | 安装 keepalived + 配置 | 3 台 Master |
| 初始化 Master | `master` | kubeadm init + flannel + 提取 token | Master1 先，再 2/3 |
| Worker 加入 | `worker` | kubeadm join | 3 台 Worker |

> **手动部署一个 6 节点 HA 集群，你需要在各台机器上重复执行约 40+ 条命令。**
> **Ansible 自动化只需一条命令：** `ansible-playbook -i inventory/hosts.ini playbooks/cluster.yml`

---

## 附录：常用排障命令

```bash
# 查看 kubelet 日志（节点级别排障）
journalctl -xeu kubelet --no-pager -n 50

# 查看某个 Pod 的日志
kubectl logs <pod-name> -n <namespace>

# 查看 Pod 详情（调度失败、镜像拉取等）
kubectl describe pod <pod-name> -n <namespace>

# 重置节点（从头来过）
kubeadm reset -f
rm -rf /etc/kubernetes /root/.kube /etc/cni/net.d

# 查看 keepalived 日志
journalctl -xeu keepalived --no-pager -n 30

# 查看 nginx 日志
journalctl -xeu nginx --no-pager -n 30
nginx -t    # 检查配置语法
```
