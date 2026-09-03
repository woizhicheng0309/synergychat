# WSL2 + Minikube `ImagePullBackOff` Debug Notes

## Symptom

Minikube 裡的 Pods 出現：

```text
ErrImagePull
ImagePullBackOff
```

## Debug Path

### 1. 查看 Pod Events

```bash
kubectl describe pod <pod-name>
```

看到 image pull 失敗：

```text
TLS handshake timeout
Client.Timeout exceeded while awaiting headers
```

所以問題不像是 Deployment YAML，而比較像 Docker Hub connectivity。

---

### 2. 測試 Docker Login

```bash
docker login
```

同樣出現 timeout。

因此開始往 network layer 排查。

---

### 3. IPv4 / IPv6 A/B Test

```bash
curl -4 https://registry-1.docker.io/v2/
curl -6 https://registry-1.docker.io/v2/
```

結果：

```text
IPv4 → 大量 timeout / reset，偶爾成功
IPv6 → 穩定回 HTTP 401 Unauthorized
```

`401 Unauthorized` 在這裡代表 Docker Registry connectivity 正常，只是 request 沒有 authentication。

---

### 4. 檢查 Minikube IPv6

```bash
minikube ssh
ip -6 route
```

Minikube node 沒有 usable IPv6 default route。

因此：

```text
WSL Host
├── IPv6 → Stable
└── IPv4 → Unstable

Minikube
├── IPv6 → Network is unreachable
└── IPv4 → Unstable
```

containerd pull image 時只能依賴有問題的 IPv4 path，因此產生 `ImagePullBackOff`。

---

### 5. 搜尋 WSL2 TCP Stall Regression

找到與目前症狀高度相似的 WSL2 IPv4 TCP stall regression。

其中 workaround 是關掉 Linux TCP timestamps：

```bash
sudo sysctl -w net.ipv4.tcp_timestamps=0
```

原本：

```text
net.ipv4.tcp_timestamps = 1
```

改成：

```text
net.ipv4.tcp_timestamps = 0
```

---

### 6. 驗證

修改前，IPv4 連續測試大量 timeout：

```text
FAILED
TLS timeout
HTTP=000
401
FAILED
...
```

修改後，IPv4 測試穩定回：

```text
HTTP=401
HTTP=401
HTTP=401
...
```

Minikube node 也需要設定：

```bash
minikube ssh -- 'sudo sh -c "echo 0 > /proc/sys/net/ipv4/tcp_timestamps"'
```

接著直接測 container runtime：

```bash
minikube ssh -- 'sudo crictl pull docker.io/bootdotdev/synergychat-web:latest'
```

成功 pull image。

最後：

```bash
kubectl get pods
```

所有 Pods：

```text
1/1 Running
```

---

## Final Debug Chain

```text
ImagePullBackOff
→ kubectl describe pod
→ Docker Hub timeout
→ docker login fail
→ IPv4 / IPv6 A/B test
→ IPv4 unstable / IPv6 stable
→ Minikube has no usable IPv6
→ search WSL2 TCP stall regression
→ disable TCP timestamps
→ IPv4 stable
→ crictl pull success
→ Pods Running
```

## Persistent Workaround

### WSL Host

`/etc/wsl.conf`

```ini
[boot]
command=sysctl -w net.ipv4.tcp_timestamps=0
systemd=true
```

### Minikube Node

```bash
minikube ssh -- 'sudo sh -c "echo net.ipv4.tcp_timestamps=0 > /etc/sysctl.d/99-wsl-tcp-workaround.conf"'
```

This persists across:

```bash
minikube stop
minikube start
```

But `minikube delete` removes the node, so the Minikube-side config must be recreated.

---

## Takeaway

The visible symptom was Kubernetes `ImagePullBackOff`, but the actual failure was much lower in the stack:

```text
Kubernetes
→ container runtime
→ HTTPS/TLS
→ TCP
→ WSL2 networking
```

The useful lesson was not the specific `sysctl` workaround, but the debugging process: keep moving down the dependency stack until the first failing layer is isolated.
