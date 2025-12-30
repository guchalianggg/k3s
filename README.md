

# 边缘计算 Serverless 平台搭建指南 (K3s + Knative)

**环境目标：** 在原生 Ubuntu (笔记本/物理机) 环境下，从零搭建基于 K3s 和 Knative (v1.20.0) 的无服务器边缘计算平台。

**⚠️ 网络环境重要提示 (非常关键)：**
1.  **推荐开启代理**：由于 K3s 需要拉取 GitHub (ghcr.io) 和 Google (gcr.io) 的镜像，不开启代理下载极慢或失败。
2.  **严禁使用 TUN 模式**：请确保你的代理软件（如 Clash/V2Ray）**仅开启系统代理**，**不要开启 TUN/增强模式/虚拟网卡模式**。
    *   *原因*：TUN 模式会接管所有流量，拦截 Kubernetes 的 CNI 插件通信，导致 Kourier 网关无限重启，Pod 无法连接 CoreDNS。

---

## 🛠️ 第一阶段：安装 K3s (基础设施层)

### 1. 安装 K3s
使用官方脚本一键安装，并禁用 Traefik（防止端口冲突）和设置配置文件权限。

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik --write-kubeconfig-mode 644" sh -
```
*   `--disable traefik`: **(关键)** 禁用 K3s 默认网关。Knative 需要使用自己的 Kourier 网关，两者共存会抢占 80/443 端口。
*   `--write-kubeconfig-mode 644`: 允许非 root 用户读取配置，方便使用 `kubectl`。

### 2. 配置 kubectl 访问权限
```bash
mkdir -p ~/.kube    
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
```

### 3. 验证集群状态
```bash
kubectl get nodes
```
*   **预期结果：** 节点状态显示为 `Ready`。

---

## 📦 第二阶段：部署 Knative Serving (平台层)

### 1. 安装核心组件 (v1.20.0)
依次安装 CRD、核心控制器和 Kourier 网关。

```bash
# 1. 安装 CRDs (资源定义)
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.20.0/serving-crds.yaml

# 2. 安装 Serving Core (核心控制器)
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.20.0/serving-core.yaml

# 3. 安装 Kourier (网关控制器)
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.20.0/kourier.yaml
```

### 2. 启用 Kourier 网关
告诉 Knative 使用 Kourier 作为默认的网络层。
```bash
kubectl patch configmap/config-network \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"ingress.class":"kourier.ingress.networking.knative.dev"}}'
```

### 3. 配置 Magic DNS (本地解析)
将所有域名强制解析到本地，解决没有公网负载均衡 IP 的问题。
```bash
kubectl patch configmap/config-domain \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"127.0.0.1.sslip.io":""}}'
```

### 4. 等待组件就绪
执行以下命令，观察所有组件状态：
```bash
kubectl get pods -A
```
*   **注意**：如果有 Pod 卡在 `ContainerCreating` 或 `ImagePullBackOff`，请立即参考**第四阶段**的解决方法。

---

## ✅ 第三阶段：部署与验证

### 1. 部署 Hello World 服务
```bash
# 如果没有安装 kn 命令行工具，请先下载或使用 kubectl apply -f yaml 的方式
kn service create helloworld-go --image ghcr.io/knative/helloworld-go:latest
```

### 2. 验证访问
等待服务创建完成后，获取 URL 并访问：

```bash
# 查看服务列表和 URL
kn service list

