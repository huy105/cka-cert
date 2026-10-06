# CKS (Certified Kubernetes Security Specialist) — Tài liệu ôn thi

> Điều kiện: phải có chứng chỉ CKA còn hiệu lực. Thi 2 tiếng, ~15-16 tasks thực hành trên terminal thật (nhiều cluster/context), không trắc nghiệm.

## 0. Setup môi trường làm bài (làm ngay khi vào phòng thi)

```bash
# Alias bắt buộc phải có, gõ tay hàng chục lần trong bài thi
alias k=kubectl
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"

# Luôn kiểm tra & chuyển đúng context/cluster trước MỖI câu — đề có nhiều cluster
kubectl config get-contexts
kubectl config use-context <context-name>
kubectl config current-context   # confirm lại trước khi làm

# Bật auto-completion + vim mode nếu quen
source <(kubectl completion bash)
complete -F __start_kubectl k
set -o vi   # nếu quen vim trong bash
```

- Mỗi câu hỏi thường ghi rõ context/node nào cần dùng — đọc kỹ dòng đầu tiên, sai context là mất điểm oan dù làm đúng.
- SSH sang node khác khi đề yêu cầu: `ssh node01`, xong việc nhớ `exit` về lại controlplane trước khi làm câu tiếp theo.
- Dùng `vim` với `set nu`, `set expandtab`, `set shiftwidth=2` (thường đã có sẵn `~/.vimrc` do đề cấu hình, kiểm tra lại `cat ~/.vimrc`).
- Trang tài liệu được phép dùng khi thi: kubernetes.io/docs, kubernetes.io/blog, github.com/kubernetes, github.com/falcosecurity, github.com/aquasecurity (trivy), github.com/cncf, gvisor.dev, kata-containers.org. Luyện thao tác search nhanh trong các trang này trước khi thi — mở sẵn tab, dùng Ctrl+F tìm keyword thay vì đọc lướt.
- Dùng nút **flag/bookmark** trên giao diện thi cho câu khó, quay lại cuối giờ — đừng dừng lại quá lâu ở 1 câu.
- Cuối mỗi mục có khối **🔗 Tra cứu khi thi**: ✅ = thuộc domain được phép mở trong phòng thi (kubernetes.io/docs, kubernetes.io/blog, falco.org/docs, trivy docs, etcd.io/docs, AppArmor wiki); 📖 = chỉ để ôn trước ở nhà, nhiều khả năng **không** mở được khi thi → nắm cú pháp trước.

