# 🏗️ Real-World Kubernetes Implementation: High Availability in On-Prem & VPS Environments

> A practical guide to setting up production-grade Kubernetes clusters, with tooling and architectural decisions based on real-world experience.

## 🧭 Introduction

This post walks through **real-world Kubernetes setups** both on-premises and in VPS environments, focusing on:
- High availability
- Storage
- Load balancing
- Disaster recovery
- Monitoring
- Tools like `kubeadm`, `Kubespray`, `Rancher`, `K3s`, `KubeVirt`, `Knative`, `Velero`, `KEDA`, and more.

## 📦 Tools & Methods for Cluster Setup

| Tool       | Description | Best Use |
|------------|-------------|----------|
| `kubeadm`  | Vanilla K8s installer | Full control, HA with manual setup |
| `Kubespray`| Ansible-based multi-node deployment | Automated HA setup |
| `Rancher`  | UI-driven multi-cluster management | Ease of use & enterprise-friendly |
| `K3s`      | Lightweight Kubernetes | Ideal for VPS or edge |
| `RKE2`     | Hardened K3s version | Production-grade light clusters |

## 🏢 On-Premises Kubernetes Architecture (Production-Grade)

### 1. Load Balancing & External Access

**Standard Approach:**

- Hardware Load Balancer (F5, NetScaler, Kemp) → MetalLB (BGP mode) → Ingress → Services

**Alternative:**

- MetalLB + ExternalDNS + reserved static IPs

### 2. Storage

**Enterprise-Grade CSI Drivers:**
- NetApp Trident
- Dell EMC CSI
- Pure Storage CSI

**Alternative Software-Defined Storage:**
- [`Rook-Ceph`](https://rook.io/)
- [`Longhorn`](https://longhorn.io/)

### 3. Networking

**Recommended CNIs:**
- [`Calico`](https://projectcalico.docs.tigera.io/) with BGP for L3 integration
- Integration with SDN (Cisco ACI, NSX-T)

## 🌐 VPS-Based Kubernetes Architecture (Production-Grade)

### 1. External Access

**Practical Setup:**
```
[Public IP:80/443]
      ↓
[Nginx Reverse Proxy on VPS Host]
      ↓
[NodePort Services (e.g., 30080)]
```

**Example Nginx Config:**
```nginx
server {
    listen 80;
    server_name app.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name app.example.com;

    ssl_certificate /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;

    location / {
        proxy_pass http://localhost:30080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 2. Multi-Node Network

Use **WireGuard** if private networking isn't available:

```ini
[Interface]
PrivateKey = <private-key>
Address = 10.0.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <peer-key>
AllowedIPs = 10.0.0.2/32
Endpoint = <peer-public-ip>:51820
```

### 3. Storage

- **Simple:** NFS Provisioner
- **Resilient:** [`Longhorn`](https://longhorn.io/)

## 💾 Best Production-Grade Storage Solutions

### 1. 🔁 Longhorn (All-Around Best)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  numberOfReplicas: "3"
  fsType: ext4
```

### 2. 🐘 Rook-Ceph (Enterprise-Scale)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  pool: replicapool
  clusterID: rook-ceph
  csi.storage.k8s.io/fstype: ext4
allowVolumeExpansion: true
reclaimPolicy: Delete
```

### 3. 🧱 OpenEBS (Flexible)

- Jiva, cStor, LocalPV support
- Easy to configure
- Great for VPS/edge nodes

## 🔒 Certificate Management

- **Let's Encrypt** + [`cert-manager`](https://cert-manager.io/)
- Enterprise PKI for on-prem

## 📈 Monitoring & Logging

| Need | Tool |
|------|------|
| Metrics | [`Prometheus`](https://prometheus.io/), [`Grafana`](https://grafana.com/) |
| Logs | [`Loki`](https://grafana.com/oss/loki/), EFK (Elasticsearch + Fluentd + Kibana) |

## 🧩 Backup & Disaster Recovery

| Type | Tool |
|------|------|
| Cluster/app backups | [`Velero`](https://velero.io/) |
| etcd snapshots | Scheduled cronjobs off-cluster |

## ⚙️ Kubernetes High Availability Practices

| Environment | Strategy |
|-------------|----------|
| On-Prem | 3+ control plane nodes, external etcd, spread across physical hardware |
| VPS | Use different provider regions, WireGuard mesh, control plane node anti-affinity |

## 🚀 Advanced Components in Production

| Component | Purpose |
|----------|---------|
| [`Knative`](https://knative.dev/) | Serverless deployments |
| [`KEDA`](https://keda.sh/) | Event-driven autoscaling |
| [`KubeVirt`](https://kubevirt.io/) | Run VMs inside Kubernetes |
| [`Node Problem Detector`](https://github.com/kubernetes/node-problem-detector) | Monitor & report node issues |
| [`HPA`](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) | Horizontal Pod Autoscaler |
| [`CRDs`](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) | Extend K8s APIs |
| [`cert-manager`](https://cert-manager.io/) | Automatic TLS certificate lifecycle |

## ✅ Final Thoughts

In real-world scenarios, success comes from aligning tools and configurations to the **environment's constraints**, not from chasing ideal architectures. Kubernetes gives you flexibility—your job is to balance **simplicity, resilience, and maintainability**.

## 📚 References

- [Kubernetes Official Docs](https://kubernetes.io/docs/)
- [Kubespray](https://github.com/kubernetes-sigs/kubespray)
- [Rancher](https://rancher.com/)
- [Longhorn](https://longhorn.io/)
- [Rook-Ceph](https://rook.io/)
- [K3s](https://k3s.io/)
- [cert-manager](https://cert-manager.io/)
- [Velero](https://velero.io/)
- [KEDA](https://keda.sh/)
- [Knative](https://knative.dev/)
- [KubeVirt](https://kubevirt.io/)