# 模拟访问 (Serverless 会自动冷启动)
curl http://helloworld-go.default.127.0.0.1.sslip.io
```
*   **预期输出：** `Hello World！`

---


## 🔧 第四阶段：常见问题与故障排除 (Troubleshooting)
大部分的问题应该都是网络问题，可能每个人的网络状况也不一样，仅供参考

### 🔴 问题一：镜像拉取失败 (ImagePullBackOff / ContainerCreating)
**现象**：Pod 一直无法启动，`kubectl describe pod` 显示拉取 `docker.io` 超时，或拉取 `gcr.io` / `ghcr.io` 被拦截（403 Forbidden）。
**原因**：国内网络环境无法连接 Docker Hub 及 Google 镜像源，且默认配置未走国内加速通道。

#### ✅ 解决方法 A：配置全局镜像加速（优先尝试）
解决 90% 的 Docker Hub 基础镜像（如 `pause`, `traefik`, `python` 等）拉取超时问题。

1.  **创建/编辑镜像配置文件**：
    ```bash
    sudo mkdir -p /etc/rancher/k3s
    sudo nano /etc/rancher/k3s/registries.yaml
    ```

2.  **写入国内加速源**（以 DaoCloud 为例）：
    ```yaml
    mirrors:
      "docker.io":
        endpoint:
          - "https://docker.m.daocloud.io"
          - "https://dockerproxy.com"
          - "https://mirror.baidubce.com"
    ```

3.  **重启 K3s 生效**：
    ```bash
    sudo systemctl restart k3s
    ```

#### ✅ 解决方法 B：手动搬运 + 标签伪装
如果配置了加速源后，**Knative 核心组件**（如 `queue-proxy`）依然因为 `gcr.io` 限制拉取失败，需使用**南京大学 (NJU)** 镜像源。

**步骤：**
1.  **确定缺少的镜像**：
    ```bash
    kubectl describe pod <pod-name> -n <namespace> | grep Image:
    ```
    *(假设缺失的是 `gcr.io/knative-releases/knative.dev/serving/cmd/queue:v1.16.0`)*

2.  **从南大源拉取** (将 `gcr.io` 替换为 `gcr.nju.edu.cn`)：
    ```bash
    sudo crictl pull gcr.nju.edu.cn/knative-releases/knative.dev/serving/cmd/queue:v1.16.0
    ```

3.  **改名 (打标签)**：
    这一步是为了欺骗 K3s，让它以为这就是它想要的那个 gcr 镜像。
    ```bash
    # 格式: sudo k3s ctr images tag <南大镜像> <目标原版镜像>
    sudo k3s ctr images tag \
      gcr.nju.edu.cn/knative-releases/knative.dev/serving/cmd/queue:v1.16.0 \
      gcr.io/knative-releases/knative.dev/serving/cmd/queue:v1.16.0
    ```

4.  **删除报错 Pod** (让它自动重建并直接使用本地镜像)：
    ```bash
    kubectl delete pod <pod-name> -n <namespace>
    ```

---

### 🔴 问题二：笔记本切换网络后集群瘫痪 (i/o timeout)
**现象**：重启电脑或切换 WiFi 后，Pod 报错 `dial tcp: lookup ... i/o timeout`，无法解析集群内部域名（如 `myservice.default.svc.cluster.local`），Activator 无法连接 Autoscaler。
**原因**：Ubuntu 的 NetworkManager 会根据连接的网络动态重写 `/etc/resolv.conf`，导致 K3s 的 CoreDNS 上游解析失效。

#### ✅ 解决方案：固化 DNS 配置

1.  **创建专属 DNS 文件**（锁定使用阿里云 DNS）：
    ```bash
    sudo mkdir -p /etc/rancher/k3s
    sudo sh -c 'echo "nameserver 223.5.5.5" > /etc/rancher/k3s/resolv.conf'
    ```

2.  **修改 K3s 配置文件**：
    编辑 `sudo nano /etc/rancher/k3s/config.yaml`，指定使用该独立文件：
    ```yaml
    resolv-conf: "/etc/rancher/k3s/resolv.conf"
    write-kubeconfig-mode: "644"
    disable:
      - traefik
    ```
    *(注：如果你已经有了 config.yaml，请只需添加 `resolv-conf` 那一行)*

3.  **重启 K3s**：
    ```bash
    sudo systemctl restart k3s
    ```
### 🔴 问题三：电脑重启后出现（CrashLoopBackOff/ CreateContainerError）
**现象/原因**：笔记本换关机后系统时间停滞，开机后 K3s 发现时间和证书不一致(<invalid> ago)
#### ✅ 解决方案：
 每次关机前使用systemctl stop k3s，手动停止k3s，开机后再start k3s；如果关机时忘记停止k3s，出现这个问题可以使用
    ```bash
    kubectl delete pod --all -A --grace-period=0 –force 
    ```

  来删除所有命名空间下的所有 Pod，让k3s重新创建建新的（带有正确时间戳和证书的）Pod，这个重新创建不需要重新拉源，因为我们本地已经有镜像源，所以恢复的很快。


