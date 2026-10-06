# CKA - Hướng dẫn các chủ đề chưa ôn

Dựa trên [first-attemp.txt](first-attemp.txt): template, cài CRI-O bằng deb, thay đổi request/limits, cài CNI, cài cluster, PriorityClass, HPA behavior.

---

## 1. PodTemplate & `spec.template`

Có 2 thứ hay bị nhầm:

**a) Resource kind `PodTemplate`** (ít dùng, nhưng đã từng xuất hiện trong killer.sh) — dùng để lưu sẵn 1 khuôn Pod, chủ yếu cho ReplicationController legacy.

```yaml
apiVersion: v1
kind: PodTemplate
metadata:
  name: nginx-template
template:
  metadata:
    labels:
      app: nginx
  spec:
    containers:
    - name: nginx
      image: nginx:1.27
```

```bash
k apply -f podtemplate.yaml
k get podtemplates
```

**b) Trường `.spec.template`** — mọi controller (Deployment, ReplicaSet, DaemonSet, StatefulSet, Job, CronJob) đều bọc PodSpec bên trong `spec.template`. Đây là chỗ hay bị hỏi sửa image/env/resources:

```bash
# Xem template hiện tại
k get deploy webapp -o jsonpath='{.spec.template.spec.containers[0].image}'

# Sửa nhanh image trong template
k set image deployment/webapp nginx=nginx:1.29

# CronJob lồng 2 lớp template: spec.jobTemplate.spec.template
k get cronjob backup -o yaml
```

Lưu ý: sửa `spec.template` của Deployment sẽ trigger rolling update (đổi hash pod-template-hash); sửa của Job/CronJob thì **không** áp dụng lại cho job đang chạy — Job là immutable ở phần template sau khi tạo (trừ vài field như `parallelism`, `activeDeadlineSeconds`).

---

## 2. Cài CRI-O bằng gói .deb (Ubuntu/Debian)

"Gói .deb" là đơn vị gói phần mềm của Debian/Ubuntu (khác `.rpm` bên CentOS/RHEL, hay build từ source). Có 2 cách cài, tùy đề thi/môi trường có internet hay không:

- **Cách A — qua APT repo (có internet):** khai báo 1 repo trỏ tới server chứa các file `.deb`, rồi `apt-get install` sẽ tự tải đúng file phù hợp và gọi `dpkg` cài, tự resolve dependency. Đây là cách guide dùng ở Bước 1 bên dưới.
- **Cách B — cài tay 1 file .deb có sẵn (offline / môi trường thi không internet):** không cần repo, chỉ cần có sẵn file `.deb` (đề thi cung cấp sẵn trong thư mục local, hoặc tự tải từ máy khác mang qua):
  ```bash
  sudo dpkg -i cri-o_1.30.0_amd64.deb
  # Nếu dpkg báo thiếu dependency, chạy lệnh này để apt tự vá:
  sudo apt-get install -f
  ```
  Môi trường thi CKA thực tế (killer.sh) thường **không có internet**, nên nếu đề bài đưa sẵn file `.deb`, dùng Cách B — không cần thêm repo qua mạng như Cách A.

### Bước 0 — Chọn đúng version còn tồn tại trên repo (tránh lỗi 404, chỉ áp dụng cho Cách A)

Repo OBS `isv:cri-o:stable` chỉ giữ vài bản CRI-O mới nhất — version cũ (vd `v1.30`) có thể đã bị gỡ khỏi repo, khiến `curl` báo `404` và `gpg` báo `no valid OpenPGP data found`. Luôn kiểm tra trước khi set biến:

```bash
# Liệt kê các version còn tồn tại trên repo
curl -s https://download.opensuse.org/repositories/isv:/cri-o:/stable/ | grep -oE 'v1\.[0-9]+' | sort -Vu
```

Chọn version khớp (hoặc gần nhất) với version Kubernetes đang cài — vd cluster dùng k8s v1.37 thì chọn CRI-O v1.36/v1.37 nếu có trong danh sách trên, không hardcode theo tài liệu cũ.

### Bước 1 — Cài đặt

