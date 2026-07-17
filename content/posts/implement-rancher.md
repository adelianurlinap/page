---
title: "Implementing Rancher to Manage Kubernetes Clusters"
date: 2026-07-15
draft: false
summary: "A hands-on write-up from my internship: building and managing multiple RKE2 Kubernetes clusters from a single pane of glass with Rancher, running on VMware vSphere—complete with HA load balancing, autoscaling, and network & storage benchmarks. Includes a full step-by-step setup guide."
tags: ["kubernetes", "rancher", "RKE2",]
---

During my internship in 2024, I worked on a project to build a Kubernetes cluster for a client (I'll call them PT XYZ). The goal was simple to state but interesting to pull off: move applications that used to run on bare metal into a more *scalable* environment, and manage the whole thing from one place using **Rancher**.

This post is part story, part tutorial. The first half walks through *what* was built and *why*; the second half is a step-by-step setup guide with the actual commands so you can follow along.

## Why Rancher?

Kubernetes is the de facto standard for container orchestration—the second-largest open-source project after Linux, and used by roughly 71% of the world's 100 largest companies. The catch is that Kubernetes is a complex distributed system. Managing a single cluster already takes effort; managing several at once is harder still.

That's where Rancher comes in as a **Kubernetes Management Platform (KMP)**. Rancher lets you:

- Manage many Kubernetes clusters from **one centralized UI**
- Provision clusters automatically onto various cloud providers
- Monitor cluster health, resources, and workloads through a friendly UI
- Apply access control and security policies globally

### How Rancher works

Rancher acts as a *proxy* to communicate with the clusters it manages. A user request flows through the *Authentication Proxy* → Rancher *API Server* → *Cluster Controller*, which watches the cluster via a *Cluster Agent*. There are two key agents: `cattle-cluster-agent` (talks to the Kubernetes API server) and `cattle-node-agent` (per-node operations like upgrades and snapshots).

![Diagram of how Rancher manages Kubernetes clusters](/images/implement-rancher/cara-kerja-rancher.png)

## The Stack

The chosen Kubernetes distribution is **RKE2** (also known as RKE Government)—a distribution built with security in mind. RKE2 is FIPS 140-2 compliant, regularly scanned with Trivy, and ships with defaults that pass the CIS Kubernetes benchmark. Unlike the older RKE, which still used Docker, RKE2 uses the **containerd** runtime that has become the Kubernetes standard.

For the virtualization layer, the clusters run on **VMware vSphere** (ESXi as the hypervisor, vCenter for management). Rancher talks to the vCenter API to provision VMs automatically, then installs Kubernetes on top of them.

Component versions used in this project:

| Component | Name | Version |
|---|---|---|
| Operating System | Ubuntu | 22.04 |
| Container Runtime | containerd | 1.7 |
| Container Orchestrator | RKE2 Stable | 1.27 |
| Ingress | Ingress Nginx | 1.9 |
| Load Balancer | HAProxy | 2.4 |
| VRRP | Keepalived | 2.2 |

## Architecture (High-Level Design)

There are **nine nodes** in total, all running on VMware. Rancher Management is deployed in a **High Availability (HA)** configuration across three nodes, with a load balancer (HAProxy + Keepalived) distributing traffic to both the Rancher servers and the Kubernetes API server. The end result is a production cluster with **3 master nodes** and **3 worker nodes**.

![High-Level Design of the implementation](/images/implement-rancher/high-level-design.png)

Node roles at a glance:

| Role | Count | Notes |
|---|---|---|
| Load Balancer + Rancher Management | 3 | HA with a shared VIP |
| Master (etcd, control plane) | 3 | Production cluster |
| Worker | 3 | Production cluster |

The network layout used in the guide below:

| Node | IP | VIP |
|---|---|---|
| rancher-mgmt-1 | 10.1.92.21 | 10.1.92.20 |
| rancher-mgmt-2 | 10.1.92.22 | (shared) |
| rancher-mgmt-3 | 10.1.92.23 | (shared) |

> **A note on credentials:** the commands below reproduce the example passwords, tokens, and keys from the project (`P@ssw0rd`, `admin12345`, etc.). Treat them as placeholders and replace them with your own secrets before running anything in a real environment.

---

# Step-by-Step Setup Guide

The build happens in layers, from the bottom up: first the HA load-balancing layer on the three management nodes, then the RKE2 cluster that hosts Rancher, then Rancher itself, and finally the production cluster provisioned onto vSphere—plus storage and a sample workload.

## Step 1 — HA Load Balancing (Keepalived + HAProxy)

The three management nodes share a single Virtual IP (`10.1.92.20`). Keepalived (VRRP) decides which node holds the VIP, and HAProxy load-balances traffic across the Rancher/RKE2 backends.

### 1a. Enable non-local bind (all three nodes)

HAProxy needs to bind to the VIP even when it isn't currently assigned to the node:

```bash
echo "net.ipv4.ip_nonlocal_bind=1" | sudo tee /etc/sysctl.d/ip_nonlocal_bind.conf
sudo sysctl -p
sudo sysctl --system
```

### 1b. Install & configure Keepalived

Install on each node, then drop in a config. The only differences between nodes are `state` and `priority` — the highest priority becomes the active VIP holder.

**rancher-mgmt-1** (`state MASTER`, `priority 101`):

```bash
sudo apt install keepalived
cat <<EOF | sudo tee -a /etc/keepalived/keepalived.conf
vrrp_script chk_haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
}
vrrp_instance haproxy-vip {
    state MASTER
    priority 101
    interface ens192            # network card
    virtual_router_id 50
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass P@ssw0rd
    }
    virtual_ipaddress {
        10.1.92.20/32           # rancher-cluster VIP
    }
    track_script {
        chk_haproxy
    }
}
EOF
```

**rancher-mgmt-2** — same config, but `state BACKUP` and `priority 100`.
**rancher-mgmt-3** — same config, but `state BACKUP` and `priority 99`.

### 1c. Enable & start Keepalived (all three nodes)

```bash
sudo systemctl restart keepalived
sudo systemctl enable keepalived
sudo systemctl status keepalived
```

### 1d. Install & configure HAProxy (all three nodes)

HAProxy fronts two backends: the RKE2 supervisor/registration port (`9345`) and the Kubernetes API server (`6443`).

```bash
sudo apt install haproxy
cat <<EOF | sudo tee -a /etc/haproxy/haproxy.cfg
### Rancher Cluster ###
frontend rancher-rke2-server
    bind 10.1.92.20:10345
    mode tcp
    option tcplog
    default_backend rancher-rke2-server

frontend rancher-rke2-apiserver
    bind 10.1.92.20:10443
    mode tcp
    option tcplog
    default_backend rancher-rke2-apiserver

backend rancher-rke2-server
    mode tcp
    option tcp-check
    balance roundrobin
    default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
    server rancher-rke2-server-1 10.1.92.21:9345 check
    server rancher-rke2-server-2 10.1.92.22:9345 check
    server rancher-rke2-server-3 10.1.92.23:9345 check

backend rancher-rke2-apiserver
    mode tcp
    option tcp-check
    balance roundrobin
    default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
    server rancher-rke2-apiserver-1 10.1.92.21:6443 check
    server rancher-rke2-apiserver-2 10.1.92.22:6443 check
    server rancher-rke2-apiserver-3 10.1.92.23:6443 check
EOF
```

Validate the config, then enable and start:

```bash
haproxy -c -V -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

## Step 2 — Bootstrap the RKE2 Cluster for Rancher

This is a three-node, all-in-one RKE2 cluster (each node is both master and worker) that Rancher itself will run on.

### 2a. Install RKE2 (all three nodes)

```bash
curl -sfL https://get.rke2.io --output install.sh
chmod +x install.sh
INSTALL_RKE2_TYPE=server ./install.sh
```

### 2b. Create the config file

On the **first** node, create `/etc/rancher/rke2/config.yaml`:

```bash
sudo mkdir -p /etc/rancher/rke2
sudo nano /etc/rancher/rke2/config.yaml
```

```yaml
write-kubeconfig-mode: "0644"
token: P@ssw0rd
tls-san:
  - rancher-rke2.xyz.co.id
  - rancher-mgmt-1.xyz.co.id
  - rancher-mgmt-2.xyz.co.id
  - rancher-mgmt-3.xyz.co.id
```

### 2c. Enable & start RKE2

```bash
systemctl enable rke2-server
systemctl start rke2-server
```

### 2d. Export the kubeconfig and PATH

```bash
cat <<EOF >> ~/.bashrc
export PATH=$PATH:/var/lib/rancher/rke2/bin
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
EOF
source ~/.bashrc
```

### 2e. Join the other two nodes

On **rancher-mgmt-2** and **rancher-mgmt-3**, point the config at the first server (through the VIP), then restart:

```bash
sudo nano /etc/rancher/rke2/config.yaml
```

```yaml
server: https://rancher-rke2.xyz.co.id:10345
# ...plus the same token / tls-san as node 1
```

```bash
systemctl restart rke2-server
```

Finally, make sure the kubeconfig on every node points at the VIP so `kubectl` talks to the load-balanced API endpoint:

```yaml
# /etc/rancher/rke2/rke2.yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <--- content omitted --->
    server: https://rancher-rke2.xyz.co.id:10443
  name: default
```

## Step 3 — Install Rancher Management

Rancher installs onto the RKE2 cluster via Helm, with cert-manager handling TLS.

### 3a. Install Helm v3

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 3b. Add the Helm repos and cert-manager CRDs

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
kubectl create namespace cattle-system
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.crds.yaml
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

### 3c. Install cert-manager

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.11.0
```

### 3d. Deploy Rancher (3 replicas for HA)

```bash
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher-rke2.xyz.co.id \
  --set replicas=3 \
  --set bootstrapPassword=admin12345
```

Once this finishes, Rancher is reachable at the hostname you set, and you get the familiar dashboard:

![Rancher dashboard](/images/implement-rancher/rancher-dashboard.png)

## Step 4 — Provision the Production Cluster on VMware vSphere

This part is done through the Rancher UI, which drives the vCenter API to create the VMs and install Kubernetes on them.

**Create the cluster:**

1. Go to **Cluster Management → Create**
2. Choose **VMware vSphere**
3. Pick the cloud credential (e.g. `vCenter Prod`)
4. Fill in the form; set **Name:** `production-rke2`

![VMware vSphere integration in Rancher](/images/implement-rancher/integrasi-vsphere.png)

**Cloud-init template** (used for both node pools):

```yaml
#cloud-config
users:
  - name: root
    ssh_authorized_keys:
      - ssh-rsa <output omitted> root@rancher-mgmt-1
  - name: ubuntu
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    shell: /bin/bash
    groups: users
ssh_pwauth: true
disable_root: false
chpasswd:
  list: |
    root:P@ssw0rd
    ubuntu:P@ssw0rd
  expire: false
```

**Master pool** (3 machines): 4 CPUs, 16 GB RAM, 100 GB disk, deployed from template `ubuntu22-kube-new`.
**Worker pool** (3 machines): 32 CPUs, 64 GB RAM, 150 GB disk, same template.

**Cluster configuration** (etcd + advanced tabs):

- Automatic etcd snapshots: **Enabled**, cron `0 */5 * * *`, keep last **5**
- Metrics: exposed to the public interface
- Additional Controller Manager args: `bind-address:0.0.0.0`
- Additional Scheduler args: `bind-address:0.0.0.0`

Rancher then provisions everything automatically—from VM creation to a running Kubernetes cluster:

![Production cluster running on the vSphere cloud provider](/images/implement-rancher/cluster-cloud-provider.png)

The result is `production-rke2` with 6 nodes on Kubernetes `v1.27.12+rke2r1`, all visible from the cluster dashboard:

![Production cluster dashboard](/images/implement-rancher/cluster-dashboard.png)

![Per-node detail](/images/implement-rancher/informasi-simpul.png)

## Step 5 — NFS Storage Class

Since data written inside a pod is lost when the pod restarts, we back persistent volumes with an **NFS provisioner** so data survives across pod restarts and is shared across nodes.

**Install the NFS client on every worker:**

```bash
sudo apt install nfs-common
```

**Deploy the NFS Subdir External Provisioner:**

```bash
kubectl create ns nfs-provisioner
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
nano values.yml
```

```yaml
nfs:
  path: /volume1/rancher-prod
  server: 10.1.8.73
storageClass:
  accessModes: ReadWriteMany
  name: nfs-storageclass
  provisionerName: nfs-cluster-provisioner
  archiveOnDelete: "false"
  onDelete: 'delete'
```

```bash
helm install --namespace nfs-provisioner nfs-cluster-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --values values.yml
```

**Make it the default storage class:**

```bash
kubectl patch storageclass nfs-storageclass -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl get sc
```

## Step 6 — Deploy a Workload with a Resource Quota

To exercise the cluster, deploy Nginx into its own namespace with a resource quota.

**Create a Resource Quota** (`nginx-rq` namespace):

```bash
export KUBECONFIG=<path-to-kubeconfig>
nano resourcequota.yaml
```

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu-rq
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```

```bash
kubectl apply -f resourcequota.yaml -n nginx-rq
```

**Deploy Nginx** (from the Rancher UI: Deployments → Create → Edit as YAML):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: nginx-rq
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - image: nginx
          name: nginx
          resources:
            requests:
              cpu: "50m"
              memory: "200Mi"
            limits:
              cpu: "100m"
              memory: "500Mi"
```

---

# Results & Testing

With the platform in place, here's what it does in practice.

## Autoscaling with HPA

On top of the Nginx deployment, a **Horizontal Pod Autoscaler (HPA)** was configured with a minimum of 1 replica, a maximum of 5, targeting 50% CPU and 100Mi memory.

![HPA metrics](/images/implement-rancher/hpa-metrics.png)

Under a stress test (`stress --cpu 4 --timeout 120`), CPU load climbed and the HPA automatically scaled the pods up. When the load dropped, it scaled them back down—resources used efficiently, on demand.

![Pods scaling up under stress test](/images/implement-rancher/pod-scale-up.png)

## High Availability & Failover

To test resilience, one master node was deliberately shut down (`shutdown -h now`). Even with a master down, cluster operations stayed healthy and all running pods kept serving—thanks to the HA control plane, where multiple masters can take over for one another.

![Pods keep running even with one master down](/images/implement-rancher/failover-pod.png)

## Performance Benchmarks

**Network** (measured with `iperf3`):

| Path | Transfer | Bandwidth |
|---|---|---|
| Master → Worker | 10.5 GBytes | 8.97 Gbits/sec |
| Worker → Worker | 13.6 GBytes | 11.6 Gbits/sec |

1. Master → Worker

![Network performance test between nodes](/images/implement-rancher/uji-jaringan.png)

2. Worker → Worker

![Network performance test between nodes 2](/images/implement-rancher/uji-jaringan-2.png)

**Storage** (measured with `FIO`, in IOPS/sec):

| Location | Random Read | Random Write |
|---|---|---|
| Rancher VM root disk (`/root`) | 89,651 | 41,722 |
| Pod on NFS storage class | 21,816 | 15,872 |

1. `/root` IOPS/s

-	Random Read/Write: 82721 / 31037
-	Random Read: 89651
-	Random Write: 41722

![Root IOPS](/images/implement-rancher/root-fio.png)

2. `NFS` IOPS/s

-	Random Read/Write: 14053 / 4654
-	Random Read: 21816
-	Random Write: 15872

![NFS IOPS](/images/implement-rancher/pod-fio.png)

Local disk clearly outperforms NFS on IOPS—an expected trade-off for network-backed storage—but NFS buys you data availability across pods and nodes.

## Conclusion

A few things this implementation proved out:

1. **Rancher successfully manages multiple Kubernetes clusters from one central pane**, improving operational coordination and control.
2. **Rancher's features held up in practice**: cloud-provider integration, the Horizontal Pod Autoscaler, and per-namespace Resource Limit & Quota Management.
3. **Automated provisioning on VMware vSphere worked smoothly**—Rancher handled the whole flow from VM creation to a ready Kubernetes cluster, needing only the cloud credentials.
4. **Network and storage performance were validated** through iperf3 and FIO testing.

Overall, deploying Rancher helped PT XYZ hit its goals: better scalability, more efficient use of IT infrastructure, and reliable application availability on a Rancher-managed Kubernetes platform.

---

*This post summarizes my internship report from the Informatics Engineering program, Faculty of Engineering, Universitas Pancasila (2024). Example credentials and internal IPs are shown for illustration—replace them with your own before using any of this in production.*

## References

1. kubernetes.io, "Production-Grade Container Orchestration." <https://kubernetes.io/>
2. CNCF, "Kubernetes Project Journey Report," 2023. <https://www.cncf.io/reports/kubernetes-project-journey-report/>
3. A. Valero, "How To Simplify Your Kubernetes Adoption Using Rancher," SUSE, 2024. <https://www.suse.com/c/rancher_blog/how-to-simplify-your-kubernetes-adoption-using-rancher/>
4. SUSE Rancher, "Rancher Server and Components." <https://ranchermanager.docs.rancher.com/v2.6/reference-guides/rancher-manager-architecture/rancher-server-and-components>
5. SUSE Rancher, "Introduction to RKE2." <https://docs.rke2.io/>
6. SUSE Rancher, "Overview of RKE." <https://rke.docs.rancher.com/>
7. N. Poulton and P. Joglekar, *The Kubernetes Book*, 2022 Edition. Leanpub, 2022.
8. VMware, "VMware vSphere Documentation." <https://docs.vmware.com/en/VMware-vSphere/index.html>
