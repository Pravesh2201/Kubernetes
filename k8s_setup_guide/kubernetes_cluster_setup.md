# Kubernetes Cluster Setup — Commands & Screenshots

> **OS:** Ubuntu 22.04 (Jammy) | **K8s:** v1.32 | **Runtime:** Docker + cri-dockerd
> **Nodes:** 1 Master + 2 Workers on AWS EC2

---

## PART 1 — ALL NODES (Master + Worker Node 1 + Worker Node 2)

> ⚠️ Run every command in this section on **all 3 nodes**

---

### Step 1: Switch to root & update system

**What & Why:**
- `sudo su` — switches to root user so we don't need `sudo` before every command
- `apt-get update` — refreshes the list of available packages
- Installing `curl`, `gnupg`, `wget` — tools needed to download Kubernetes packages later

```bash
sudo su
apt-get update && apt-get upgrade -y
apt-get install -y apt-transport-https ca-certificates curl gnupg wget
```

---

### Step 2: Disable Swap

**What & Why:**
- Kubernetes **requires swap to be disabled**. If swap is enabled, kubelet will refuse to start.
- `swapoff -a` — disables swap immediately
- The `sed` command makes it permanent across reboots by commenting it out in `/etc/fstab`

```bash
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

---

### Step 3: Enable kernel modules & sysctl settings

**What & Why:**
- `overlay` and `br_netfilter` — kernel modules required for container networking
- The sysctl settings allow iptables to see bridged traffic, which is required for Kubernetes networking to work correctly between pods

```bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
```

---

### Step 4: Install Docker

**What & Why:**
- Docker is the **container runtime** — it actually runs the containers inside pods
- `daemon.json` configures Docker to use `systemd` as the cgroup driver, which must match kubelet's cgroup driver to avoid conflicts
- `overlay2` is the recommended storage driver for Ubuntu

```bash
apt-get install -y docker.io
systemctl start docker
systemctl enable docker

cat <<EOF | tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {"max-size": "100m"},
  "storage-driver": "overlay2"
}
EOF

systemctl daemon-reload
systemctl restart docker
```
<img width="684" height="878" alt="Screenshot 2026-04-03 at 7 27 46 PM" src="https://github.com/user-attachments/assets/900c90a8-5166-43af-a19f-847dff1a7d65" />

---

### Step 5: Install cri-dockerd

**What & Why:**
- Kubernetes v1.24+ removed built-in Docker support
- **cri-dockerd** is a shim/adapter that acts as a bridge between Kubernetes and Docker
- Without this, Kubernetes cannot communicate with Docker to create/manage containers

```bash
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.14/cri-dockerd_0.3.14.3-0.ubuntu-jammy_amd64.deb

dpkg -i cri-dockerd_0.3.14.3-0.ubuntu-jammy_amd64.deb

systemctl daemon-reload
systemctl enable cri-docker.service
systemctl enable cri-docker.socket
systemctl start cri-docker.service
```
<img width="1466" height="551" alt="Screenshot 2026-04-03 at 7 29 50 PM" src="https://github.com/user-attachments/assets/7cbe2ada-71bf-4d95-9622-9a23598db1f3" />


---

### Step 6: Install kubeadm, kubelet, kubectl

**What & Why:**
- `kubelet` — agent that runs on every node and manages containers
- `kubeadm` — tool used to bootstrap and initialize the cluster
- `kubectl` — command line tool to interact with and manage the cluster
- `apt-mark hold` — prevents these packages from being accidentally auto-upgraded

```bash
mkdir -p /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list

apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