```bash
# Biến đã bao gồm sẵn chữ "v" — khớp đúng tên thư mục trên repo
CRIO_VERSION=v1.36

# Thêm repo CRI-O (theo trang cri-o.io / OpenSUSE Build Service)
curl -fsSL https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key |
  sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$CRIO_VERSION/deb/ /" |
  sudo tee /etc/apt/sources.list.d/cri-o.list

sudo apt-get update
sudo apt-get install -y cri-o

sudo systemctl daemon-reload
sudo systemctl enable --now crio
sudo systemctl status crio
```

Nếu `curl -fsSL ... Release.key` vẫn trả rỗng/404 sau khi đổi version: kiểm tra lại chính xác tên biến đã export trong shell hiện tại (`echo $CRIO_VERSION`) — lỗi phổ biến khác là chạy lệnh `curl | gpg` ở terminal khác với terminal đã `export` biến, khiến `$CRIO_VERSION` rỗng và URL thành `.../stable://deb/...`.

### Bước 2 — Cấu hình: cài xong thì sửa ở đâu

Cài xong CRI-O chạy độc lập, chưa liên quan gì tới kubelet — phải trỏ kubelet sang nó. Có 4 chỗ cần biết, theo đúng thứ tự cần kiểm tra/sửa:

| # | File cần sửa | Mục đích | Sửa khi nào |
| - | ------------ | -------- | ----------- |
| 1 | `/etc/crio/crio.conf` (và `/etc/crio/crio.conf.d/*.conf`) | Cấu hình chính của CRI-O (cgroup driver, pause image, registries...) | Khi cần đổi `cgroup_manager`, pause image, hoặc thêm registry mirror |
| 2 | `/etc/containers/registries.conf` | Danh sách registry CRI-O được phép pull image | Khi bài thi yêu cầu pull từ registry riêng/insecure registry |
| 3 | `/var/lib/kubelet/config.yaml` (field `containerRuntimeEndpoint`) — hoặc flag `--container-runtime-endpoint` trong `/etc/systemd/system/kubelet.service.d/*.conf` | Nói cho **kubelet** biết dùng CRI-O thay vì containerd | Ngay sau khi cài CRI-O, trước khi `kubeadm init`/`join`, hoặc khi đổi runtime trên node đã chạy |
| 4 | `/etc/crictl.yaml` | Nói cho **crictl** (tool debug) biết socket nào để nói chuyện | Để dùng `crictl ps`, `crictl images` không cần gõ `--runtime-endpoint` mỗi lần |

Gói `.deb` sẽ tự tạo sẵn thư mục `/etc/crio/` — không cần tự `mkdir`. **Nhưng có 2 kiểu đóng gói khác nhau tùy version/repo, cần tự kiểm tra bằng `ls /etc/crio/` chứ đừng giả định:**
- **Kiểu 1:** có sẵn file `/etc/crio/crio.conf` với default value, thư mục `crio.conf.d/` rỗng.
- **Kiểu 2 (thực tế hay gặp, kể cả bản mới):** **không có** `crio.conf` — mọi default nằm gọn trong `/etc/crio/crio.conf.d/10-crio.conf`. Đây là kiểu ảnh chụp bên trên (`ls /etc/crio/crio.conf.d/` chỉ ra `10-crio.conf`, không có `crio.conf` ở ngoài).

Kiểm tra nhanh xem máy bạn thuộc kiểu nào:
```bash
ls -la /etc/crio/                 # có file crio.conf không, hay chỉ có crio.conf.d/
ls /etc/crio/crio.conf.d/         # xem tên các drop-in đã có sẵn
dpkg -L cri-o | grep /etc/crio    # xem package thực sự đã tạo file nào
```

**Sửa file 3 — trỏ kubelet sang CRI-O:**

