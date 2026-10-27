# Provisioning an NKP Management Cluster and Deploying Workload Clusters

This tutorial shows how to stand up a self-managed **Nutanix Kubernetes Platform (NKP)** management cluster and then create managed workload clusters from it, using both the CLI and the Kommander UI.

## Architecture Overview

NKP uses a hub-and-spoke model:

- **Management cluster** (self-managed): runs Cluster API (CAPI) controllers + Kommander UI/platform services.
- **Workload clusters** (managed): created and lifecycle-managed by the management cluster.

```
┌─────────────────────────────┐
│   Management Cluster        │
│  (self-managed)             │
│  • CAPI controllers         │
│  • Kommander / UI           │
│  • Platform apps            │
└──────────────┬──────────────┘
               │ creates & manages
       ┌───────┴───────┐
       ▼               ▼
┌─────────────┐ ┌─────────────┐
│ Workload    │ │ Workload    │
│ Cluster 1   │ │ Cluster 2   │
└─────────────┘ └─────────────┘
```

## Prerequisites

- NKP CLI and supporting binaries (download from Nutanix Support Portal; requires appropriate license for non-AHV)
- Infrastructure provider:
  - **AHV**: Prism Central reachable, Nutanix CSI storage container ready
  - **vSphere**: vCenter 7.0.3+, VM template prepared
- Jump/bastion host with Docker, kubectl, and an SSH key pair
- Network planning:
  - One free IP for the control-plane endpoint (VIP)
  - MetalLB IP range for LoadBalancer services (must not overlap pod/service CIDRs)
- Base OS image (Nutanix-provided Rocky Linux image for AHV, or custom template for vSphere)

## 1. Create the Self-Managed Management Cluster (CLI)

The management cluster is created with the `nkp` CLI using the `--self-managed` flag. This is the standard and recommended way to bootstrap the control plane.

### Set environment variables (example for AHV)

```bash
export CLUSTER_NAME="nkp-mgmt"
export NUTANIX_ENDPOINT="https://prism-central.example.com:9440"
export NUTANIX_CLUSTER="your-pe-cluster"
export NUTANIX_SUBNET="your-subnet"
export CONTROL_PLANE_VIP="10.0.1.100"
export LB_IP_RANGE="10.0.1.101-10.0.1.110"
export VM_IMAGE="nkp-rocky-linux-image-name"
export STORAGE_CONTAINER="your-storage-container"
export SSH_PUBLIC_KEY="~/.ssh/id_rsa.pub"
```

### Create the cluster

```bash
nkp create cluster nutanix \
  --cluster-name "${CLUSTER_NAME}" \
  --endpoint "${NUTANIX_ENDPOINT}" \
  --control-plane-endpoint-ip "${CONTROL_PLANE_VIP}" \
  --control-plane-prism-element-cluster "${NUTANIX_CLUSTER}" \
  --control-plane-subnets "${NUTANIX_SUBNET}" \
  --control-plane-replicas 3 \
  --control-plane-vm-image "${VM_IMAGE}" \
  --worker-prism-element-cluster "${NUTANIX_CLUSTER}" \
  --worker-subnets "${NUTANIX_SUBNET}" \
  --worker-replicas 3 \
  --worker-vm-image "${VM_IMAGE}" \
  --kubernetes-service-load-balancer-ip-range "${LB_IP_RANGE}" \
  --csi-storage-container "${STORAGE_CONTAINER}" \
  --ssh-public-key-file "${SSH_PUBLIC_KEY}" \
  --self-managed
```

What happens:
1. A temporary bootstrap cluster (KinD) is created locally.
2. CAPI controllers are installed into the bootstrap cluster.
3. The real management cluster is provisioned.
4. Control is pivoted so the new cluster becomes self-managed (CAPI moves onto it).
5. Kommander and default platform services are installed (on AHV).

### Retrieve kubeconfig and access the UI

```bash
nkp get kubeconfig -c "${CLUSTER_NAME}" > "${CLUSTER_NAME}.conf"
export KUBECONFIG="${CLUSTER_NAME}.conf"
kubectl get nodes
```

Get the dashboard credentials:

```bash
nkp get dashboard --kubeconfig="${CLUSTER_NAME}.conf"
# or open directly
nkp open dashboard --kubeconfig="${CLUSTER_NAME}.conf"
```