**🔗 Tra cứu khi thi:**
- ✅ [kubectl Quick Reference (alias, completion, jsonpath)](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- ✅ [kubectl command reference (cú pháp từng lệnh)](https://kubernetes.io/docs/reference/kubectl/generated/)
- 📖 [LF — danh sách tài liệu được phép mở khi thi (kiểm tra lại trước ngày thi)](https://docs.linuxfoundation.org/tc-docs/certification/certification-resources-allowed)

---

> **Tỷ trọng domain chính thức (CNCF Curriculum v1.34, xem [nguồn](https://github.com/cncf/curriculum/blob/master/CKS_Curriculum%20v1.34.pdf)):** Cluster Setup 15%, Cluster Hardening 15%, System Hardening 10%, Minimize Microservice Vulnerabilities 20%, Supply Chain Security 20%, Monitoring/Logging/Runtime Security 20%. (Một số blog ghi 10%/15% lẫn lộn giữa Cluster Setup và System Hardening — bản dưới đây đã sửa theo đúng PDF gốc.)

## 0.5 Kiến thức nền cần biết trước (CKS đào sâu Linux/security hơn CKA)

CKA tập trung vào vận hành cluster (Deployment, Service, Storage...) và giả định bạn đã nắm phần đó rồi. CKS đào sâu thêm **lớp bảo mật bên dưới**, phần lớn nằm ở tầng Linux kernel và tầng admission control của API server — những thứ CKA gần như không đụng tới. Dưới đây là vài khái niệm nền, đọc trước khi vào các mục có ghi chú "(mới so với CKA)":

- **Linux Capabilities**: theo Unix truyền thống, user `root` (UID 0) có toàn quyền, user thường thì không có gì đặc biệt. Capabilities chia nhỏ "toàn quyền root" thành ~40 quyền riêng lẻ (VD `NET_BIND_SERVICE` = được bind port <1024, `SYS_ADMIN` = một mớ quyền quản trị hệ thống, `CHOWN` = đổi owner file...). Container mặc định chạy với 1 tập capabilities nhỏ hơn full root nhưng vẫn thừa so với nhu cầu thực tế của app — xem mục 3.3.
- **Seccomp (secure computing mode)**: cơ chế của kernel Linux để lọc syscall (lời gọi hệ thống, VD `open`, `read`, `execve`...) mà 1 process được phép gọi. Trong ~350 syscall của Linux, container thường chỉ cần vài chục cái — seccomp profile giới hạn lại để giảm bề mặt tấn công nếu container bị chiếm quyền. Xem mục 3.2.
- **AppArmor (Linux Security Module — LSM)**: LSM là framework của kernel cho phép gắn thêm "Mandatory Access Control" (MAC) — quy định ai được truy cập file/network gì, tách biệt với permission Unix cổ điển (chỉ theo owner/group/other). AppArmor là 1 LSM cụ thể (SELinux là LSM khác, không nằm trong CKS). Xem mục 3.1.
- **Admission Controller**: đoạn logic chạy trong API server, xen vào giữa lúc request đã authenticate/authorize xong nhưng **trước khi** được ghi vào etcd — có thể validate (chặn) hoặc mutate (sửa) request. Pod Security Admission (4.2), ImagePolicyWebhook (5.3), OPA/Gatekeeper (4.3) đều thuộc nhóm này — CKA hầu như không dùng tới cơ chế này.
- **Runtime security**: khác với các cơ chế trên (chặn *trước khi* container chạy), runtime security theo dõi hành vi **trong lúc container đang chạy** để phát hiện bất thường — đây là việc Falco làm (mục 6.1). Có thể hình dung như một IDS (Intrusion Detection System) dành cho container, bắt syscall qua kernel module hoặc eBPF.
- **Sandbox runtime (gVisor/Kata)**: container thông thường (runc) chia sẻ chung kernel với host — nếu container thoát được ra khỏi giới hạn (container escape), nó chạm thẳng tới kernel thật của node. gVisor/Kata chèn thêm 1 lớp cách ly giữa container và kernel host để giảm rủi ro này. Xem mục 6.4.

---

## 1. Cluster Setup (15%)

### 1.1 Network Policy — cú pháp chi tiết
NetworkPolicy chọn pod bằng `podSelector`, sau đó whitelist nguồn/đích qua `ingress`/`egress`. **Mặc định K8s cho phép mọi traffic** — NetworkPolicy chỉ có tác dụng khi có CNI hỗ trợ (Calico, Cilium, Weave...; **Flannel không hỗ trợ**).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: myns
spec:
  podSelector: {}          # {} = áp dụng cho MỌI pod trong namespace
  policyTypes: [Ingress, Egress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: myns
spec:
  podSelector:
    matchLabels: {app: backend}     # policy áp lên pod có label này
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - podSelector: {matchLabels: {app: frontend}}       # pod cùng namespace
        - namespaceSelector: {matchLabels: {env: prod}}      # pod ở namespace khác có label env=prod
        - ipBlock:
            cidr: 10.0.0.0/24
            except: [10.0.0.5/32]
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector: {matchLabels: {app: database}}
      ports: [{protocol: TCP, port: 5432}]
    # LƯU Ý: nếu có egress rule, phải mở thêm DNS (port 53 UDP/TCP tới kube-system)
    # nếu không app sẽ không resolve được service name
    - to:
        - namespaceSelector: {}
      ports:
        - {protocol: UDP, port: 53}
        - {protocol: TCP, port: 53}
```

**Điểm hay bị bẫy trong đề:**
- `from` với nhiều item trong **cùng 1 list** = OR (khớp 1 trong các điều kiện). Nếu `podSelector` và `namespaceSelector` nằm **cùng 1 object** (không phải 2 item riêng) thì là AND (pod đó phải match cả 2 điều kiện, tức là "pod có label X, nằm trong namespace có label Y").
```yaml
  - from:
      - podSelector: {matchLabels: {app: frontend}}
        namespaceSelector: {matchLabels: {env: prod}}   # AND: cùng 1 gạch đầu dòng
```
- Không khai báo `policyTypes: [Egress]` mà có `egress:` thì rule egress **không có tác dụng** — phải khai rõ.
- `podSelector: {}` trong `spec` (không phải trong `from`) = tất cả pod; nhưng nếu để trống hẳn phần `ingress:`/`egress:` (không có key) = deny toàn bộ chiều đó.

```bash
kubectl apply -f netpol.yaml
kubectl describe networkpolicy deny-all -n myns
kubectl get netpol -n myns -o yaml
# Test policy hoạt động thật — luôn làm bước này để chắc chắn ăn điểm:
kubectl exec -n myns frontend-pod -- curl -m 3 backend-svc:8080
kubectl exec -n myns other-pod -- curl -m 3 backend-svc:8080   # phải timeout/fail
```

**🔗 Tra cứu khi thi:**
- ✅ [Network Policies — có YAML mẫu default-deny, ipBlock, namespaceSelector](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- ✅ [Declare Network Policy (ví dụ test bằng wget/curl)](https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/)

### 1.2 CIS Benchmark cho cluster (mới so với CKA)
> **Là gì:** CIS (Center for Internet Security) Benchmark là 1 bộ checklist bảo mật chuẩn, viết sẵn cho từng loại hệ thống (Kubernetes, Linux, Docker...). `kube-bench` là tool tự động chạy checklist "CIS Kubernetes Benchmark" và báo PASS/FAIL/WARN cho từng mục — không phải công cụ riêng của K8s, chỉ là script đối chiếu config hiện tại với best-practice đã biết trước (VD "anonymous-auth phải là false", "quyền file cert phải là 600"...).
```bash
kube-bench run --targets master,node,etcd
kube-bench run --targets master --check 1.2.1
kube-bench run --targets node --check 4.2.1
# đọc kỹ output [FAIL], remediation ghi sẵn lệnh cần sửa (thường là sửa flag trong static pod manifest)
```

**🔗 Tra cứu khi thi:**
- ✅ [kube-apiserver flags (tra flag khi sửa theo remediation)](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)
- ✅ [kubelet flags](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)
- ✅ [KubeletConfiguration (/var/lib/kubelet/config.yaml)](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
- 📖 [kube-bench README (lệnh run, --targets, --check)](https://github.com/aquasecurity/kube-bench)

### 1.3 Ingress với TLS
```bash
kubectl create secret tls my-tls --cert=tls.crt --key=tls.key -n myns
```
```yaml
spec:
  tls:
    - hosts: [myapp.example.com]
      secretName: my-tls
```
- Không expose Dashboard, etcd, kubelet API ra ngoài internet — kiểm tra Service type, tránh `NodePort`/`LoadBalancer` cho các resource nhạy cảm.
- Kiểm tra port đang listen: `netstat -tulpn` hoặc `ss -tulpn`, đối chiếu với danh sách port K8s cần (6443 apiserver, 2379-2380 etcd, 10250 kubelet, 10257/10259 controller-manager/scheduler).

**🔗 Tra cứu khi thi:**
- ✅ [Ingress — mục TLS](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls)
- ✅ [Secret kiểu kubernetes.io/tls](https://kubernetes.io/docs/concepts/configuration/secret/#tls-secrets)
- ✅ [kubectl create secret tls](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_secret_tls/)

### 1.4 Protect node metadata and endpoints
Trên cloud (AWS/GCP/Azure), instance metadata service (thường là IP `169.254.169.254`) trả về credentials/token nếu pod gọi trực tiếp được ra ngoài — đây là hướng tấn công leo quyền phổ biến (SSRF → lấy cloud credentials của node).
```yaml
# NetworkPolicy chặn pod gọi tới metadata IP
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: deny-metadata-access, namespace: myns}
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except: [169.254.169.254/32]
```
```bash
kubectl exec pod -- curl -m 2 http://169.254.169.254/latest/meta-data/   # phải bị chặn sau khi áp policy
# đồng thời đảm bảo kubelet không expose read-only port (10255) và API không cho anonymous request đọc node info
```

**🔗 Tra cứu khi thi:**
- ✅ [Ports and Protocols — danh sách port control-plane/worker](https://kubernetes.io/docs/reference/networking/ports-and-protocols/)
- ✅ [NetworkPolicy — ipBlock + except (chặn 169.254.169.254)](https://kubernetes.io/docs/concepts/services-networking/network-policies/#networkpolicy-resource)
- ✅ [Kubelet authentication/authorization (anonymous, webhook)](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/)

### 1.5 Verify platform binaries before deploying (mới so với CKA)
> **Vì sao quan trọng:** đây là 1 phần của "supply chain security" (xem thêm mục 5) — nếu binary `kubelet`/`kubeadm` tải về bị thay bằng bản đã chèn mã độc (do MITM lúc download, hoặc mirror bị compromise), chạy nó = chạy mã độc với quyền root trên node. Checksum (sha256) là 1 "vân tay" của file gốc do nhà phát hành công bố — so khớp checksum không đảm bảo file "tốt", chỉ đảm bảo file tải về **đúng y hệt** bản gốc, không bị chỉnh sửa giữa đường.

Trước khi cài `kubeadm`/`kubelet`/`kubectl` tải về, phải verify checksum để tránh binary bị chèn mã độc.
```bash
# tải binary + file checksum tương ứng
curl -LO "https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubelet"
curl -LO "https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubelet.sha256"

# so sánh checksum — 2 cách
echo "$(cat kubelet.sha256)  kubelet" | sha256sum --check
sha256sum kubelet   # rồi so tay với nội dung file .sha256

# kết quả mong đợi: "kubelet: OK"
```
Đề có thể cho sẵn 1 binary + checksum lệch, yêu cầu phát hiện rồi tải lại bản đúng.

**🔗 Tra cứu khi thi:**
- ✅ [Download Kubernetes — link binary + file .sha256](https://kubernetes.io/releases/download/)
- ✅ [Install kubectl — có ví dụ sha256sum --check](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
- ✅ [Verify Signed Kubernetes Artifacts](https://kubernetes.io/docs/tasks/administer-cluster/verify-signed-artifacts/)

---

## 2. Cluster Hardening (15%)

### 2.1 RBAC — cú pháp Role/ClusterRole chi tiết
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: {name: pod-reader, namespace: myns}
rules:
  - apiGroups: [""]              # "" = core API group
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]      # subresource — khai riêng
    verbs: ["get"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    resourceNames: ["my-deploy"]  # giới hạn đúng 1 object cụ thể
    verbs: ["get", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: {name: read-pods, namespace: myns}
subjects:
  - kind: User
    name: jane
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: myapp-sa
    namespace: myns
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```
- `ClusterRole` + `ClusterRoleBinding` = quyền toàn cluster (mọi namespace).
- `ClusterRole` + `RoleBinding` (trong 1 namespace) = quyền của ClusterRole đó nhưng chỉ áp dụng trong namespace đó — pattern hay dùng để tái sử dụng 1 ClusterRole cho nhiều namespace khác nhau.
- Wildcard `"*"` dùng được cho `apiGroups`, `resources`, `verbs` nhưng **tránh dùng trong đề thi** trừ khi đề yêu cầu rõ — giám khảo chấm theo least-privilege.

```bash
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n myns
kubectl create rolebinding read-pods --role=pod-reader --user=jane -n myns
kubectl create clusterrole node-reader --verb=get,list --resource=nodes
kubectl create clusterrolebinding node-read --clusterrole=node-reader --user=jane

# Kiểm tra quyền — lệnh dùng RẤT nhiều trong bài thi
kubectl auth can-i list pods --as=jane -n myns
kubectl auth can-i '*' '*' --as=system:serviceaccount:myns:default
kubectl auth can-i --list --as=jane -n myns
kubectl describe role pod-reader -n myns
kubectl describe rolebinding read-pods -n myns
```

**🔗 Tra cứu khi thi:**
- ✅ [Using RBAC Authorization — Role/ClusterRole/Binding mẫu, aggregated roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- ✅ [kubectl create role (--verb, --resource, --resource-name)](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_role/)
- ✅ [kubectl auth can-i (--as, --list)](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_auth/kubectl_auth_can-i/)
- ✅ [RBAC Good Practices — quyền nguy hiểm cần tránh](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)

### 2.2 ServiceAccount
```yaml
apiVersion: v1
kind: ServiceAccount
metadata: {name: myapp-sa, namespace: myns}
automountServiceAccountToken: false
---
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: myapp-sa
  automountServiceAccountToken: false   # có thể override ở cấp Pod, ưu tiên hơn cấp SA
```
```bash
kubectl create serviceaccount myapp-sa -n myns
kubectl get sa myapp-sa -n myns -o yaml
kubectl patch deployment myapp -n myns -p '{"spec":{"template":{"spec":{"automountServiceAccountToken":false}}}}'
```

**🔗 Tra cứu khi thi:**
- ✅ [Configure Service Accounts — automountServiceAccountToken, projected token](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
- ✅ [Service Accounts (concept)](https://kubernetes.io/docs/concepts/security/service-accounts/)
- ✅ [Managing Service Accounts (admin)](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)

### 2.3 Hardening API server / kubelet
```bash
# /etc/kubernetes/manifests/kube-apiserver.yaml — sửa trực tiếp, kubelet tự apply lại (static pod)
--anonymous-auth=false
--authorization-mode=Node,RBAC
--profiling=false
--enable-admission-plugins=NodeRestriction,EventRateLimit
--insecure-port=0                 # (bản cũ) đảm bảo không mở port không mã hoá
--tls-cert-file=/etc/kubernetes/pki/apiserver.crt
--tls-private-key-file=/etc/kubernetes/pki/apiserver.key

# theo dõi pod tự khởi động lại sau khi sửa file:
watch crictl ps   # hoặc: watch kubectl get pod -n kube-system

# /var/lib/kubelet/config.yaml
anonymous:
  enabled: false
authorization:
  mode: Webhook
readOnlyPort: 0
protectKernelDefaults: true
```
```bash
systemctl daemon-reload
systemctl restart kubelet
systemctl status kubelet
journalctl -u kubelet -f    # kiểm tra lỗi nếu kubelet không start được sau khi sửa config
```

**🔗 Tra cứu khi thi:**
- ✅ [kube-apiserver flags (--anonymous-auth, --authorization-mode, --enable-admission-plugins…)](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)
- ✅ [KubeletConfiguration — authentication/authorization/readOnlyPort](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
- ✅ [Kubelet authn/authz](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/)
- ✅ [Controlling Access to the Kubernetes API](https://kubernetes.io/docs/concepts/security/controlling-access/)
- ✅ [Securing a Cluster](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/)

### 2.4 Vô hiệu hoá API version / resource nguy hiểm
```bash
kubectl api-versions | grep policy
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis
```

**🔗 Tra cứu khi thi:**
- ✅ [API overview — bật/tắt API group bằng --runtime-config](https://kubernetes.io/docs/reference/using-api/#enabling-or-disabling)
- ✅ [Deprecated API Migration Guide](https://kubernetes.io/docs/reference/using-api/deprecation-guide/)

### 2.5 Upgrade Kubernetes để tránh lỗ hổng (kubeadm)
```bash
kubectl get nodes -o wide          # xem version hiện tại từng node
kubeadm upgrade plan               # xem version có thể lên + cảnh báo

# trên control-plane:
apt-get update && apt-get install -y kubeadm=1.31.1-1.1
kubeadm upgrade apply v1.31.1
kubectl drain <node> --ignore-daemonsets
apt-get install -y kubelet=1.31.1-1.1 kubectl=1.31.1-1.1
systemctl daemon-reload && systemctl restart kubelet
kubectl uncordon <node>

# trên worker node: chỉ cần "kubeadm upgrade node" (không cần "apply")
kubeadm upgrade node
```
Thứ tự bắt buộc: nâng `kubeadm` trước → `upgrade plan/apply` → drain node → nâng `kubelet`/`kubectl` → restart kubelet → uncordon. Làm control-plane trước, worker sau, từng node một.

**🔗 Tra cứu khi thi:**
- ✅ [Upgrading kubeadm clusters — copy lệnh từng bước](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- ✅ [Upgrading Linux worker nodes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/)
- ✅ [Changing the Kubernetes package repository (pkgs.k8s.io)](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/)
- ✅ [Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)

---

## 3. System Hardening (10%)

### 3.1 AppArmor — cách viết profile (mới so với CKA)
> **Là gì / vì sao cần:** permission Unix thường (rwx theo owner/group/other) không đủ chi tiết — 1 process chạy với quyền của user nào đó thì tự động có mọi quyền của user đó trên mọi file. AppArmor là 1 Linux Security Module (LSM), cho phép gắn thêm 1 lớp rule **theo từng chương trình cụ thể** (không phải theo user), kiểu "process X chỉ được đọc file A, dù user chạy nó có quyền đọc cả ổ đĩa". Đây gọi là Mandatory Access Control (MAC) — khác permission Unix (Discretionary Access Control). Trong K8s, AppArmor giới hạn 1 container dù bên trong container có bị chiếm quyền root thì cũng không escape ra ngoài phạm vi file/network đã whitelist.

Chạy trên **node**, không phải trên control plane API. Profile là file text đặt tại `/etc/apparmor.d/`.

```
#include <tunables/global>

profile k8s-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  file,                      # cho phép mọi thao tác file mặc định (base rule)
  deny /tmp/** w,             # CHẶN ghi vào /tmp và mọi thứ bên trong
  deny /etc/** w,
  network inet tcp,           # cho phép mở socket TCP
  capability net_bind_service,
}
```
Cú pháp cốt lõi cần nhớ: mỗi dòng kết thúc bằng dấu `,`; `deny` đứng trước để chặn; quyền file gồm `r` (read) `w` (write) `x`/`ux`/`Px` (execute); `**` = đệ quy mọi file con, `*` = 1 cấp.

```bash
apparmor_parser -q /etc/apparmor.d/k8s-deny-write   # load profile trên node
aa-status                                            # xem danh sách profile đã load + enforce/complain
```
Gắn vào pod bằng annotation (chuẩn cũ, vẫn ra trong đề) hoặc field `securityContext.appArmorProfile` (K8s 1.30+):
```yaml
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/mycontainer: localhost/k8s-deny-write
# hoặc bản mới:
spec:
  containers:
    - name: mycontainer
      securityContext:
        appArmorProfile:
          type: Localhost
          localhostProfile: k8s-deny-write
```
Verify: `kubectl exec pod -- touch /tmp/x` phải báo `Permission denied`.

**🔗 Tra cứu khi thi:**
- ✅ [Restrict a Container's Access to Resources with AppArmor — field appArmorProfile + annotation cũ](https://kubernetes.io/docs/tutorials/security/apparmor/)
- ✅ [AppArmor Documentation (cú pháp profile, apparmor_parser)](https://gitlab.com/apparmor/apparmor/-/wikis/Documentation)

### 3.2 Seccomp — cách viết profile JSON (mới so với CKA)
> **Là gì / vì sao cần:** mọi process muốn "nói chuyện" với kernel (đọc/ghi file, mở socket, tạo process con...) đều phải gọi syscall. Linux có khoảng 300-350 syscall, nhưng 1 app web thông thường chỉ dùng vài chục cái. Seccomp là bộ lọc của kernel chặn bớt syscall không cần thiết — nếu attacker khai thác được lỗ hổng và chèn code chạy trong container, code đó cũng bị giới hạn chỉ gọi được các syscall đã whitelist (VD không gọi được `ptrace`, `mount`, `reboot`...). Khác AppArmor (giới hạn theo *file/network path*), seccomp giới hạn theo *syscall* — 2 lớp bổ sung nhau, không thay thế nhau.

Profile JSON đặt tại `/var/lib/kubelet/seccomp/profiles/<tên>.json` trên node.

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["read", "write", "exit", "exit_group", "open", "close", "fstat", "mmap", "brk"],
      "action": "SCMP_ACT_ALLOW"
    },
    {
      "names": ["chmod", "chown", "setuid"],
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}
```
`defaultAction: SCMP_ACT_ERRNO` = whitelist mode (mặc định chặn, chỉ cho phép syscalls liệt kê). Có thể đảo lại `SCMP_ACT_ALLOW` mặc định + `SCMP_ACT_ERRNO` cho từng syscall cụ thể để làm blacklist.

```yaml
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault    # profile mặc định do container runtime cung cấp
  containers:
    - name: app
      securityContext:
        seccompProfile:
          type: Localhost
          localhostProfile: profiles/my-seccomp.json   # đường dẫn tương đối trong thư mục profiles ở trên
```
```bash
# strace để xem container thực sự gọi syscall nào trước khi viết whitelist (nếu đề cho phép)
kubectl exec pod -- cat /proc/1/status | grep Seccomp   # 2 = filtered (đang áp seccomp)
```

**🔗 Tra cứu khi thi:**
- ✅ [Restrict a Container's Syscalls with seccomp — có file profile JSON mẫu (audit/violation/fine-grained)](https://kubernetes.io/docs/tutorials/security/seccomp/)
- ✅ [Seccomp reference — field seccompProfile, đường dẫn /var/lib/kubelet/seccomp](https://kubernetes.io/docs/reference/node/seccomp/)

### 3.3 Capabilities / SecurityContext (mới so với CKA)
> **Là gì / vì sao cần:** container runtime mặc định cấp cho container 1 tập ~14 capabilities (không phải full root, nhưng cũng không phải "không có gì" — VD mặc định đã có `CHOWN`, `SETUID`, `NET_RAW`...). Best practice CKS là **drop hết (`ALL`) rồi chỉ add lại đúng cái app thật sự cần**, thay vì tin vào default của runtime. `runAsNonRoot`/`allowPrivilegeEscalation: false` chặn thêm 2 hướng khác: chạy bằng UID 0, và tự nâng quyền qua binary SUID bên trong image.

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]
```
```bash
kubectl exec pod -- whoami
kubectl exec pod -- id
kubectl get pod x -o jsonpath='{.spec.containers[0].securityContext}'
kubectl exec pod -- cat /proc/1/status | grep Cap   # đối chiếu bitmask capability nếu đề hỏi sâu
```

**🔗 Tra cứu khi thi:**
- ✅ [Configure a Security Context — runAsUser, capabilities, readOnlyRootFilesystem, allowPrivilegeEscalation](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- ✅ [Linux kernel security constraints for Pods and containers](https://kubernetes.io/docs/concepts/security/linux-kernel-security-constraints/)
- ✅ [Pod Security Standards — danh sách field "restricted" yêu cầu](https://kubernetes.io/docs/concepts/security/pod-security-standards/)

### 3.4 Linux cơ bản (điểm yếu nhiều người mất điểm)
```bash
useradd -m -s /bin/bash user1
passwd -l user1                       # khoá password login
usermod -aG sudo user1
chmod 600 /etc/shadow
chown root:root /etc/kubernetes/pki/*.key
systemctl status kubelet
systemctl enable --now kubelet
journalctl -u kubelet -f
iptables -L -n -v
iptables -A INPUT -p tcp --dport 4444 -j DROP
ss -tulpn
find / -perm -4000 -type f 2>/dev/null   # tìm SUID binaries
find / -nouser -o -nogroup 2>/dev/null   # file mồ côi owner
crontab -l -u root                        # kiểm tra cron job nghi ngờ
```

**🔗 Tra cứu khi thi:**
- ✅ [Security Checklist](https://kubernetes.io/docs/concepts/security/security-checklist/)
- ✅ [Hardening Guide — Authentication Mechanisms](https://kubernetes.io/docs/concepts/security/hardening-guide/authentication-mechanisms/)
- 📖 [Linux man pages (ss, systemctl, useradd…) — khi thi dùng `man`/`--help` trên terminal](https://man7.org/linux/man-pages/)

---

## 4. Minimize Microservice Vulnerabilities (20%)

### 4.1 Trivy — scan image
```bash
trivy image nginx:1.19
trivy image --severity HIGH,CRITICAL myrepo/myimage:tag
trivy image --exit-code 1 --severity CRITICAL myimage:tag   # dùng trong CI/admission, exit code != 0 để fail pipeline
trivy image --ignore-unfixed myimage:tag
trivy fs .                                                   # quét source code / Dockerfile trong thư mục
```

**🔗 Tra cứu khi thi:**
- ✅ [Trivy documentation (aquasecurity.github.io/trivy redirect về đây)](https://trivy.dev/latest/docs/)
- ✅ [Trivy — scan container image (--severity, --ignore-unfixed)](https://trivy.dev/latest/docs/target/container_image/)
- 📖 [Trivy GitHub](https://github.com/aquasecurity/trivy)

### 4.2 Pod Security Admission (thay PodSecurityPolicy) (mới so với CKA)
> **Là gì / vì sao cần:** đây là 1 **admission controller có sẵn** trong API server (không cần cài thêm gì), hoạt động bằng cách gắn label lên **namespace**. Nó không linh hoạt bằng OPA/Gatekeeper (mục 4.3, tự viết rule) nhưng dựng sẵn 3 bộ rule chuẩn nên nhanh gọn cho các case phổ biến (chặn privileged, chặn hostNetwork, bắt buộc non-root...). PodSecurityPolicy (PSP) là cơ chế cũ đã bị xoá khỏi K8s từ bản 1.25 — CKS hiện chỉ hỏi Pod Security Admission.

3 mức: `privileged` (không giới hạn) < `baseline` (chặn known privilege escalation cơ bản) < `restricted` (siết chặt nhất, bắt buộc non-root, drop ALL capabilities...).
```bash
kubectl label ns myns pod-security.kubernetes.io/enforce=restricted
kubectl label ns myns pod-security.kubernetes.io/enforce=baseline --overwrite
kubectl label ns myns pod-security.kubernetes.io/warn=restricted
kubectl label ns myns pod-security.kubernetes.io/audit=restricted
```
Verify: tạo pod vi phạm (VD `privileged: true`) trong namespace `enforce=restricted` → phải bị API server từ chối ngay khi `kubectl apply`.

**🔗 Tra cứu khi thi:**
- ✅ [Pod Security Admission — label enforce/audit/warn + version](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- ✅ [Pod Security Standards (privileged/baseline/restricted)](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- ✅ [Enforce Pod Security Standards with Namespace Labels](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/)
- ✅ [Enforce PSS bằng cấu hình admission controller (exemptions)](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-admission-controller/)

### 4.3 OPA / Gatekeeper — cách viết rule (Rego) (mới so với CKA)
> **Là gì / vì sao cần:** OPA (Open Policy Agent) là 1 engine đánh giá policy tổng quát (không riêng cho K8s); Gatekeeper là bản đóng gói OPA thành admission webhook cho K8s. Khác Pod Security Admission (rule cố định sẵn), Gatekeeper cho tự viết rule bằng ngôn ngữ Rego — dùng khi cần policy đặc thù mà 3 mức baseline/restricted/privileged không cover được (VD "bắt buộc phải có label team", "chỉ được pull image từ registry nội bộ"...). Cần phân biệt: OPA/Gatekeeper chặn **lúc apply** (trước khi resource được tạo); Falco (mục 6.1) phát hiện hành vi **lúc đang chạy** — 2 lớp khác thời điểm, bổ sung nhau chứ không thay thế.

Gatekeeper gồm 2 tầng: **ConstraintTemplate** (định nghĩa logic bằng Rego, giống như "class") và **Constraint** (áp dụng logic đó lên resource cụ thể, giống "instance").

```yaml
# 1. ConstraintTemplate — định nghĩa rule bằng Rego
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names: {kind: K8sRequiredLabels}
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels: {type: array, items: {type: string}}
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg}] {
          required := input.parameters.labels
          provided := input.review.object.metadata.labels
          missing := required[_]
          not provided[missing]
          msg := sprintf("Thiếu label bắt buộc: %v", [missing])
        }
---
# 2. Constraint — áp dụng ConstraintTemplate lên resource
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds: [{apiGroups: [""], kinds: ["Pod"]}]
    namespaces: ["myns"]        # (tuỳ chọn) giới hạn phạm vi
  enforcementAction: deny        # deny = chặn thật; dryrun = chỉ log không chặn
  parameters:
    labels: ["team"]
```
Cấu trúc Rego cần nhớ: `package <tên>` khai ở đầu; rule `violation[{"msg": msg}] { ... }` — mỗi dòng trong block là 1 điều kiện AND với nhau, nếu **tất cả** đúng thì rule "match" và trả về vi phạm; `input.review.object` là resource đang được xét; `not <biểu thức>` phủ định điều kiện.

Rule chặn image dùng tag `:latest` (mẫu hay gặp trong đề):
```rego
package k8sdisallowedtags

violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  endswith(container.image, ":latest")
  msg := sprintf("Image '%v' không được dùng tag latest", [container.image])
}
```
```bash
kubectl apply -f constrainttemplate.yaml
kubectl apply -f constraint.yaml
kubectl get constrainttemplates
kubectl get k8srequiredlabels                 # kind lấy theo spec.crd.spec.names.kind, viết thường
kubectl describe k8srequiredlabels require-team-label   # xem phần Status > Violations nếu enforcementAction: dryrun
```

**🔗 Tra cứu khi thi:**
- ✅ [K8s Blog — OPA Gatekeeper (có ví dụ ConstraintTemplate + Constraint, thuộc kubernetes.io/blog nên mở được)](https://kubernetes.io/blog/2019/08/06/opa-gatekeeper-policy-and-governance-for-kubernetes/)
- ✅ [Dynamic Admission Control (ValidatingWebhook)](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
- ✅ [ValidatingAdmissionPolicy (CEL) — phương án built-in thay Gatekeeper](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)
- 📖 [Gatekeeper How-to](https://open-policy-agent.github.io/gatekeeper/website/docs/howto)
- 📖 [Gatekeeper Library — rule mẫu (allowedrepos, requiredlabels…)](https://open-policy-agent.github.io/gatekeeper-library/website/)

### 4.4 Pod-to-Pod encryption (Cilium, Istio mTLS) (mới so với CKA)
> **Là gì / vì sao cần:** mặc định traffic giữa các pod đi dưới dạng plaintext trên mạng ảo của cluster — ai chiếm được 1 node hoặc bắt gói tin trên mạng đó là đọc được hết. mTLS (mutual TLS) nghĩa là **cả 2 chiều** (client lẫn server) đều xuất trình chứng chỉ để xác thực lẫn nhau, khác TLS thường (chỉ server có cert, client thì không) — dùng mTLS thì 2 service xác nhận đúng danh tính của nhau trước khi trao đổi dữ liệu đã mã hoá. Cilium mã hoá ở tầng network (WireGuard, trong suốt với app); Istio mTLS làm ở tầng sidecar proxy (Envoy) chèn vào mỗi pod trong mesh — 2 cách tiếp cận khác layer, không bắt buộc dùng cả 2.

K8s không tự mã hoá traffic giữa các pod — cần CNI/service mesh hỗ trợ.

**Cilium (mã hoá tầng transparent, WireGuard):**
```bash
cilium config view | grep encryption
cilium config set encryption-type wireguard
kubectl -n kube-system rollout restart ds/cilium
cilium status | grep Encryption   # xác nhận "Encryption: Wireguard  [NodeEncryption: Disabled]"
```

**Istio (mTLS giữa các pod trong mesh):**
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata: {name: default, namespace: myns}
spec:
  mtls:
    mode: STRICT     # bắt buộc mTLS, từ chối traffic plaintext
```
```bash
kubectl apply -f peerauth.yaml
istioctl proxy-config secret <pod> -n myns   # xem chứng chỉ được sidecar cấp
# verify: gọi service mà không qua sidecar (curl trực tiếp) phải bị từ chối do STRICT mode
```

**🔗 Tra cứu khi thi:**
- 📖 [Cilium — WireGuard transparent encryption](https://docs.cilium.io/en/stable/security/network/encryption-wireguard/)
- 📖 [Cilium — IPsec transparent encryption](https://docs.cilium.io/en/stable/security/network/encryption-ipsec/)
- 📖 [Istio — Mutual TLS Migration (PeerAuthentication STRICT)](https://istio.io/latest/docs/tasks/security/authentication/mtls-migration/)
- 📖 [Istio — PeerAuthentication reference](https://istio.io/latest/docs/reference/config/security/peer_authentication/)
- ✅ [Manage TLS Certificates in a Cluster](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/)

### 4.5 Secrets
```bash
kubectl create secret generic db-cred --from-literal=user=admin --from-literal=pass=1234
kubectl create secret generic tls-secret --from-file=./tls.crt --from-file=./tls.key
kubectl get secret db-cred -o jsonpath='{.data.pass}' | base64 -d
```
- Mount secret dạng **volume** thay vì **env var** khi có thể (env var dễ lộ qua `kubectl describe`, log, process list; volume thì không).
- Không commit secret ở dạng plaintext trong manifest lưu trong git.

**🔗 Tra cứu khi thi:**
- ✅ [Secrets — các type, mount dạng volume/env, immutable](https://kubernetes.io/docs/concepts/configuration/secret/)
- ✅ [Good practices for Kubernetes Secrets](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)
- ✅ [Distribute Credentials Securely Using Secrets](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/)

---

## 5. Supply Chain Security (20%)

### 5.0 Hiểu và siết supply chain (SBOM, CI/CD, artifact repository) (mới so với CKA)
> **Là gì / vì sao cần:** "supply chain" ở đây là toàn bộ chuỗi từ lúc viết code → build image → push registry → deploy — mỗi bước đều là 1 điểm có thể bị chèn mã độc (dependency độc hại, image bị thay bằng bản giả trong registry, pipeline CI bị chiếm quyền...). SBOM (Software Bill of Materials) là bản "danh sách thành phần" của image, giống nhãn thành phần trên hộp thực phẩm — liệt kê image được build từ package/dependency nào, version bao nhiêu. Khi có 1 CVE mới công bố cho 1 thư viện cụ thể, có SBOM thì tra cứu ngay image nào đang dùng thư viện đó mà không cần scan lại từ đầu.

```bash
# sinh SBOM (Software Bill of Materials) cho image bằng syft
syft myrepo/myimage:tag -o json > sbom.json
syft myrepo/myimage:tag -o table

# hoặc dùng trivy để vừa sinh SBOM vừa scan luôn
trivy image --format cyclonedx --output sbom-cyclonedx.json myimage:tag

# giới hạn registry được phép pull image trong cluster — dùng admission (Gatekeeper) hoặc containerd config
```
```yaml
# Gatekeeper: chỉ cho phép pull từ registry nội bộ
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata: {name: repo-is-internal}
spec:
  match: {kinds: [{apiGroups: [""], kinds: ["Pod"]}]}
  parameters:
    repos: ["myregistry.internal/"]
```
- SBOM giúp trace được image build từ package/dependency nào — dùng để đối chiếu khi có CVE mới công bố.
- CI/CD: đảm bảo pipeline build image có bước scan (trivy) + sign (cosign) trước khi push lên registry, registry production chỉ nhận image đã ký.

**🔗 Tra cứu khi thi:**
- ✅ [Trivy — SBOM (--format cyclonedx/spdx-json)](https://trivy.dev/latest/docs/supply-chain/sbom/)
- ✅ [Verify Signed Kubernetes Artifacts (SBOM, cosign)](https://kubernetes.io/docs/tasks/administer-cluster/verify-signed-artifacts/)
- ✅ [Images — imagePullPolicy, digest, private registry](https://kubernetes.io/docs/concepts/containers/images/)
- 📖 [bom (kubernetes-sigs) — sinh SBOM SPDX](https://github.com/kubernetes-sigs/bom)
- 📖 [syft README](https://github.com/anchore/syft)
- 📖 [Gatekeeper Library — K8sAllowedRepos](https://open-policy-agent.github.io/gatekeeper-library/website/validation/allowedrepos/)

### 5.1 Image signing / verify (cosign) (mới so với CKA)
> **Là gì / vì sao cần:** signing (ký số) trả lời câu hỏi "image này có đúng do team mình build ra không, có bị ai thay đổi giữa đường không". Cơ chế: giữ 1 private key (`cosign.key`), ký lên digest của image; ai có `cosign.pub` (public key) verify được chữ ký đó mà không cần biết private key. Nếu attacker chèn 1 image độc hại vào registry (kể cả cùng tên tag), verify sẽ fail vì không ký được bằng private key thật. Đây là bước cuối trong chuỗi supply chain: scan (Trivy) tìm lỗ hổng đã biết → sign (cosign) đảm bảo nguồn gốc → registry/cluster chỉ chấp nhận image đã ký.

```bash
cosign generate-key-pair
cosign sign --key cosign.key myrepo/myimage:tag
cosign verify --key cosign.pub myrepo/myimage:tag
cosign verify --key cosign.pub myrepo/myimage:tag | jq .
```

**🔗 Tra cứu khi thi:**
- ✅ [Verify Signed Kubernetes Artifacts (có lệnh cosign verify mẫu)](https://kubernetes.io/docs/tasks/administer-cluster/verify-signed-artifacts/)
- 📖 [Sigstore — Signing containers with cosign](https://docs.sigstore.dev/cosign/signing/signing_with_containers/)

### 5.2 Static analysis manifest / Dockerfile
```bash
kubesec scan pod.yaml
kube-linter lint deployment.yaml
conftest test deployment.yaml -p policy/
hadolint Dockerfile
```
Ví dụ 1 policy Rego cho `conftest` chặn container chạy privileged:
```rego
package main

deny[msg] {
  input.kind == "Pod"
  container := input.spec.containers[_]
  container.securityContext.privileged == true
  msg := sprintf("Container '%v' không được chạy privileged", [container.name])
}
```

**🔗 Tra cứu khi thi:**
- ✅ [Pod Security Standards — checklist field cần soi khi review manifest](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- 📖 [kubesec](https://kubesec.io/)
- 📖 [conftest (Rego cho manifest/Dockerfile)](https://www.conftest.dev/)
- 📖 [hadolint (lint Dockerfile)](https://github.com/hadolint/hadolint)

### 5.3 ImagePolicyWebhook (admission controller chặn image không rõ nguồn gốc) (mới so với CKA)
> **Là gì / vì sao cần:** đây là 1 admission controller sẵn có trong API server, nhưng thay vì tự chứa logic, nó gọi ra 1 webhook server bên ngoài để hỏi "image này có được phép chạy không" mỗi khi có pod mới được tạo. Điểm mấu chốt cần nhớ là `defaultAllow: false` — nghĩa là **fail-closed**: nếu webhook không trả lời được (down, timeout...) thì mặc định TỪ CHỐI thay vì cho qua, tránh trường hợp webhook chết làm mất luôn kiểm soát an ninh.

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
    - command:
        - kube-apiserver
        - --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
        - --admission-control-config-file=/etc/kubernetes/admission/admission-config.yaml
```
```yaml
# /etc/kubernetes/admission/admission-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
  - name: ImagePolicyWebhook
    configuration:
      imagePolicy:
        kubeConfigFile: /etc/kubernetes/admission/kubeconf-for-webhook.yaml
        allowTTL: 50
        denyTTL: 50
        retryBackoff: 500
        defaultAllow: false   # quan trọng: fail-closed, image không xác định được thì DENY
```

**🔗 Tra cứu khi thi:**
- ✅ [Admission Controllers — mục ImagePolicyWebhook (có file AdmissionConfiguration + kubeconfig mẫu)](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#imagepolicywebhook)
- ✅ [kube-apiserver Admission (v1) config API](https://kubernetes.io/docs/reference/config-api/apiserver-config.v1/)
- ✅ [kubeconfig — format file backend webhook](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)

### 5.4 Best practice build image
```dockerfile
# multi-stage build để giảm attack surface
FROM golang:1.22 AS build
WORKDIR /app
COPY . .
RUN go build -o app .

FROM gcr.io/distroless/base
COPY --from=build /app/app /app
USER 1000
ENTRYPOINT ["/app"]
```
- Không để secrets trong layer (dùng `--secret` của BuildKit thay vì `ARG`/`ENV`).
- Pin version cụ thể, tránh `latest`; dùng digest (`image@sha256:...`) khi cần bất biến tuyệt đối.

**🔗 Tra cứu khi thi:**
- ✅ [Images — tag vs digest](https://kubernetes.io/docs/concepts/containers/images/)
- ✅ [Security Checklist — mục Images](https://kubernetes.io/docs/concepts/security/security-checklist/#images)
- 📖 [Docker — Dockerfile best practices](https://docs.docker.com/build/building/best-practices/)
- 📖 [Docker — Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)

---

## 6. Monitoring, Logging & Runtime Security (20%)

### 6.1 Falco — cách viết rule chi tiết (phần hay bị mất điểm nhất) (mới so với CKA)
> **Là gì / vì sao cần:** tất cả các cơ chế ở mục 3-5 (AppArmor, seccomp, OPA, Pod Security...) đều là "phòng ngừa" — chặn hoặc giới hạn *trước khi/trong lúc* container được tạo. Falco là lớp khác hẳn: nó chạy nền trên node (đọc syscall qua kernel module hoặc eBPF), theo dõi **những gì thực sự xảy ra bên trong container lúc runtime**, và bắn cảnh báo khi thấy hành vi khớp rule (VD tự dưng có ai `exec` shell vào container production, có process đọc `/etc/shadow`, có kết nối ra IP lạ...). Nó không chặn hành vi (mặc định), chỉ log/alert — vì vậy luôn cần kết hợp với các lớp phòng ngừa ở trên, không thay thế được.

**Cấu trúc 1 rule Falco:**
```yaml
- rule: <tên rule, duy nhất>
  desc: <mô tả>
  condition: <biểu thức boolean — điều kiện để trigger>
  output: <format log khi trigger, dùng field %xxx.yyy>
  priority: <EMERGENCY|ALERT|CRITICAL|ERROR|WARNING|NOTICE|INFO|DEBUG>
  tags: [tag1, tag2]        # tuỳ chọn
```

**3 loại object trong file rule:**
- `rule`: as trên — thứ thực sự trigger alert.
- `macro`: đoạn `condition` tái sử dụng được, đặt tên rồi gọi lại trong rule khác.
- `list`: danh sách giá trị (VD danh sách tên process) dùng trong condition.

```yaml
- macro: container
  condition: (container.id != host)

- list: shell_binaries
  items: [bash, sh, zsh, csh, ksh]

- macro: spawned_process
  condition: evt.type = execve and evt.dir = <

- rule: Shell trong container
  desc: Phát hiện có shell được exec bên trong container
  condition: >
    spawned_process and container
    and proc.name in (shell_binaries)
  output: >
    Shell được chạy trong container
    (user=%user.name container_id=%container.id container_name=%container.name
    image=%container.image.repository command=%proc.cmdline)
  priority: WARNING
  tags: [container, shell]
```

**Field hay dùng nhất khi viết `condition`/`output`:**
| Nhóm | Field | Ý nghĩa |
|---|---|---|
| Process | `proc.name`, `proc.cmdline`, `proc.pname` | tên process, dòng lệnh, process cha |
| Event | `evt.type`, `evt.dir` | loại syscall (execve, open, connect...), `<` = enter/exit |
| File | `fd.name`, `fd.directory`, `fd.type` | đường dẫn file/socket đang thao tác |
| Container | `container.id`, `container.name`, `container.image.repository` | thông tin container |
| Network | `fd.sip`, `fd.sport`, `fd.rip`, `fd.rport` | IP/port nguồn-đích khi có network event |
| User | `user.name`, `user.uid` | user thực thi |
| K8s | `k8s.pod.name`, `k8s.ns.name` | (khi có k8s metadata enrich) |

**Toán tử condition thường dùng:** `=`/`!=`, `in`, `contains`, `startswith`, `and`/`or`/`not`, so sánh field-với-field (`fd.name = proc.name` ví dụ minh hoạ, thực tế ít dùng).

**4 rule mẫu hay ra trong đề CKS — học thuộc dạng này:**

```yaml
# 1. Phát hiện ghi file dưới thư mục nhị phân hệ thống (thường có sẵn trong falco_rules.yaml, chỉ cần biết đọc)
- rule: Write below binary dir
  desc: Ghi file bên dưới /bin, /usr/bin, /sbin...
  condition: >
    bin_dir and evt.dir = < and open_write
    and not package_mgmt_procs
  output: "File ghi dưới thư mục binary (file=%fd.name proc=%proc.name)"
  priority: ERROR

# 2. Phát hiện đọc file nhạy cảm (/etc/shadow, config secret...)
- rule: Read sensitive file untrusted
  desc: Process không nằm trong whitelist đọc /etc/shadow hoặc /etc/passwd
  condition: >
    open_read and sensitive_files
    and not proc.name in (allowed_procs)
  output: "Đọc file nhạy cảm (user=%user.name file=%fd.name proc=%proc.name)"
  priority: WARNING

# 3. Phát hiện kết nối mạng ra ngoài không mong muốn từ container
- rule: Unexpected outbound connection
  desc: Container mở kết nối TCP ra ngoài dải IP cho phép
  condition: >
    outbound and container
    and not fd.sip in (allowed_outbound_ips)
  output: "Kết nối outbound bất thường (container=%container.name dest=%fd.rip:%fd.rport)"
  priority: NOTICE

# 4. Phát hiện chạy công cụ quản lý package trong container (dấu hiệu cài mã độc)
- rule: Launch Package Management Process in Container
  desc: apt/yum/apk chạy trong container lúc runtime
  condition: >
    spawned_process and container
    and proc.name in (apt, apt-get, yum, dnf, apk, dpkg)
  output: "Package management chạy trong container (proc=%proc.name container=%container.name)"
  priority: ERROR
```

**Nơi thêm rule + cách chạy:**
```bash
# thêm custom rule vào file local, KHÔNG sửa trực tiếp falco_rules.yaml gốc
vi /etc/falco/falco_rules.local.yaml

# file cấu hình chính khai đường dẫn include các rule file:
cat /etc/falco/falco.yaml | grep rules_file

falco --validate /etc/falco/falco_rules.local.yaml   # kiểm tra cú pháp trước khi apply
falco -r /etc/falco/falco_rules.local.yaml            # chạy thử với rule file cụ thể
journalctl -fu falco                                   # xem alert realtime (nếu chạy dạng systemd service)
# hoặc nếu Falco chạy dạng container/pod trong cluster:
kubectl logs -n falco -l app=falco -f
```
**Mẹo:** đề thường cho sẵn 1 rule gần đúng và yêu cầu sửa/hoàn thiện — đọc kỹ `condition` hiện có, chỉ thêm đúng phần thiếu (macro/list) thay vì viết lại từ đầu. Luôn `falco --validate` trước khi coi là xong.

**🔗 Tra cứu khi thi:**
- ✅ [Falco documentation](https://falco.org/docs/)
- ✅ [Falco — Rules basic elements (rule/macro/list)](https://falco.org/docs/concepts/rules/basic-elements/)
- ✅ [Falco — Supported fields (proc.name, fd.name, container.id…)](https://falco.org/docs/reference/rules/supported-fields/)
- ✅ [Falco — Output formatting](https://falco.org/docs/concepts/outputs/formatting/)
- ✅ [Falco — Override/append rule có sẵn](https://falco.org/docs/concepts/rules/overriding/)
- 📖 [falco_rules.yaml gốc (tham khảo macro có sẵn)](https://github.com/falcosecurity/rules/blob/main/rules/falco_rules.yaml)

### 6.2 Audit Logging (API server) — cách viết audit policy (mới so với CKA)
> **Là gì / vì sao cần:** audit log ghi lại **ai đã gọi API nào, lúc nào, làm gì** với API server — khác hẳn log ứng dụng hay log container. Dùng để trả lời câu hỏi kiểu "ai đã xoá deployment này lúc 2h sáng" khi điều tra sự cố. Vì log full request/response (`RequestResponse`) rất nặng, audit policy cho phép chọn mức log khác nhau theo từng resource/user (VD log ít với `kube-proxy` vì nó gọi API liên tục, log đầy đủ với thao tác `delete` trên `pods` ở namespace `prod`).

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages: ["RequestReceived"]     # bỏ bớt stage không cần để giảm dung lượng log
rules:
  - level: None                      # None = không log gì
    users: ["system:kube-proxy"]
    verbs: ["watch"]
  - level: Metadata                  # log metadata (ai, khi nào, resource nào) — KHÔNG log body request/response
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
  - level: Request                   # log thêm cả request body, không log response
    resources:
      - group: "apps"
        resources: ["deployments"]
  - level: RequestResponse           # log đầy đủ cả request lẫn response — nặng nhất
    resources:
      - group: ""
        resources: ["pods"]
    namespaces: ["prod"]
    verbs: ["create", "delete", "update", "patch"]
  - level: Metadata                  # rule cuối = catch-all cho mọi thứ còn lại
```
4 mức level (từ nhẹ đến nặng): `None` < `Metadata` < `Request` < `RequestResponse`. Rule được match theo thứ tự từ trên xuống, dừng ở rule đầu tiên khớp — nên đặt rule cụ thể trước, catch-all để cuối.

```bash
# gắn vào kube-apiserver.yaml
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
--audit-log-path=/var/log/kubernetes/audit.log
--audit-log-maxage=7
--audit-log-maxbackup=3
--audit-log-maxsize=100

# cần thêm hostPath volume mount audit-policy.yaml + thư mục log vào pod kube-apiserver (sửa volumes/volumeMounts)

tail -f /var/log/kubernetes/audit.log | jq .
cat /var/log/kubernetes/audit.log | jq 'select(.verb=="delete")'
```

**🔗 Tra cứu khi thi:**
- ✅ [Auditing — có audit-policy.yaml mẫu + flag/volume cho kube-apiserver](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/)
- ✅ [kube-apiserver Audit Configuration (v1) — field Policy/Rule](https://kubernetes.io/docs/reference/config-api/apiserver-audit.v1/)
- ✅ [kube-apiserver flags (--audit-log-maxage, --audit-log-maxbackup…)](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)

### 6.3 Encryption at Rest (Secrets trong etcd) (mới so với CKA)
> **Là gì / vì sao cần:** nhiều người nghĩ `kubectl create secret` là "an toàn" vì giá trị hiển thị base64 chứ không phải plaintext — nhưng base64 chỉ là encode, không phải mã hoá, ai đọc trực tiếp dữ liệu trong etcd (nơi K8s lưu mọi state) đều decode được ngay. Encryption at Rest mã hoá thật sự dữ liệu secret **trước khi** ghi xuống đĩa của etcd, dùng 1 key riêng (`aescbc` ở đây). Provider `identity: {}` để cuối cùng là fallback đọc dữ liệu **chưa** mã hoá (cho những secret tạo từ trước khi bật tính năng này) — không có nó, cluster sẽ không đọc được các secret cũ.

```yaml
# enc.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources: ["secrets"]
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-32-byte-key>   # tạo bằng: head -c 32 /dev/urandom | base64
      - identity: {}    # provider cuối = fallback đọc dữ liệu chưa mã hoá (cho phép đọc secret cũ)
```
```bash
head -c 32 /dev/urandom | base64     # sinh key
--encryption-provider-config=/etc/kubernetes/enc/enc.yaml   # thêm flag vào kube-apiserver.yaml
# nhớ mount volume chứa file enc.yaml vào pod apiserver

# verify đã mã hoá thật:
ETCDCTL_API=3 etcdctl get /registry/secrets/default/mysecret --prefix -w fields \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
# output phải thấy tiền tố "k8s:enc:aescbc:v1:key1" thay vì thấy giá trị secret dạng plaintext

# secret cũ tạo trước khi bật encryption cần re-write để được mã hoá:
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

**🔗 Tra cứu khi thi:**
- ✅ [Encrypting Confidential Data at Rest — EncryptionConfiguration mẫu, lệnh etcdctl verify, re-write secret](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
- ✅ [EncryptionConfiguration API](https://kubernetes.io/docs/reference/config-api/apiserver-config.v1/)
- ✅ [Operating etcd clusters (cert flag cho etcdctl)](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
- ✅ [etcd documentation](https://etcd.io/docs/)

### 6.4 Sandbox / RuntimeClass (gVisor, Kata) (mới so với CKA)
> **Là gì / vì sao cần:** container thông thường (runtime `runc`) là process bình thường trên host, chỉ bị giới hạn bằng namespace + cgroup — vẫn dùng chung 1 kernel Linux với host và với mọi container khác. Nếu có lỗ hổng kernel bị khai thác (container escape), attacker chạm được thẳng tới host thật. `gVisor` (Google) chèn 1 kernel giả lập ở user-space chặn giữa container và kernel thật, chỉ cho qua các syscall đã kiểm tra; `Kata Containers` chạy hẳn container trong 1 VM nhẹ riêng (có kernel riêng). Cả 2 đánh đổi hiệu năng lấy cách ly mạnh hơn — dùng cho workload không tin cậy (chạy code người dùng ngoài gửi lên, multi-tenant...). `RuntimeClass` là cách khai báo trong K8s để pod chọn dùng runtime nào.

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata: {name: gvisor}
handler: runsc
---
apiVersion: v1
kind: Pod
spec:
  runtimeClassName: gvisor
```
```bash
kubectl get runtimeclass
crictl info | grep -A5 runsc
kubectl exec pod -- dmesg | grep -i gvisor   # xác nhận thực sự chạy trong sandbox
```

**🔗 Tra cứu khi thi:**
- ✅ [Runtime Class — RuntimeClass YAML + runtimeClassName](https://kubernetes.io/docs/concepts/containers/runtime-class/)
- 📖 [gVisor — containerd quick start (runsc handler)](https://gvisor.dev/docs/user_guide/containerd/quick_start/)
- 📖 [Kata Containers docs](https://github.com/kata-containers/kata-containers/tree/main/docs)

### 6.5 Hạn chế exec/attach vào pod production
```yaml
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["pods/exec", "pods/attach"]
    verbs: []          # không cấp verb "create" cho subresource exec/attach = chặn hoàn toàn
```
```bash
kubectl auth can-i create pods/exec --as=jane -n prod   # phải trả về "no"
```

**🔗 Tra cứu khi thi:**
- ✅ [RBAC — subresource pods/exec, pods/attach](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#referring-to-resources)
- ✅ [Admission Controllers (DenyServiceExternalIPs, NodeRestriction…)](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- ✅ [Get a Shell to a Running Container](https://kubernetes.io/docs/tasks/debug/debug-application/get-shell-running-container/)

---

## 7. Lộ trình luyện tập đề xuất

1. **KodeKloud CKS course** (video + 20+ labs) → làm hết trước khi qua bước 2.
2. **Killercoda CKS scenarios** (miễn phí, ~40 bài theo từng domain) — làm mỗi bài 2-3 lần tới khi không cần xem đáp án.
3. **killer.sh** (2 lượt được tặng kèm khi đăng ký thi) — làm thử full 2 tiếng, mục tiêu đạt ≥90% trước khi thi thật.
4. Ôn lại Linux cơ bản: user/group, permission, systemd, iptables/ss, SUID.
5. Học thuộc cấu trúc rule ở mục 6.1 (Falco) và 4.3 (Rego/OPA) — đây là 2 phần "viết từ đầu" khó nhất, không có sẵn generator như `kubectl create`.
6. Trước ngày thi: đọc lại toàn bộ alias/setup ở mục 0, luyện gõ YAML netpol/securityContext/RuntimeClass/audit-policy thuộc lòng không cần tra doc.

## 8. Chiến thuật làm bài
- Đọc hết đề trước khi gõ lệnh, xác định đúng **context/cluster/namespace**.
- Câu dễ (RBAC, NetworkPolicy, SecurityContext) làm trước — 3-5 phút/câu.
- Câu khó/tốn thời gian (viết Falco rule, audit policy, encryption) để cuối, đánh dấu bookmark (nút flag trong giao diện thi) để quay lại.
- Với câu Falco: nếu đề cho sẵn rule gần đúng, chỉ sửa phần thiếu — đừng viết lại từ đầu; luôn `falco --validate` trước khi coi là xong.
- Làm xong luôn **verify lại** (curl thử NetworkPolicy, `kubectl auth can-i`, đọc log Falco, `etcdctl get` kiểm tra encryption...) — nhiều task chấm theo hành vi thực tế, không chỉ theo YAML tồn tại.
- Quản lý thời gian: không để 1 câu quá 12-15 phút nếu bí, bỏ qua rồi quay lại.