```bash
sudo vi /var/lib/kubelet/config.yaml
```
```yaml
containerRuntimeEndpoint: unix:///var/run/crio/crio.sock
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

**Sửa file 4 — trỏ crictl sang CRI-O:**

```bash
sudo tee /etc/crictl.yaml <<CRIEOF
runtime-endpoint: unix:///var/run/crio/crio.sock
image-endpoint: unix:///var/run/crio/crio.sock
timeout: 10
debug: false
CRIEOF
```

**Sửa file 1 — đảm bảo cgroup driver khớp kubelet (thường phải là `systemd`):**

Không cần sửa hết mọi file, chỉ sửa đúng 1 chỗ đang thắng. Thứ tự đọc: CRI-O đọc `crio.conf` trước (nếu có), rồi đọc từng file trong `crio.conf.d/` theo thứ tự tên file, file đọc **sau cùng** ghi đè giá trị cùng key của mọi file đọc trước nó. Luôn kiểm tra trước khi sửa:

```bash
ls -la /etc/crio/                         # crio.conf có tồn tại không
ls /etc/crio/crio.conf.d/                 # danh sách drop-in đã có sẵn
sudo crio config | grep cgroup_manager    # giá trị effective cuối cùng (đã merge tất cả nguồn)
```

- **Nếu KHÔNG có `/etc/crio/crio.conf`** (trường hợp phổ biến — mọi default nằm trong `crio.conf.d/10-crio.conf`, giống ảnh chụp máy thật): sửa thẳng `10-crio.conf` đã có sẵn đó, không cần tạo `crio.conf` mới.
  ```bash
  sudo vi /etc/crio/crio.conf.d/10-crio.conf
  ```
  Tìm/thêm trong block `[crio.runtime]`:
  ```
  cgroup_manager = "systemd"
  ```
- **Nếu CÓ `crio.conf`** và `crio.conf.d/` rỗng → sửa thẳng `/etc/crio/crio.conf`, block `[crio.runtime]`, thêm dòng tương tự trên.
- **Nếu CÓ cả hai** và `crio.conf.d/` đã có file khác cũng set `cgroup_manager` → phải sửa đúng file đang thắng (file có tên sort sau cùng trong `crio.conf.d/`), sửa `crio.conf` sẽ vô tác dụng vì bị ghi đè.
- Cách an toàn nhất khi thi (khỏi cần xác định file nào đang thắng): luôn tạo/dùng 1 drop-in tên số lớn để chắc chắn đọc sau cùng — nếu `10-crio.conf` đã tồn tại thì tạo `99-cgroup-manager.conf` để ghi đè nó thay vì sửa trực tiếp:
  ```bash
  sudo tee /etc/crio/crio.conf.d/99-cgroup-manager.conf <<CGEOF
  [crio.runtime]
  cgroup_manager = "systemd"
  CGEOF
  ```

```bash
sudo systemctl restart crio
sudo crio config | grep cgroup_manager    # confirm lại giá trị effective đã đổi
```

### Bước 3 — Kiểm tra lại

```bash
sudo crictl info                                  # đọc được /etc/crictl.yaml, không cần --runtime-endpoint
sudo crictl ps -a
k get node -o wide                                # cột CONTAINER-RUNTIME phải là cri-o://x.y.z
```

Nếu node cũ đang đổi từ containerd sang CRI-O (không phải cài mới hoàn toàn), sau khi restart kubelet có thể cần `sudo kubeadm upgrade node` để control-plane cập nhật lại thông tin runtime của node đó.

### Bước 4 — Đổi hẳn từ containerd sang CRI-O trên node đang chạy (không phải cài mới)

Nếu node đã là thành viên cluster và đang chạy containerd (hoặc Docker), cài xong CRI-O (Bước 1) và trỏ kubelet (Bước 2) là **đủ về mặt kỹ thuật** (kubelet chỉ nói chuyện với 1 socket tại 1 thời điểm), nhưng nên làm đúng thứ tự sau để tránh downtime và trạng thái mập mờ giữa 2 runtime:

1. **Cordon + drain node trước** — di chuyển pod sang node khác trước khi đổi runtime, vì pod cũ do containerd tạo sẽ không tự chuyển sang CRI-O khi restart kubelet (chỉ ảnh hưởng container mới):
   ```bash
   k cordon <node>
   k drain <node> --ignore-daemonsets --delete-emptydir-data
   ```

2. **Cài CRI-O + cấu hình** theo đúng Bước 1 và Bước 2 ở trên (trỏ kubelet + crictl sang `crio.sock`, khớp `cgroup_manager`).

3. **Dừng containerd** để tránh 2 runtime cùng chạy song song, dễ nhầm khi debug bằng `crictl`:
   ```bash
   sudo systemctl stop containerd
   sudo systemctl disable containerd
   ```

4. **Restart kubelet** sau khi đã chắc chắn `containerRuntimeEndpoint` trỏ đúng `unix:///var/run/crio/crio.sock` (xem file 3 ở Bước 2):
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart kubelet
   ```

5. **Cập nhật lại thông tin node cho control-plane** (kubeadm lưu runtime cũ trong annotation của node lúc join/init):
   ```bash
   sudo kubeadm upgrade node
   ```

6. **Xác nhận rồi mới uncordon:**
   ```bash
   k get node <node> -o wide     # cột CONTAINER-RUNTIME phải đổi thành cri-o://x.y.z
   k uncordon <node>
   ```

Lưu ý: DaemonSet nào mount thẳng socket containerd (`/run/containerd/containerd.sock`) qua `hostPath` (vd 1 số CNI/CSI/monitoring agent cấu hình tay) sẽ lỗi sau khi đổi runtime — phải sửa lại path sang `/run/crio/crio.sock` hoặc dùng biến tự phát hiện runtime nếu chart hỗ trợ.

### Tài liệu tham khảo

- CRI-O install guide (theo OS): https://github.com/cri-o/cri-o/blob/main/install.md
- CRI-O official docs (cấu hình, `crio.conf`): https://github.com/cri-o/cri-o/blob/main/docs/crio.conf.5.md
- Kubernetes — Container Runtimes (chọn/đổi runtime, cgroup driver): https://kubernetes.io/docs/setup/production-environment/container-runtimes/
- kubeadm — Configuring cgroup driver: https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/
- `crictl` reference: https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/

---

## 3. Thay đổi Requests/Limits (Change Request, Limits)

### Tình huống thi: 1 Pod không lên (Pending) vì request quá cao

Đây là dạng bài rất hay gặp: có Deployment/Pod set `resources.requests` lớn hơn tài nguyên còn trống trên mọi node, scheduler không tìm được chỗ đặt pod, pod đứng mãi ở `Pending`. Việc cần làm là **giảm request cho khớp với tài nguyên thật có**, không phải tăng node.

**Cách tính request của scheduler:**
- Request của 1 Pod = **tổng** request của tất cả container trong pod (container thường + init container tính riêng, scheduler lấy `max(tổng container thường, từng init container)`).
- 1 node được chọn cho pod khi: `(tổng request của các pod đang chạy trên node) + (request của pod mới) <= Allocatable` của node đó (theo từng loại resource: cpu, memory riêng biệt).
- `Allocatable` **không phải** là tổng RAM/CPU vật lý — nó là `Capacity` trừ đi phần dành cho hệ điều hành + kubelet + runtime (`system-reserved`, `kube-reserved`, `eviction-hard`). Xem 2 giá trị này bằng `kubectl describe node <node>` (phần `Capacity:` và `Allocatable:`).
- Nếu pod không có `requests`, scheduler coi request = 0 cho mục đích xếp chỗ, nhưng nó vẫn được tính request bằng `limits` nếu `requests` bị thiếu nhưng `limits` có set (Kubernetes tự set requests = limits khi requests không khai báo).

**Chẩn đoán:**

```bash
# 1. Xem pod đang Pending, lý do gì
k get pods
k describe pod <pod>
# Events sẽ có dòng dạng:
# Warning  FailedScheduling  ... 0/3 nodes are available: 3 Insufficient cpu, 3 Insufficient memory.

