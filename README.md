# README

Using the [Cluster API reference quickstart](https://cluster-api.sigs.k8s.io/user/quick-start) as an inspiration,
we'll walk through a an abbreviated deployment of both a management cluster,
and and associated *managed* cluster using the Cluster API CLI.

## Overview

This is a Kubernetes Cluster API (CAPI) testing workspace for generating and managing cluster configurations using the `clusterctl` CLI and Docker as the infrastructure provider.

The idea is to bootstrap a management control plane via kind,
then build a manifest that can be used to provision a cluster named `dev1`
using the Docker infrastructure provider.

Versions used:

- CAPI: v1.12.4
- K8s: v1.35.0

## End-to-End Setup: Full Reproducible Workflow

### Step 0 — Download clusterctl

Download the clusterctl binary for your platform directly from the CAPI GitHub release,
rename it to `clusterctl` in the current directory, and set execute permissions:

```bash
# macOS ARM64 (Apple Silicon)
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.12.4/clusterctl-darwin-arm64 \
  -o clusterctl
chmod +x clusterctl
./clusterctl version
```

For other platforms, replace `clusterctl-darwin-arm64` with the appropriate asset:

| Platform | Asset name |
|---|---|
| macOS ARM64 | `clusterctl-darwin-arm64` |
| macOS x86_64 | `clusterctl-darwin-amd64` |
| Linux x86_64 | `clusterctl-linux-amd64` |
| Linux ARM64 | `clusterctl-linux-arm64` |

### Step 1 — Create the kind management cluster

The kind cluster must mount the host Docker socket so the CAPD controller can create
Docker containers as cluster nodes. Use `kind-config.yaml` for this:

```bash
kind create cluster --name capi-mgmt --config kind-config.yaml
```

`kind-config.yaml` is checked into this repo and contains:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraMounts:
  - hostPath: /var/run/docker.sock
    containerPath: /var/run/docker.sock
```

**Why the `extraMounts` is required:** On macOS, Docker runs in a VM. The CAPD controller
runs as a pod inside kind and needs to call the Docker API to create containers as cluster
nodes. Without this mount, `/var/run/docker.sock` does not exist inside the kind node and
all DockerCluster/DockerMachine reconciliation fails.

### Step 2 — Initialize CAPI providers

The `ClusterTopology` feature gate must be enabled at init time. The `--flavor development`
template (used in Step 4) uses ClusterClass, which requires this feature:

```bash
source .env
CLUSTER_TOPOLOGY=true ./clusterctl init --infrastructure docker
```

Verify all four providers are installed:

```bash
kubectl get providers -A
```

Expected output shows `cluster-api`, `kubeadm` (bootstrap + control-plane), and `docker`
all at `v1.12.4`.

### Step 3 — Generate the dev1 cluster manifest

The Docker provider ships its template as `cluster-template-development.yaml` in the
GitHub release (not `cluster-template.yaml`). The `--flavor development` flag is required
to resolve it; without it, clusterctl returns a 404 error.

```bash
source .env
./clusterctl generate cluster dev1 \
  --kubernetes-version ${KUBE_VERSION} \
  --control-plane-machine-count 1 \
  --worker-machine-count 1 \
  --infrastructure ${INFRA_PROVIDER}:${INFRA_VERSION} \
  --flavor development > dev1-cluster.yaml
```

*Note: the `flavor` flag is necessary to explicitly set to `development`,
otherwise capi cli will attempt to resolve provider from the wrong infrastructure provider manifest
from the associated provider repository.*

### Step 4 - Review the manifest

Take a look at the generated manifest,
and the resources declared in it:

- `ClusterClass` -> see the configuration specifing `ClusterTemplates`, `MachineTemplates`, `MachinePools` and `KubeadmConfigTemplates`, admission control and pod security policies
- `DockerClusterTemplate`
- `KubeadmControlPlaneTemplate`
- `DockerMachineTemplate` -> 2 of them, one for control-plane, the other for the worker
- `DockerMachinePoolTemplate` -> note that `MachinePool` is an experimental (beta) feature as of v1.12.4,
- `KubeadmConfigTemplate`
- `Cluster`

### Step 5 — Apply the manifest

```bash
kubectl apply -f dev1-cluster.yaml
```

Watch provisioning progress (run from a separate terminal window):

```bash
watch ./clusterctl describe cluster dev1
```

Both machines (control-plane + worker) will appear as Docker containers. Wait for
`Cluster/dev1` to show `STATUS: True / Available`...

But there is problem...

### Step 5.a - Review current state

Let's take a look at the provider state (docker).

```bash
docker ps
```

You should see something like:

```
CONTAINER ID   IMAGE                                COMMAND                  CREATED          STATUS          PORTS                                              NAMES
a5b3b7ae13f1   kindest/node:v1.35.0                 "/usr/local/bin/entr…"   4 minutes ago    Up 4 minutes                                                       dev1-md-0-j7mrp-ltqs2-b4nnz
41ce29439169   kindest/node:v1.35.0                 "/usr/local/bin/entr…"   9 minutes ago    Up 9 minutes    127.0.0.1:55008->6443/tcp                          dev1-kmffq-h8frl
0ee93f8dc250   kindest/haproxy:v20230606-42a2262b   "haproxy -W -db -f /…"   9 minutes ago    Up 9 minutes    0.0.0.0:55006->6443/tcp, 0.0.0.0:55007->8404/tcp   dev1-lb
242e7dbad8ce   kindest/node:v1.35.0                 "/usr/local/bin/entr…"   11 minutes ago   Up 11 minutes   127.0.0.1:62202->6443/tcp                          capi-mgmt-control-plane
```

CAPI (running from the kindest node on docker container `capi-mgmt-control-plane`) creates the 3 other machines as docker containers:

- `dev1-lb`: The load balancer for the new managed `dev1` cluster, using `ha-proxy`
- `dev1-kmffq-h8frl`: `dev1` cluster control plane node
- `dev1-md-0-j7mrp-ltqs2-b4nnz`: `dev1` cluster worker node

### Step 5.b — Install a CNI on dev1

CAPI provisions nodes but does not install a CNI.
The quickstart does this with
Nodes will remain `NotReady` until one is installed.
Get the `dev1` kubeconfig first:

```bash
./clusterctl get kubeconfig dev1 > dev1.kubeconfig
```

The kubeconfig points to a Docker-internal IP (e.g. `172.18.0.x`) which is not reachable
from macOS. Patch it to use the load balancer's localhost port instead:

```bash
# Point the kubeconfig to the exposed port of the load balancer, rather than the inaccessible container IP.
sed -i -e "s/server:.*/server: https:\/\/$(docker port dev1-lb 6443/tcp | sed "s/0.0.0.0/127.0.0.1/")/g" ./dev1.kubeconfig
```

Then install Calico, and watch for nodes to become ready:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.3/manifests/calico.yaml \
  --kubeconfig=dev1.kubeconfig --insecure-skip-tls-verify

kubectl get nodes --kubeconfig=dev1.kubeconfig --insecure-skip-tls-verify -w
```

Once the nodes become ready, you should see similar to following:

```
NAME                          STATUS   ROLES           AGE     VERSION
dev1-kmffq-h8frl              Ready    control-plane   8m45s   v1.35.0
dev1-md-0-j7mrp-ltqs2-b4nnz   Ready    <none>          3m24s   v1.35.0
```

## Verify cluster state

Check out the previous `clusterctl describe cluster dev1` command watch,
you should see similar to the following:

```
NAME                                                REPLICAS AVAILABLE READY UP TO DATE STATUS REASON     SINCE  MESSAGE
Cluster/dev1                                        2/2      2         2     2          True   Available  34s
├─ClusterInfrastructure - DockerCluster/dev1-6jwrk                                      True   Ready      7m37s
├─ControlPlane - KubeadmControlPlane/dev1-kmffq     1/1      1         1     1          True   Available  6m46s
│ └─Machine/dev1-kmffq-h8frl                        1        1         1     1          True   Ready      27s
└─Workers
  └─MachineDeployment/dev1-md-0-j7mrp               1/1      1         1     1          True   Available  34s
    └─Machine/dev1-md-0-j7mrp-ltqs2-b4nnz           1        1         1     1          True   Ready      34s
```

You can see some of the relationships between the [CAPI resources](https://cluster-api.sigs.k8s.io/user/concepts#concepts).

Take a look at some of the relate k8s resources:

```bash
kubectl get dockerclusters -o wide
```

```
NAME         CLUSTER   AGE
dev1-6jwrk   dev1      28m
```

```bash
kubectl get machines -o wide
```

If you drill into each machine resource (`kubectl get machine/<name>`), you can also see the nested `KubeadmConfig` resources bootstrapped with the associated `DockerMachine`.

```
NAME                          CLUSTER   NODE NAME                     PROVIDER ID                              READY   AVAILABLE   UP-TO-DATE   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         PAUSED   PHASE     AGE   VERSION
dev1-kmffq-h8frl              dev1      dev1-kmffq-h8frl              docker:////dev1-kmffq-h8frl              True    True        True         172.18.0.4    172.18.0.4    Debian GNU/Linux 12 (bookworm)   False    Running   28m   v1.35.0
dev1-md-0-j7mrp-ltqs2-b4nnz   dev1      dev1-md-0-j7mrp-ltqs2-b4nnz   docker:////dev1-md-0-j7mrp-ltqs2-b4nnz   True    True        True         172.18.0.5    172.18.0.5    Debian GNU/Linux 12 (bookworm)   False    Running   23m   v1.35.0
```

```bash
kubectl get dockermachines -o wide
```

```
NAME                          CLUSTER   MACHINE                       PROVIDERID                               READY   AGE
dev1-kmffq-h8frl              dev1      dev1-kmffq-h8frl              docker:////dev1-kmffq-h8frl              true    29m
dev1-md-0-j7mrp-ltqs2-b4nnz   dev1      dev1-md-0-j7mrp-ltqs2-b4nnz   docker:////dev1-md-0-j7mrp-ltqs2-b4nnz   true    23m
```

```bash
kubectl get kubeadmcontrolplanes -o wide
```

```
NAME         CLUSTER   AVAILABLE   DESIRED   CURRENT   READY   AVAILABLE   UP-TO-DATE   PAUSED   INITIALIZED   AGE   VERSION
dev1-kmffq   dev1      True        1         1         1       1           1            False    true          30m   v1.35.0
```

```bash
kubectl get machinedeployments -o wide
```

```
NAMESPACE   NAME              CLUSTER   AVAILABLE   DESIRED   CURRENT   READY   AVAILABLE   UP-TO-DATE   PAUSED   PHASE     AGE   VERSION
default     dev1-md-0-j7mrp   dev1      True        1         1         1       1           1            False    Running   38m   v1.35.0
```

At this point, feel free to experiment with the new `dev1` cluster,
perhaps run `kubecl get ...` on the above queried resorurces.

---

## Teardown

```bash
# Delete the workload cluster (CAPI will clean up Docker containers)
kubectl delete cluster dev1

# Delete the management cluster
kind delete cluster --name capi-mgmt
```

---

## Key Commands Reference

### Check clusterctl version

```bash
./clusterctl version
```

### Describe cluster topology and health

```bash
./clusterctl describe cluster dev1
```

### Access dev1 cluster

```bash
kubectl --kubeconfig=dev1-local.kubeconfig --insecure-skip-tls-verify <command>
```

---

## Configuration

All version and provider configuration lives in `.env`:

| Variable | Current Value | Purpose |
|---|---|---|
| `KUBE_VERSION` | `v1.35.0` | Kubernetes version for generated clusters |
| `CORE_VERSION` | `v1.12.4` | Cluster API core version |
| `INFRA_PROVIDER` | `docker` | Infrastructure provider |
| `INFRA_VERSION` | `v1.12.4` | Infrastructure provider version |
| `BOOTSTRAP_VERSION` | `v1.12.4` | Kubeadm bootstrap provider version |
| `CONTROL_PLANE_MACHINE_TYPE` | `control-plane` | Machine type label for control plane |
| `WORKER_MACHINE_TYPE` | `worker` | Machine type label for workers |

---

## Architecture

```
kind-config.yaml
     │
     ▼
`kind create cluster` (capi-mgmt)   ← Docker socket mounted from host
     │
     ▼
`CLUSTER_TOPOLOGY=true clusterctl init --infrastructure docker`
     │
     ▼
`clusterctl generate cluster dev1 --flavor development > dev1-cluster.yaml`
     │
     ▼
`kubectl apply -f dev1-cluster.yaml`
     │
     ▼
`kubectl apply` calico CNI on `dev1`
     │
     ▼
`dev1` nodes Ready
```

- **`kind-config.yaml`** — kind cluster config with Docker socket `extraMount` (required for CAPD on macOS)
- **`.env`** — source before running any commands; sets all version/provider variables
- **`clusterctl`** — CAPI CLI v1.12.4, downloaded from GitHub release (see Step 0)
- **`dev1-cluster.yaml`** — generated cluster manifest (ClusterClass-based, `quick-start` topology)

---

## Known Gotchas

| Symptom | Cause | Fix |
|---|---|---|
| `failed to get file "cluster-template.yaml" ... 404` | Docker provider has no default flavor template | Add `--flavor development` to `clusterctl generate` |
| `admission webhook denied ... ClusterTopology feature flag` | Feature gate off at init time | Use `CLUSTER_TOPOLOGY=true ./clusterctl init` |
| `Cannot connect to the Docker daemon at unix:///var/run/docker.sock` | Docker socket not mounted in kind node | Recreate kind cluster using `kind-config.yaml` |
| `dial tcp 172.18.0.x:6443: i/o timeout` | dev1 kubeconfig uses Docker-internal IP | Patch kubeconfig to use `localhost:<dev1-lb port>` |
| Nodes stuck `NotReady` | No CNI installed | Apply Calico (or another CNI) to the dev1 cluster |

---

## Binaries

- `clusterctl` — downloaded per Step 0; v1.12.4, darwin/arm64