The Kommander UI is typically available via the first IP in your MetalLB range (Traefik).

## 2. Verify the Management Cluster

```bash
kubectl get nodes
kubectl get pods -A | grep -E 'capi|kommander|traefik|csi'
```

Log into the NKP UI and confirm the management cluster appears as healthy.

## 3. Deploy Workload Clusters

### Option A – Via CLI

Workload clusters are created **without** `--self-managed` and must target the management cluster’s kubeconfig.

```bash
export WORKLOAD_NAME="workload-01"
export WORKLOAD_VIP="10.0.1.120"
export WORKLOAD_LB_RANGE="10.0.1.121-10.0.1.130"

nkp create cluster nutanix \
  --cluster-name "${WORKLOAD_NAME}" \
  --kubeconfig "${CLUSTER_NAME}.conf" \
  --endpoint "${NUTANIX_ENDPOINT}" \
  --control-plane-endpoint-ip "${WORKLOAD_VIP}" \
  --control-plane-prism-element-cluster "${NUTANIX_CLUSTER}" \
  --control-plane-subnets "${NUTANIX_SUBNET}" \
  --control-plane-replicas 3 \
  --control-plane-vm-image "${VM_IMAGE}" \
  --worker-prism-element-cluster "${NUTANIX_CLUSTER}" \
  --worker-subnets "${NUTANIX_SUBNET}" \
  --worker-replicas 3 \
  --worker-vm-image "${VM_IMAGE}" \
  --kubernetes-service-load-balancer-ip-range "${WORKLOAD_LB_RANGE}" \
  --csi-storage-container "${STORAGE_CONTAINER}" \
  --ssh-public-key-file "${SSH_PUBLIC_KEY}"
```

Monitor progress:

```bash
kubectl --kubeconfig="${CLUSTER_NAME}.conf" get clusters -A
kubectl wait --for=condition=ControlPlaneReady "clusters/${WORKLOAD_NAME}" --timeout=30m
```

### Option B – Via the Kommander UI (Recommended for most users)

Once the management cluster is running and you are logged into the UI:

1. Select the desired **Workspace**.
2. Click **Add Cluster** → **Create Cluster**.
3. Fill in the wizard sections:
   - **Basic info**: Cluster name, labels, infrastructure provider (Nutanix / vSphere / etc.).
   - **Control plane**: Prism/AOS cluster or vSphere details, VM image, endpoint IP, replica count, sizing.
   - **Worker node pools**: Same infrastructure details, replica count, optional autoscaling.
   - **Networking**: Load-balancer IP range (MetalLB), any additional VIPs.
   - **Storage**: Storage container (AHV) or datastore (vSphere).
   - **SSH key**: Public key for node access.
4. Review and click **Create**.

The cluster appears on the Clusters page in “Provisioning” status and moves to “Provisioned” / “Joined” when ready (typically 10–20 minutes).

You can download the workload cluster’s kubeconfig directly from the cluster card’s Actions menu.

### Option C – YAML-driven

Generate manifests with a dry-run and apply them:

```bash
nkp create cluster nutanix ... --dry-run -o yaml > workload.yaml
kubectl apply -f workload.yaml --kubeconfig="${CLUSTER_NAME}.conf"
```

## 4. Day-2 Operations

- Scale: adjust node pool replicas via CLI or UI
- Upgrade: use `nkp upgrade` workflows or the UI
- Delete: `nkp delete cluster --cluster-name <name> --kubeconfig <mgmt-kubeconfig>`
- Platform applications are automatically pushed to managed clusters according to your license tier

## Tips & Common Issues

- Control-plane VIP and MetalLB ranges must be unique per cluster and outside pod/service CIDRs
- On AHV the Nutanix CSI driver is installed by default; on vSphere the vSphere CSI is used
- Ensure the jump host has enough resources (minimum ~4 vCPU / 8 GB for the bootstrap process)
- For air-gapped environments add registry mirror flags and pre-load images
- Infrastructure provider credentials (Prism Central or vCenter) must be configured in the UI before creating clusters from the wizard

## References

- Official NKP documentation (Nutanix portal)
- Cluster API (CAPI) concepts
- Nutanix CSI / storage container requirements