# 2. Xem request pod đang đòi hỏi
k get pod <pod> -o jsonpath='{.spec.containers[*].resources.requests}'

# 3. Xem tài nguyên còn trống trên từng node
k describe node <node>
# Kéo xuống phần:
#   Capacity / Allocatable   -> tổng tài nguyên khả dụng
#   Allocated resources      -> đã có bao nhiêu request/limit đang chiếm chỗ, còn lại bao nhiêu %

# Cách khác, nhanh hơn để so sánh nhiều node cùng lúc:
k top nodes
k describe nodes | grep -A 5 "Allocated resources"
```

**Cách sửa:** hạ `requests` của pod/deployment xuống mức thấp hơn phần còn trống đã thấy ở bước chẩn đoán (xem các cách sửa a, b, c bên dưới). Nếu limits cũng đang set quá cao và request tự "ăn theo" limits (do không khai báo requests riêng), phải khai báo `requests` tường minh, thấp hơn `limits`.

**a) Sửa nhanh bằng `kubectl set resources`:**

```bash
k set resources deployment webapp \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=500m,memory=256Mi \
  -c webapp-container

# Xóa limit
k set resources deployment webapp --limits=cpu=0,memory=0
```

**b) Sửa trực tiếp qua edit/patch:**

```bash
k edit deployment webapp
# hoặc
k patch deployment webapp --type=json -p='[{"op":"replace","path":"/spec/template/spec/containers/0/resources/limits/memory","value":"512Mi"}]'
```

**c) In-place Pod resize (không cần restart Pod)** — feature từ 1.27 (alpha) → GA dần các version mới, hay bị hỏi trong bản thi mới nhất:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resize-demo
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "250m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
    resizePolicy:
    - resourceName: cpu
      restartPolicy: NotRequired
    - resourceName: memory
      restartPolicy: RestartContainer
```