systemctl enable kubelet
```
<img width="1166" height="867" alt="Screenshot 2026-04-03 at 7 31 02 PM" src="https://github.com/user-attachments/assets/6ce37648-9c17-4e4c-9f86-3589415de70f" />

---

### Step 7: Verify all services are running

**What & Why:**
- Confirms Docker, cri-dockerd, and kubelet are all active before proceeding
- If any service is not running, fix it before moving to the next part

```bash
systemctl status docker
systemctl status cri-docker.service
systemctl status kubelet
```

<img width="658" height="835" alt="Screenshot 2026-04-03 at 7 32 05 PM" src="https://github.com/user-attachments/assets/14f98416-0477-486b-983d-210b74cde594" />


---

## PART 2 — MASTER NODE ONLY

> ⚠️ Run every command in this section on the **Master Node only**

---

### Step 8: Initialize the Kubernetes Cluster

**What & Why:**
- `kubeadm init` sets up the **control plane** components (API server, scheduler, controller manager, etcd)
- `--cri-socket` tells kubeadm to use Docker (via cri-dockerd) as the runtime
- `--pod-network-cidr` defines the IP range for pods — must match the CNI plugin (Calico uses `192.168.0.0/16`)

```bash
kubeadm init --cri-socket unix:///var/run/cri-dockerd.sock --pod-network-cidr=192.168.0.0/16
```

<img width="1469" height="419" alt="Screenshot 2026-04-03 at 7 33 00 PM" src="https://github.com/user-attachments/assets/c49b0130-57fc-4da1-bd55-fba876c54dc5" />

<img width="851" height="378" alt="Screenshot 2026-04-03 at 7 33 43 PM" src="https://github.com/user-attachments/assets/2e531660-b95a-45ed-9ac0-e929135cd745" />

---

### Step 9: Configure kubectl access

**What & Why:**
- By default, `kubectl` needs a config file to know which cluster to talk to
- These commands copy the admin config to your home directory so `kubectl` works without extra flags

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

<img width="469" height="52" alt="Screenshot 2026-04-03 at 7 34 53 PM" src="https://github.com/user-attachments/assets/0dc249f7-8031-41a8-a424-8077a8f67ac5" />

---

### Step 10: Install Calico Network Plugin (CNI)

**What & Why:**
- By default, pods cannot communicate with each other across nodes
- **Calico** is a CNI (Container Network Interface) plugin that sets up the pod network
- Without a CNI plugin, nodes will stay in `NotReady` state

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/calico.yaml
```

<img width="955" height="514" alt="Screenshot 2026-04-03 at 7 35 28 PM" src="https://github.com/user-attachments/assets/8c6229f7-240e-4353-8a39-ff27c886e86b" />

---

### Step 11: Generate Worker Node Join Command

**What & Why:**
- This generates a unique token + command that worker nodes use to securely join the cluster
- The token expires after 24 hours; regenerate with the same command if needed

```bash
kubeadm token create --print-join-command
```

> 📋 Copy the output — it will look like:
> ```
> kubeadm join 172.31.31.151:6443 --token xxxxx --discovery-token-ca-cert-hash sha256:xxxxx
> ```

<img width="923" height="323" alt="Screenshot 2026-04-03 at 7 39 28 PM" src="https://github.com/user-attachments/assets/6ebde9d1-4643-4c92-b1f9-f6da648e8803" />


<img width="1466" height="466" alt="Screenshot 2026-04-03 at 7 37 30 PM" src="https://github.com/user-attachments/assets/1f1c8ebb-9724-4c3e-83fd-2214d83137f2" />


---

## PART 3 — WORKER NODES ONLY (Worker 1 & Worker 2)

> ⚠️ Run on **both worker nodes**

---

### Step 12: Join the Cluster

**What & Why:**
- This command connects the worker node to the master and registers it in the cluster
- `--cri-socket` flag is required because both `containerd` and `cri-dockerd` are present; we must specify Docker explicitly
- Without this flag you will get: `found multiple CRI endpoints on the host` error

```bash
kubeadm join <MASTER_IP>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --cri-socket unix:///var/run/cri-dockerd.sock
```


---

## PART 4 — MASTER NODE — Verification

---

### Step 13: Check all nodes joined successfully

**What & Why:**
- Confirms all 3 nodes are visible and in `Ready` state
- It may take 1–2 minutes for nodes to become `Ready` while Calico pods start up

```bash
kubectl get nodes
```

Expected output:
```
NAME        STATUS   ROLES           AGE   VERSION
master      Ready    control-plane   5m    v1.32.x
worker-1    Ready    <none>          2m    v1.32.x
worker-2    Ready    <none>          2m    v1.32.x
```

<img width="789" height="389" alt="Screenshot 2026-04-03 at 7 40 27 PM" src="https://github.com/user-attachments/assets/756f92bf-064e-43bb-a8f3-1d01986a33f5" />

---

### Step 14: Check all system pods are running

```bash
kubectl get pods -n kube-system
```

---

## PART 5 — Deploy Nginx Application

---

### Step 15: Create Nginx Deployment

