# Ansible 核心概念与实战学习总结

基于你在部署 Kubernetes 过程中的体验，这篇总结将结合你已有的认知（幂等性、无代理、模块、主机清单、Playbook、Roles），补充进阶概念，并拆解本项目是如何利用 Ansible 跑起来的。

---

## 1. 核心概念回顾与补充

你的初步认识非常准确！这里为你完整梳理一遍，并补充几个你在本项目中实际用到、但可能还没意识到的高级特性：

### ✅ 你已经掌握的基础
- **无代理（Agentless）**：不需要在被控端（Master/Worker 节点）安装任何客户端，完全依赖 SSH 协议通信。
- **幂等性（Idempotency）**：执行一次和执行一万次的结果是一样的。比如告诉 Ansible “确保 Nginx 已安装”，如果没装它会装，如果装了它就什么都不做，直接返回 `OK`。
- **主机清单（Inventory）**：告诉 Ansible 要管理哪些机器（本项目中的 `hosts.ini`），可以对机器进行分组（如 `[kube_master]`、`[kube_node]`）。
- **模块（Modules）**：Ansible 执行任务的最小单元。相当于一个个现成的命令。比如 `apt` 模块装软件，`copy` 模块拷文件，`shell` 模块敲命令。
- **剧本（Playbook）**：把一堆模块的调用逻辑按照顺序写在 YAML 文件里，就像剧本一样指挥 Ansible 干活（本项目中的 `cluster.yml`）。
- **角色（Roles）**：把 Playbook 进一步“模块化”。把相关的 tasks、配置文件、变量全部打包在一个目录里，方便复用（比如本项目的 `nginx` role、`master` role）。

### 🚀 你需要补充的进阶概念（本项目大量使用）

#### 1. 变量与模板（Variables & Templates）
- **变量**：你可以定义全局变量（在 `group_vars/all.yml` 中），比如 `k8s_version`、`ha_vip_ip`。
- **模板（Jinja2）**：Ansible 的杀手级功能。你可以写一个配置文件模板（`.j2` 后缀），里面使用 `{{ 变量名 }}`。Ansible 拷贝到目标机器时，会自动把变量替换成真实的值。
  - *案例*：`keepalived.conf.j2` 中的 `{{ ha_vip_ip }}` 最终被渲染成了 `192.168.9.100`。

#### 2. 触发器（Handlers）
- **按需重启**：有些服务（比如 Nginx）只有在配置文件发生改变时才需要重启。
- **工作机制**：在 tasks 里写 `notify: Restart nginx`，然后在 `handlers/main.yml` 里定义如何重启。如果配置没变，任务是 `OK` 状态，触发器不会执行；如果配置变了，任务是 `Changed` 状态，执行完所有任务后会自动触发重启。

#### 3. 事实收集（Facts / Setup）
- 每次跑 Playbook 的第一步，Ansible 会默认执行一个隐含的 `Gathering Facts` 任务。它会去目标机器上收集系统信息（比如操作系统类型、IP 地址、CPU 架构等）。
- *案例*：我们的剧本里有 `when: ansible_os_family == "Debian"`。它就是通过 Facts 自动判断这台机器是 Ubuntu 还是 CentOS，从而决定用 `apt` 还是 `yum` 装软件。

#### 4. 条件判断与循环（Conditionals & Loops）
- `when` 语句用于条件判断。
  - *案例*：在 `cluster.yml` 中部署 Nginx 的条件是 `when: groups['kube_master'] | length > 1`（如果 Master 节点数大于 1 才部署高可用组件）。

---

## 2. 本项目（ansible-k8s）是如何运行的？

了解了概念，我们来看看这个项目跑起来的代码执行流：

### 第一步：读取入口配置
当你执行 `ansible-playbook -i inventory/hosts.ini playbooks/cluster.yml` 时，Ansible 首先读取：
1. `hosts.ini`：知道有 3 个 Master，3 个 Worker，并自动把它们划分到 `[k8s_cluster]` 总组里。
2. `group_vars/all.yml`：读取 K8s 版本、网段、VIP 等全局配置。

### 第二步：按 Playbook 顺序执行 Roles

打开 `playbooks/cluster.yml`，你会看到它被分为了 4 个大 Play（场景）：

1. **基础环境 Play**（针对所有节点 `hosts: k8s_cluster`）
   - 调用 `common` role：关 Swap、配 IPVS、改内核参数。
   - 调用 `containerd` role：安装并配置容器运行时。
   - 调用 `kubernetes-base` role：装好 kubeadm、kubelet、kubectl。

2. **高可用 Play**（针对 Master 节点 `hosts: kube_master`）
   - 判断：只有多于 1 个 Master 时才执行！
   - 调用 `nginx` role：配置 TCP 四层负载均衡转发 6443 端口。
   - 调用 `keepalived` role：配置 VIP 漂移。

3. **Master 初始化 Play**（针对 Master 节点 `hosts: kube_master`）
   - 这里有个关键参数 `serial: 1`：让 Ansible **一台一台地执行**，而不是同时执行并发。
   - Master1：生成配置 -> `kubeadm init` -> 部署 Flannel -> 生成 Join 命令并存为内部变量。
   - Master2/3：利用 Master1 传过来的 Join 命令和证书 Key 自动执行 `kubeadm join --control-plane`。

4. **Worker 加入 Play**（针对 Worker 节点 `hosts: kube_node`）
   - 利用 Master1 传过来的普通 Join 命令执行 `kubeadm join`。

---

## 3. 日常维护常用的 Ansible 命令（Ad-Hoc）

除了跑一键部署（Playbook），你平时还可以用 Ansible 当做“批量执行神器”（也就是 Ad-hoc 命令）：

**测试所有节点连通性：**
```bash
ansible -i inventory/hosts.ini all -m ping
```

**批量在所有节点执行 Shell 命令（比如重置集群）：**
```bash
ansible -i inventory/hosts.ini all -m shell -a "kubeadm reset -f"
```

**只查看某个节点的内存情况：**
```bash
ansible -i inventory/hosts.ini master1 -m command -a "free -h"
```

**批量拷贝文件到所有 Worker 节点：**
```bash
ansible -i inventory/hosts.ini kube_node -m copy -a "src=/tmp/test.txt dest=/opt/test.txt"
```

---

## 💡 总结

Ansible 最强大的地方在于：**它把复杂的运维逻辑（Shell 脚本、判断逻辑、配置修改）抽象成了可读性极高、声明式的 YAML 代码（Infrastructure as Code）**。

在 Kubernetes 部署这种复杂场景中，如果用 Shell 脚本，你要处理各种“如果装了怎么办”、“如果是 CentOS 怎么办”、“节点 1 生成了 Token 怎么传给节点 2” 的噩梦逻辑；而用 Ansible，你只需要定义“我要它达到什么状态（State）”，剩下的脏活累活全由 Ansible 自动帮你摆平。