```bash
# Resize pod đang chạy, không tạo lại pod
k patch pod resize-demo --subresource resize --patch '{"spec":{"containers":[{"name":"app","resources":{"requests":{"cpu":"500m"},"limits":{"cpu":"1"}}}]}}'

k get pod resize-demo -o jsonpath='{.status.containerStatuses[0].resources}'
```

**d) Liên quan LimitRange / ResourceQuota** (namespace-level defaults):

```bash
k get limitrange -n prod -o yaml
k get resourcequota -n prod
# Nếu pod bị reject do vượt quota, kiểm tra:
k describe resourcequota -n prod
```

Lỗi hay gặp: sửa Deployment mà quên namespace có LimitRange, khiến pod bị set default request/limit khác ý muốn, hoặc bị Forbidden nếu vượt ResourceQuota.

---

## 4. Cài đặt CNI

Sau khi `kubeadm init`, node ở trạng thái `NotReady` cho tới khi có CNI.

```bash
# Kiểm tra chưa có CNI
k get nodes            # STATUS: NotReady
k get pods -n kube-system   # coredns Pending
```

**Quy trình chuẩn: pull file về trước, sửa CIDR cho khớp, rồi mới apply** (không apply thẳng từ URL, vì thường phải sửa lại CIDR cho khớp `--pod-network-cidr` đã dùng lúc `kubeadm init`):

```bash
# 1. Pull manifest về máy (ví dụ Calico)
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml

# 2. Kiểm tra CIDR mặc định trong file có khớp --pod-network-cidr đã init không
grep -n "CALICO_IPV4POOL_CIDR" -A 2 calico.yaml

# 3. Sửa lại CIDR nếu khác (vd đổi 192.168.0.0/16 -> 10.244.0.0/16)
sed -i 's#192.168.0.0/16#10.244.0.0/16#' calico.yaml
# hoặc mở sửa tay:
vi calico.yaml

# 4. Apply file đã sửa
k apply -f calico.yaml
```

Tương tự với Flannel (CIDR mặc định `10.244.0.0/16`, sửa trong ConfigMap `kube-flannel-cfg` phần `net-conf.json` nếu khác) hoặc Weave Net:

```bash
# Flannel
curl -O https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
grep -n '"Network"' -A 1 kube-flannel.yml
sed -i 's#10.244.0.0/16#<CIDR-thuc-te>#' kube-flannel.yml
k apply -f kube-flannel.yml

# Weave Net
curl -L -O "https://github.com/weaveworks/weave/releases/download/latest_release/weave-daemonset-k8s.yaml"
k apply -f weave-daemonset-k8s.yaml
```

**Điểm quan trọng:**
- `--pod-network-cidr` lúc `kubeadm init` phải khớp CIDR trong manifest CNI (Calico mặc định `192.168.0.0/16`, Flannel mặc định `10.244.0.0/16`). Đây là lý do phải pull file về sửa trước, không apply thẳng từ URL.
- Sau khi apply CNI, verify:
  ```bash
  k get pods -n kube-system -o wide     # calico-node / kube-flannel chạy trên mọi node
  k get nodes                            # STATUS: Ready
  ls /etc/cni/net.d/                     # có file 10-calico.conflist hoặc tương tự
  ls /opt/cni/bin/                       # binary plugin
  ```
- Nếu 1 node mới join mà NotReady dù cluster khác Ready rồi thì thường do thiếu binary ở `/opt/cni/bin` trên node đó, hoặc container runtime socket sai — không phải thiếu CNI manifest (manifest apply 1 lần cho cả cluster qua DaemonSet).

---

## 5. Cài đặt Cluster (kubeadm init từ đầu)

