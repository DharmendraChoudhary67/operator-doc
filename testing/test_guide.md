# Valkey Operator — PVC + TopologySpreadConstraints Test Guide

This guide walks through spinning up a local Kind cluster and testing the Valkey operator
with persistent volumes (PVC) and topology spread constraints (TSC).

## Prerequisites

- [`kind`](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) installed
- [`kubectl`](https://kubernetes.io/docs/tasks/tools/) installed
- Docker running (with multi-arch support if on Apple Silicon)

## Operator Image

The operator image is published to Docker Hub and supports **both `amd64` and `arm64`**
(multi-arch manifest), so it works on both Intel and Apple Silicon machines without any
extra configuration:

```
hunter47dj/valkey-operator:pvc-tsc-test
```

This image reference is already baked into `install_valkey_operator.yaml`, so no manual
image overrides are needed — `kubectl apply` will pull it directly from Docker Hub.

---

## Test Files

| File | Purpose |
|------|---------|
| `kind_cluster.yaml` | Kind cluster definition (`valkey-tsc-pvc-test`, 1 control-plane + 3 worker) |
| `install_valkey_operator.yaml` | Full operator install manifest (CRDs, RBAC, controller Deployment) |
| `valkey_PVC_TSC_test.yaml` | `ValkeyCluster` CR — 3 shards, 1 replica, 2Gi PVC, TSC, resource limits |

---

## Steps

### 1. Prepare working directory

Copy the test manifests to a temp location (keeps `/tmp` clean and avoids long paths):

```bash
mkdir -p /tmp/valkey-test
cp kind_cluster.yaml        /tmp/valkey-test/kind-cluster.yaml
cp valkey_PVC_TSC_test.yaml /tmp/valkey-test/valkeycluster-tsc-pvc.yaml
```

### 2. Create the Kind cluster

```bash
kind create cluster --config /tmp/valkey-test/kind-cluster.yaml
```

This creates a cluster named **`valkey-tsc-pvc-test`** with one control-plane node and
one worker node.

> **Note:** The `ValkeyCluster` spec uses `whenUnsatisfiable: DoNotSchedule` with
> `topologyKey: kubernetes.io/hostname`. With only 2 nodes in this cluster config, the
> TSC maxSkew constraint will be applied across those 2 nodes. For a full 3-node spread
> (one shard primary per distinct hostname), add extra workers to `kind_cluster.yaml`:
> ```yaml
> nodes:
>   - role: control-plane
>   - role: worker
>   - role: worker   # add this
>   - role: worker   # and this
> ```

### 3. Install the operator

```bash
kubectl apply -f /tmp/valkey-test/install_valkey_operator.yaml
```

This creates:
- The `valkey-operator-system` namespace
- `ValkeyCluster` and `ValkeyNode` CRDs
- RBAC (ClusterRole, ClusterRoleBinding, ServiceAccount)
- The controller-manager Deployment pulling `hunter47dj/valkey-operator:pvc-tsc-test`

### 4. Wait for the operator to be Ready

```bash
kubectl -n valkey-operator-system rollout status deploy/valkey-operator-controller-manager
```

Expected output:
```
deployment "valkey-operator-controller-manager" successfully rolled out
```

### 5. Create the ValkeyCluster

```bash
kubectl apply -f /tmp/valkey-test/valkeycluster-tsc-pvc.yaml
```

This creates a `ValkeyCluster` named **`valkey-tsc-pvc-demo`** with:
- 3 shards, 1 replica each (6 pods total)
- 2Gi PVCs per pod with `reclaimPolicy: Retain`
- TSC: `maxSkew: 1` on `kubernetes.io/hostname`, `DoNotSchedule`
- `maxmemory: 100mb`, eviction policy: `allkeys-lfu`
- Resource requests: 128Mi / 50m CPU — limits: 256Mi / 500m CPU

### 6. Watch reconciliation

```bash
kubectl get valkeycluster,valkeynode,pods,pvc -w
```

You should see:
- `ValkeyCluster` state transition from `Pending` → `Forming` → `Running`
- `ValkeyNode` objects created per shard slot
- Pods reaching `Running` state
- PVCs bound with 2Gi each

### 7. Verify TSC spread

```bash
kubectl get pods -L valkey.io/shard,statefulset.kubernetes.io/pod-name -o wide
```

The `NODE` column should show pods from different shards distributed across different
hostnames, confirming the topology spread constraint is enforced.

---

## Cleanup

```bash
kind delete cluster --name valkey-tsc-pvc-test
```

> Because PVCs use `reclaimPolicy: Retain`, deleting the cluster will also delete the
> underlying Kind Docker volumes. This is expected for local testing.