**What & Why:**
- `kubectl create deployment` creates a **Deployment** object in Kubernetes
- Kubernetes will automatically pull the `nginx` image and schedule a pod on one of the worker nodes

```bash
kubectl create deployment nginx --image=nginx
```

<img width="977" height="95" alt="Screenshot 2026-04-03 at 7 42 11 PM" src="https://github.com/user-attachments/assets/a51330bc-96a3-4ed9-917b-359d170a1112" />


---

### Step 16: Verify Deployment and Pod

```bash
kubectl get deployments
kubectl get pods
kubectl get pods -o wide
```

> `-o wide` shows which worker node the pod is running on

<img width="892" height="690" alt="Screenshot 2026-04-03 at 7 43 02 PM" src="https://github.com/user-attachments/assets/b9caeb8d-6cd6-4e15-87fb-c213c86db533" />

---

### Step 17: Expose Nginx as NodePort Service

**What & Why:**
- By default, pods are not accessible from outside the cluster
- `kubectl expose` creates a **Service** that maps a port on every node to the pod's port 80
- `NodePort` means the app is accessible via `<WorkerNodeIP>:<NodePort>` from the internet

```bash
kubectl expose deployment nginx --port=80 --type=NodePort
```
<img width="761" height="45" alt="Screenshot 2026-04-03 at 7 43 33 PM" src="https://github.com/user-attachments/assets/d7827983-7563-4844-ac68-1433939dbd55" />


---

### Step 18: Get the assigned NodePort

```bash
kubectl get svc nginx
```

Expected:
```
NAME    TYPE       CLUSTER-IP      PORT(S)        AGE
nginx   NodePort   10.96.xxx.xxx   80:32456/TCP   10s
```

> Note the port after `80:` — e.g. `32456`. Use this to access the app.

<img width="642" height="83" alt="Screenshot 2026-04-03 at 7 44 10 PM" src="https://github.com/user-attachments/assets/61fa3aef-8f11-45e9-89a5-d54559249ab8" />


---

### Step 19: Access Nginx in Browser

```
http://<WORKER_NODE_PUBLIC_IP>:<NodePort>
```

Example:
```
http://3.109.xx.xx:32456
```

> You should see the **"Welcome to nginx!"** page

<img width="1457" height="565" alt="Screenshot 2026-04-03 at 6 07 39 PM" src="https://github.com/user-attachments/assets/cbc2ea5e-89d4-4f83-8c5f-70c1002e0f0a" />

<img width="1457" height="545" alt="Screenshot 2026-04-03 at 6 08 05 PM" src="https://github.com/user-attachments/assets/0e3b7354-d5cf-4a8b-8b9c-83c1c7b60a5c" />


---

### Step 20: Scale the Deployment

**What & Why:**
- Kubernetes makes it easy to scale applications up or down with a single command
- This spreads the pods across worker nodes for load balancing

```bash
# Scale up to 3 replicas
kubectl scale deployment nginx --replicas=3

# Check pods spread across workers
kubectl get pods -o wide
```

---

## Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Malformed entry in kubernetes.list` | Line break in echo command | Write the `deb` line as a single line with no `\` break |
| `multiple CRI endpoints found` | Both containerd & cri-dockerd running | Add `--cri-socket unix:///var/run/cri-dockerd.sock` to join command |
| `kubelet.service does not exist` | kubelet not installed yet | Run `apt-get install -y kubelet` first |
| Node stuck in `NotReady` | Calico pods still starting | Wait 1–2 min; check with `kubectl get pods -n kube-system` |
| Cannot access Nginx in browser | Security group blocking port | Allow NodePort range `30000-32767` in AWS Security Group inbound rules |

---

## AWS Security Group Rules

| Port | Protocol | Purpose |
|------|----------|---------|
| 6443 | TCP | Kubernetes API Server |
| 10250 | TCP | kubelet API |
| 10251 | TCP | kube-scheduler |
| 10252 | TCP | kube-controller-manager |
| 2379-2380 | TCP | etcd (master only) |
| 30000-32767 | TCP | NodePort services (worker nodes) |
| All traffic | All | Within cluster private subnet |


## 👨‍💻 Author

**Pravesh Kumar**
📬 [LinkedIn](https://www.linkedin.com/in/pravesh22) · [GitHub](https://github.com/pravesh2201)