**Trên control-plane node:**

```bash
# 1. Pre-flight (xem bảng checklist trong SELF_REMIND.MD)
swapon --show                 # phải trống (swap off)
sudo swapoff -a

# 2. Load kernel module + sysctl (bắt buộc trước khi init)
sudo tee /etc/modules-load.d/k8s.conf <<MODEOF
overlay
br_netfilter
MODEOF
sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/k8s.conf <<SYSEOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
SYSEOF
sudo sysctl --system

# 3. kubeadm init
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=<CONTROL_PLANE_IP> \
  --kubernetes-version=v1.30.0

# 4. Config kubectl cho user hiện tại
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 5. Cài CNI (xem mục 4)

# 6. Lưu lệnh join cho worker (in ra cuối output kubeadm init)
```

**Trên worker node:**

```bash
sudo kubeadm join <CONTROL_PLANE_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

**Nếu quên token / cần token mới:**

```bash
# Trên control-plane
kubeadm token create --print-join-command
```

**Verify:**

```bash
k get nodes -o wide
k get pods -A
k cluster-info
```

Đề thi thường test riêng lẻ từng bước (chỉ sysctl, chỉ init, chỉ join, chỉ CNI) chứ ít khi bắt làm hết từ đầu, nhưng nắm được toàn bộ flow giúp debug khi 1 bước bị thiếu.

---

## 6. PriorityClass

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: Uu tien cao cho service quan trong
```

```bash
k apply -f priorityclass.yaml
k get priorityclass
```

Gán vào Pod/Deployment:

```yaml
spec:
  template:
    spec:
      priorityClassName: high-priority
      containers:
      - name: app
        image: nginx
```

**Cơ chế cần nhớ:**
- `value` càng cao thì càng ưu tiên. Có 2 class hệ thống sẵn: `system-cluster-critical` và `system-node-critical` (giá trị rất cao, dùng cho control-plane/kube-system).
- Chỉ được đặt 1 PriorityClass với `globalDefault: true` trong cả cluster — pod nào không set `priorityClassName` sẽ nhận priority này.
- Khi cluster thiếu tài nguyên, scheduler có thể preempt (đuổi) pod priority thấp hơn để chỗ cho pod priority cao hơn, trừ khi pod bị đuổi có `preemptionPolicy: Never` ở PriorityClass của chính pod đó (set trên PriorityClass của pod cần bảo vệ, không phải pod đi chiếm chỗ).
- Debug: `k describe pod <pod>` sẽ có event Preempted nếu bị đuổi; xem field `.spec.priority` (giá trị số, tự động điền từ PriorityClass) và `.spec.priorityClassName`.

---

## 7. HPA Behavior

HorizontalPodAutoscaler mặc định scale up nhanh, scale down chậm (ổn định). Field `behavior` cho phép custom tốc độ này.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
      selectPolicy: Min
```

**Giải thích field:**
- `stabilizationWindowSeconds`: HPA nhìn lại metric trong cửa sổ thời gian này và chọn giá trị an toàn nhất (max cho scaleUp, min cho scaleDown) để tránh dao động liên tục. Mặc định: scaleUp = 0, scaleDown = 300s.
- `policies[].type`: `Pods` (số lượng pod tuyệt đối) hoặc `Percent` (phần trăm so với hiện tại).
- `selectPolicy`: `Max` (default cho scaleUp, scale nhanh nhất có thể), `Min` (default cho scaleDown, scale chậm nhất), hoặc `Disabled` (tắt hẳn scale theo hướng đó, ví dụ set `scaleDown.selectPolicy: Disabled` để không bao giờ scale down).

**Commands hữu ích:**

```bash
k autoscale deployment webapp --min=2 --max=10 --cpu-percent=60
k get hpa webapp-hpa -o yaml
k describe hpa webapp-hpa           # xem current/target metrics và events
k get hpa -w                        # theo dõi realtime

# Test load để trigger scale
k run -it load-generator --image=busybox --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://webapp; done"
```

**Lưu ý khi thi:** nếu không thấy scale dù CPU cao, kiểm tra metrics-server có chạy không (`k top pods` phải trả kết quả, không lỗi metrics not available), và Deployment target phải có `resources.requests` khai báo (HPA CPU % tính dựa trên request, không có request thì không tính % được).
