## Setup

```bash
# Setup before exam
alias k=kubectl

# Kubectl completion
source <(kubectl completion bash)
complete -o default -F __start_kubectl k

# Một vài alias hữu ích
alias kgp='k get pods'
alias kgs='k get svc'
alias kgd='k get deploy'
alias kgn='k get nodes'
alias kdp='k describe pod'
alias kaf='k apply -f'
alias kdel='k delete'

# Biến thường dùng
export do='--dry-run=client -o yaml'

# This should already be in your .vimrc
set expandtab
set tabstop=2
set shiftwidth=2
```

## Kubectl command

```bash
# Pod with labels
k run nginx --image=nginx:1.27 --port=80 --labels=app=web,tier=frontend --env=LOG_LEVEL=debug --env=APP_ENV=prod

# Pod with command override
k run nginx --image=nginx:1.27 -- sh -c "echo 'Hello' && sleep 3600"

# Deployment with resource requests
k create deployment webapp --image=nginx:1.27 --replicas=3

# Expose deployment as ClusterIP (default, internal only)
k expose deployment webapp --port=80 --target-port=8080

# Create service without deploying (generate YAML)
k create service clusterip web --tcp=80:8080 $do > svc.yaml

# Expose deployment as ClusterIP (default, internal only)
k expose deployment webapp --name static-pod-service --port=80 --target-port=8080

# Expose as NodePort (accessible on all nodes)
k expose deployment webapp --name static-pod-service --port=80 --target-port=8080 --type=NodePort

# Expose as LoadBalancer (cloud-only)
k expose deployment webapp --name static-pod-service --port=80 --target-port=8080 --type=LoadBalancer

# Create ServiceAccount
k create sa my-app -n prod


# Create Role (allow specific verbs on specific resources)
k create role pod-reader --verb=get,list,watch --resource=pods -n prod

# Create RoleBinding (bind role to user/sa)
k create rolebinding read-pods --role=pod-reader --serviceaccount=prod:my-app -n prod

# ClusterRoleBinding
k create clusterrolebinding read-nodes --clusterrole=node-reader --serviceaccount=prod:my-app

# Check if user/SA has permission
k auth can-i list pods -n prod --as=system:serviceaccount:prod:my-app

# ConfigMap from literal values
k create configmap app-config --from-literal=LOG_LEVEL=debug --from-literal=DB_HOST=postgres.prod

# ConfigMap from file
k create configmap app-config --from-file=config.properties

-----------------------------------------------------------------
# Các folder cần biết
| Đường dẫn                    | Dùng khi nào           | Mức độ |
| ---------------------------- | ---------------------- | ------ |
| `/etc/kubernetes/`           | kubeadm, control plane | ★★★★★  |
| `/etc/kubernetes/manifests/` | Static Pods            | ★★★★★  |
| `/etc/kubernetes/pki/`       | Certificate            | ★★★★   |
| `/var/lib/etcd/`             | etcd data              | ★★★★★  |
| `/var/lib/kubelet/`          | Kubelet                | ★★★★   |
| `/etc/cni/net.d/`            | CNI config             | ★★★    |
| `/opt/cni/bin/`              | CNI binaries           | ★★★    |
| `/etc/systemd/system/`       | kubelet service        | ★★★    |
| `/var/log/`                  | Debug log              | ★★★    |
| `~/.kube/config`             | kubeconfig             | ★★★★★  |

-----------------------------------------------------------------

# Snapshot
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup.db --write-table

# Restore
ETCDCTL_API=3 etcdctl snapshot restore /tmp/etcd-backup.db \
  --data-dir=/var/lib/etcd-restored

# Find the static pod path
cat /var/lib/kubelet/config.yaml | grep staticPodPath
# Usually: /etc/kubernetes/manifests

-----------------------------------------------------------------
# 1. Update kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.35.0-1.1
sudo apt-mark hold kubeadm

# 2. Plan
sudo kubeadm upgrade plan

# 3. Apply
sudo kubeadm upgrade apply v1.35.0

# 4. Update kubelet + kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.35.0-1.1 kubectl=1.35.0-1.1
sudo apt-mark hold kubelet kubectl

# 5. Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
---------------------------------------------------------------
# 1. From control plane: drain the worker
k drain worker-1 --ignore-daemonsets --delete-emptydir-data

# 2. SSH to worker, update packages
sudo apt-mark unhold kubeadm kubelet kubectl
sudo apt-get update
sudo apt-get install -y kubeadm=1.35.0-1.1 kubelet=1.35.0-1.1 kubectl=1.35.0-1.1
sudo apt-mark hold kubeadm kubelet kubectl

# 3. Upgrade node
sudo kubeadm upgrade node

# 4. Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 5. From control plane: uncordon
k uncordon worker-1
--------------------------------------
# Mount configMap or secret
envFrom:
- configMapRef:
    name: app-config
- secretRef:
    name: db-creds
```

## Check list kubeadm setup
| Kiểm tra          | Lệnh                                             |
| ----------------- | ------------------------------------------------ |
| Swap              | `swapon --show`                                  |
| Container runtime | `systemctl status containerd`                    |
| CRI               | `crictl info`                                    |
| Overlay module    | `lsmod \| grep overlay`                          |
| br_netfilter      | `lsmod \| grep br_netfilter`                     |
| IPv4 Forward      | `sysctl net.ipv4.ip_forward`                     |
| bridge iptables   | `sysctl net.bridge.bridge-nf-call-iptables`      |
| cgroup driver     | `grep SystemdCgroup /etc/containerd/config.toml` |

**Backup:**
```bash
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

Where to find the cert paths: `cat /etc/kubernetes/manifests/etcd.yaml` and look for `--cert-file`, `--key-file`, `--trusted-ca-file`.

**Verify:**
```bash
ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup.db --write-table
```

**Restore:**
```bash
# 1. Restore to a new directory
ETCDCTL_API=3 etcdctl snapshot restore /tmp/etcd-backup.db \
  --data-dir=/var/lib/etcd-restored

# 2. Update etcd manifest to use the restored directory
sudo vi /etc/kubernetes/manifests/etcd.yaml
# Change --data-dir=/var/lib/etcd → --data-dir=/var/lib/etcd-restored
# Change hostPath path: /var/lib/etcd → /var/lib/etcd-restored

# 3. Wait for etcd to restart (it's a static pod)
# kubectl may be unresponsive for 
```