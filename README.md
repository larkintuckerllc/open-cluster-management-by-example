# Open Cluster Management by Example

A hands-on guide to exploring [Open Cluster Management](https://open-cluster-management.io/) (OCM) using local Kubernetes clusters.

---

## Prerequisites

The following tools must be installed before proceeding:

- **Docker** — used by kind to run Kubernetes nodes as containers
- **kind** v0.31.0 or later — creates local Kubernetes clusters inside Docker
- **kubectl** — interacts with the Kubernetes API on both hub and managed clusters

> **macOS note:** On macOS, Docker containers run inside a Linux VM. This means the Docker internal network (`172.18.x.x`) is not directly reachable from the host. The fix is documented in the [Joining](#joining-cluster1-to-the-hub) section below.

---

## Installation

### 1. Install clusteradm

`clusteradm` is the OCM CLI used to initialize the hub and register managed clusters.

```bash
curl -L https://raw.githubusercontent.com/open-cluster-management-io/clusteradm/main/install.sh | INSTALL_DIR=$HOME/.local/bin bash
```

The script auto-detects your OS and architecture. On macOS it installs to `$HOME/.local/bin` to avoid requiring `sudo`.

Ensure the install directory is on your PATH:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
```

Verify the installation:

```bash
clusteradm version
```

### 2. Create the hub cluster

The hub cluster requires a custom kind configuration to add `host.docker.internal` to the API server's TLS certificate Subject Alternative Names (SANs). This is necessary on macOS so that the klusterlet agents running inside Docker containers can reach the hub API server by a hostname that resolves correctly from both the Mac host and within containers.

Save the following as `kind-hub.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
kubeadmConfigPatches:
- |
  kind: ClusterConfiguration
  apiServer:
    certSANs:
    - "localhost"
    - "127.0.0.1"
    - "host.docker.internal"
    - "kubernetes"
    - "kubernetes.default"
    - "kubernetes.default.svc"
    - "kubernetes.default.svc.cluster.local"
```

Create the hub cluster:

```bash
kind create cluster --name hub --image kindest/node:v1.32.11 --config kind-hub.yaml
```

### 3. Initialize OCM on the hub

```bash
clusteradm init --context kind-hub --wait
```

On success, the command prints a `clusteradm join` command containing a bootstrap token and hub API server URL. Save this output — you will need the token in the [Joining](#joining-cluster1-to-the-hub) section.

Verify all hub components are running:

```bash
kubectl get pods -n open-cluster-management --context kind-hub
kubectl get pods -n open-cluster-management-hub --context kind-hub
```

You should see the following pods all in `Running` state:

| Namespace | Pod | Role |
|---|---|---|
| `open-cluster-management` | `cluster-manager` (x3) | Operator that manages the hub component lifecycle |
| `open-cluster-management-hub` | `cluster-manager-registration-controller` | Handles cluster registration and lifecycle |
| `open-cluster-management-hub` | `cluster-manager-registration-webhook` | Validates registration-related resources |
| `open-cluster-management-hub` | `cluster-manager-placement-controller` | Implements the Placement API for workload scheduling |
| `open-cluster-management-hub` | `cluster-manager-addon-manager-controller` | Manages the add-on framework lifecycle |
| `open-cluster-management-hub` | `cluster-manager-addon-webhook` | Validates add-on resources |
| `open-cluster-management-hub` | `cluster-manager-work-webhook` | Validates ManifestWork resources |

### 4. Create the managed cluster (cluster1)

```bash
kind create cluster --name cluster1 --image kindest/node:v1.32.11
```

---

## Joining cluster1 to the Hub

### macOS networking background

`clusteradm join` uses a single `--hub-apiserver` URL for two purposes:

1. **At install time** — the `clusteradm` process (running on your Mac) fetches the hub's CA certificate to validate connectivity.
2. **At runtime** — the klusterlet agent pod (running inside cluster1's Docker container) uses the same URL to call back to the hub.

This creates a conflict on macOS:

| URL | Reachable from Mac? | Reachable from inside cluster1 container? |
|---|---|---|
| `https://127.0.0.1:<port>` | Yes | No (`127.0.0.1` = the pod itself) |
| `https://172.18.0.2:6443` | No (Docker VM network) | Yes |

The solution is `host.docker.internal` — a hostname provided by Docker Desktop that resolves to the Mac host from inside containers, making it reachable from both sides. This is why `host.docker.internal` must be included in the hub's TLS certificate SANs (configured in `kind-hub.yaml` above).

However, `host.docker.internal` only resolves automatically *inside* Docker containers. The macOS host itself does not resolve it by default, which means the `clusteradm join` command (running on the host) would also fail. The fix is to add `host.docker.internal` to your macOS `/etc/hosts` file:

```bash
sudo sh -c 'echo "127.0.0.1 host.docker.internal" >> /etc/hosts'
```

Verify it resolves:

```bash
ping -c 1 host.docker.internal
```

> This networking complexity is **macOS-specific**. On Linux, the Docker network is directly accessible from the host, so `172.18.0.2:6443` would work from both sides without any additional configuration.

### Join cluster1

Using the bootstrap token from `clusteradm init`, run the join command with `host.docker.internal` as the hub API server hostname. Replace `<token>` with the token from the init output, and `<port>` with the port kind assigned to the hub:

```bash
clusteradm join \
  --hub-token <token> \
  --hub-apiserver https://host.docker.internal:<port> \
  --cluster-name cluster1 \
  --context kind-cluster1 \
  --wait
```

### Accept the registration on the hub

OCM requires an administrator to explicitly approve a cluster's join request. On the hub:

```bash
clusteradm accept --clusters cluster1 --context kind-hub --wait
```

### Verify

```bash
kubectl get managedclusters --context kind-hub
```

Expected output:

```
NAME       HUB ACCEPTED   MANAGED CLUSTER URLS                  JOINED   AVAILABLE   AGE
cluster1   true           https://cluster1-control-plane:6443   True     True        ...
```

The `cluster1` cluster is now fully registered and managed by the hub.

---

## Resources installed by OCM

When `clusteradm join` runs on cluster1 and `clusteradm accept` approves it on the hub, OCM installs resources on both sides.

### Hub-side resources

| Resource | Kind | Description |
|---|---|---|
| `cluster1` | `ManagedCluster` | The hub's representation of cluster1; holds labels, capacity, and status conditions |
| `cluster1` | `Namespace` | Dedicated namespace on the hub for all cluster1-scoped resources |
| `cluster1-registration-agent` | `ClusterRole` / `ClusterRoleBinding` | Grants cluster1's klusterlet agent permission to update its own `ManagedCluster` status |
| `cluster1-work-agent` | `ClusterRole` / `ClusterRoleBinding` | Grants the work agent permission to read `ManifestWork` resources in the `cluster1` namespace |
| (approved) | `CertificateSigningRequest` | The CSR the klusterlet submitted for its client cert; approved by `clusteradm accept` |

### cluster1-side resources

| Resource | Namespace | Kind | Description |
|---|---|---|---|
| `klusterlet` | `open-cluster-management-agent` | `Klusterlet` (CRD) | Top-level custom resource representing the agent configuration |
| `klusterlet` | `open-cluster-management-agent` | `Deployment` | Operator that manages the lifecycle of the registration and work agents |
| `klusterlet-registration-agent` | `open-cluster-management-agent` | `Deployment` | Registers cluster1 with the hub, maintains heartbeat, and handles CSR rotation |
| `klusterlet-work-agent` | `open-cluster-management-agent` | `Deployment` | Watches `ManifestWork` resources on the hub and applies them locally |
| `klusterlet` | `open-cluster-management-agent` | `ServiceAccount` + RBAC | Identity and permissions used by the klusterlet operator |
| `bootstrap-hub-kubeconfig` | `open-cluster-management-agent` | `Secret` | Short-lived kubeconfig using the bootstrap token, used only during initial registration |
| `hub-kubeconfig-secret` | `open-cluster-management-agent` | `Secret` | Long-lived kubeconfig with the client cert issued after CSR approval; used by agents at runtime |
| OCM CRDs | cluster-scoped | `CustomResourceDefinition` | Installs `klusterlets.operator.open-cluster-management.io` and related types |

### Registration flow

The bootstrap token secret gets cluster1 connected initially → the klusterlet submits a CSR → the hub admin approves it via `clusteradm accept` → a signed client cert is issued → the bootstrap secret is replaced by the permanent `hub-kubeconfig-secret` → the `ManagedCluster` resource on the hub moves to `Available: True`.

---

## Deploying resources with ManifestWork

`ManifestWork` is the core OCM primitive for pushing Kubernetes resources from the hub to a managed cluster. You define what you want deployed, on which cluster, and OCM's work agent handles the rest.

### How it works

A `ManifestWork` resource lives in the managed cluster's namespace on the hub (e.g., `cluster1`). The klusterlet work agent on the managed cluster watches for these resources and applies the embedded manifests locally. The hub then tracks the applied status.

### Hello world example

The following `ManifestWork` deploys a `ConfigMap` to cluster1 without ever touching cluster1 directly.

Save the following as `manifestwork-hello.yaml`:

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: hello-world
  namespace: cluster1        # must match the managed cluster name on the hub
spec:
  workload:
    manifests:
      - apiVersion: v1
        kind: ConfigMap
        metadata:
          name: hello-ocm
          namespace: default
        data:
          message: "Hello from the hub!"
```

Apply it on the hub:

```bash
kubectl apply -f manifestwork-hello.yaml --context kind-hub
```

### Verify delivery

Check that the `ConfigMap` appeared on cluster1:

```bash
kubectl get configmap hello-ocm -n default --context kind-cluster1
kubectl get configmap hello-ocm -n default --context kind-cluster1 -o jsonpath='{.data.message}'
```

Expected output:

```
Hello from the hub!
```

You can also inspect the delivery status on the hub. The `status.conditions` field reports whether the work agent successfully applied the resources:

```bash
kubectl get manifestwork hello-world -n cluster1 --context kind-hub -o yaml
```

Look for a condition with `type: Applied` and `status: "True"`.

### Cleanup

Deleting the `ManifestWork` from the hub causes OCM to remove the resources from cluster1 automatically:

```bash
kubectl delete manifestwork hello-world -n cluster1 --context kind-hub
```

Verify the `ConfigMap` is gone from cluster1:

```bash
kubectl get configmap hello-ocm -n default --context kind-cluster1
```

---

## Other OCM use cases

### Placement API

`Placement` makes workload distribution dynamic. Instead of hardcoding a managed cluster name in a `ManifestWork`, you define selection criteria — labels, cluster properties, or resource availability — and OCM resolves which clusters match at scheduling time.

**Example:** "Deploy to all clusters in the `us-east` region that have at least 4 available CPUs."

This is the foundation for multi-cluster scheduling and is used by several higher-level OCM features.

### ManifestWorkReplicaSet

Combines `ManifestWork` with `Placement` into a single hub-side resource. Define a set of manifests once and a placement rule, and OCM creates and manages individual `ManifestWork` resources for every matching cluster automatically. This is the idiomatic way to deploy resources fleet-wide without managing per-cluster objects by hand.

### Policy-based governance

The `governance-policy-framework` add-on introduces a `Policy` API for describing desired cluster state as compliance rules. The framework audits all managed clusters against the policies and reports compliance status on the hub. Policies can also auto-remediate violations.

**Example use cases:** enforcing RBAC baselines, requiring specific `NetworkPolicy` resources, or auditing security configurations fleet-wide.

### Application lifecycle (Subscriptions)

The `application-manager` add-on provides a GitOps-style application delivery model. A `Channel` resource points to a Git repository, Helm chart registry, or object store. A `Subscription` binds a set of clusters (via `Placement`) to a channel, and OCM pulls and deploys the application content automatically when changes are detected.

### Add-on framework

OCM has a first-class framework for building and distributing agents across managed clusters. Several official add-ons are available beyond those listed above:

| Add-on | What it does |
|---|---|
| `cluster-proxy` | Opens a reverse tunnel from the hub to managed clusters — useful when managed clusters are not directly reachable from the hub |
| `managed-serviceaccount` | Projects a `ServiceAccount` token from the hub into managed clusters, enabling cross-cluster API access without manual credential distribution |

---

## Alternatives to OCM

OCM's core capability — declaring resources on a control plane and propagating them to managed clusters — is shared by several other open source tools. The right choice depends on whether the primary concern is application delivery or broader fleet management.

### GitOps-based tools

**Argo CD**
The most widely adopted multi-cluster delivery tool. Each managed cluster is registered as a target, and `Application` resources define what to deploy from a Git repository. The `ApplicationSet` controller extends this to generate `Application` resources dynamically across many clusters — the multi-cluster equivalent of OCM's `ManifestWorkReplicaSet`. Argo CD pulls from Git; OCM pushes from the hub.

**Flux CD**
Similar GitOps model to Argo CD but more composable and Kubernetes-native in its API design. `Kustomization` and `HelmRelease` resources define what to deploy. Multi-cluster is handled by running a Flux instance per cluster or by using a central management cluster with `Kubeconfig` references to remotes.

### Multi-cluster control plane tools

**Karmada**
The closest architectural equivalent to OCM — also a CNCF project, also hub-and-spoke, also with its own propagation policy API for scheduling resources across clusters. Karmada's model is closer to "extend the Kubernetes API across clusters": you apply a `Deployment` to Karmada and it distributes it, rather than wrapping it in a `ManifestWork` envelope.

**Crossplane**
Focuses on infrastructure provisioning (databases, cloud resources) rather than application deployment, but uses the same control-plane model. More relevant when managing cloud resources across environments than deploying application manifests.

### Comparison

| Tool | Model | Source of truth | Multi-cluster scheduling |
|---|---|---|---|
| OCM | Push from hub | Hub cluster | Placement API |
| Argo CD | Pull from Git | Git repository | ApplicationSet |
| Flux CD | Pull from Git | Git repository | Per-cluster instances |
| Karmada | Push from hub | Hub cluster | PropagationPolicy |
| Crossplane | Reconcile loop | Hub cluster | N/A (infrastructure focus) |

### When to use OCM vs the alternatives

OCM and Karmada are **cluster fleet management** platforms — they treat multi-cluster as a first-class problem and add cluster lifecycle, add-ons, and compliance policy governance on top of resource propagation.

Argo CD and Flux are **GitOps delivery** tools — Git is the source of truth, and multi-cluster is achieved by targeting multiple clusters from a central place. They do not manage cluster registration, agent lifecycle, or compliance policy.

In practice, Argo CD is most commonly reached for when the primary concern is application delivery from Git. OCM tends to appear when the problem is broader fleet management — cluster registration, compliance enforcement, and workload scheduling across a large number of clusters. The two are not mutually exclusive: it is common to use Argo CD for application delivery on top of an OCM-managed fleet.
