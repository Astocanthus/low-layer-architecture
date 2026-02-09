# LOW-LAYER Infrastructure Architecture

This document describes the **low-level infrastructure architecture** of the LOW-LAYER platform: bare metal bootstrap, root cluster, operator model, and organization provisioning.

> **Status**: Draft - Architecture discussion (not yet implemented)
>
> **Related**: [platform.md](platform.md) describes the logical platform model (tenants, orgs, environments). This document describes how it runs underneath.

---

## Design Principles

1. **Stateless**: No dedicated database. All state lives in Kubernetes etcd via CRDs
2. **Kubernetes-native**: Operators, CRDs, reconciliation loops, CAPI
3. **Single trust root**: OpenBao (Vault) for all PKI, secrets, and identity
4. **Recursive**: Low-Layer can host Low-Layer. Same stack everywhere
5. **Immutable infrastructure**: Fedora CoreOS, versioned images, no config drift

---

## Layer Model

```
LAYER 0 : Bare Metal
  └── Physical servers (Low-Layer datacenter or customer on-prem)

LAYER 1 : Root Kubernetes Cluster (Fedora CoreOS images)
  ├── Masters (Image A) : kubelet + OpenBao server (Raft HA, TPM unseal)
  └── Workers (Image B) : kubelet + OpenBao agent (cert rotation)

LAYER 2 : Platform Services (pods on root cluster)
  ├── ll-operator (single Go binary)
  ├── ll-console (Angular)
  ├── Keycloak (clustered) + PostgreSQL (clustered)
  ├── Rook Ceph (storage: PVC, Cinder backend, S3/RGW)
  ├── cert-manager (Vault issuer)
  └── CAPI controllers

LAYER 3 : OpenStack Minimal (Helm charts on root cluster)
  ├── Keystone, Nova, Neutron, Glance, Placement, Cinder
  └── Provides IaaS: VMs, networking, block storage (Ceph-backed)

LAYER 4 : Organization Kubernetes Clusters (VMs on OpenStack, provisioned by CAPI)
  ├── Control plane + workers
  ├── OpenBao tenant + Keycloak tenant + PostgreSQL (per org)
  ├── Low-Layer agents (DaemonSet)
  └── Client workloads
```

---

## Fedora CoreOS Images

Two immutable images for the root cluster nodes:

### Image A: `coreos-ll-master`

- **Fedora CoreOS** (rpm-ostree, Ignition)
- **OpenBao server** (podman quadlet, Raft storage, TPM auto-unseal)
- **OpenBao Agent** (cert templating for kubelet, apiserver, etcd)
- **Kubelet** (systemd, starts after Vault Agent has generated certs)
- **Cilium** (CNI)
- **Role**: Root cluster master (control plane)
- **Count**: 3 (or 5 for reinforced HA)

### Image B: `coreos-ll-worker`

- **Fedora CoreOS** (rpm-ostree, Ignition)
- **OpenBao Agent** (podman quadlet, cert rotation only)
- **Kubelet** (systemd, certs from Vault Agent)
- **Cilium** (CNI)
- **Role**: Root cluster worker (runs OpenStack + platform services)
- **Count**: N (scales with workload)

### Boot Sequence

```
Image A (Masters):
  t=0    VM boot, Ignition applies config
  t=5s   OpenBao server starts (podman), TPM auto-unseal
  t=8s   3 OpenBao nodes discover peers (DNS/static), form Raft cluster
  t=10s  Vault Agent generates kube certs from OpenBao PKI
  t=12s  Kubelet starts, forms Kubernetes control plane
  t=15s  Cilium initializes
  t=20s  Root cluster control plane = READY

Image B (Workers):
  t=0    VM boot, Ignition applies config
  t=5s   OpenBao Agent connects to masters, gets kubelet certs
  t=8s   Kubelet starts, joins control plane
  t=10s  Cilium initializes
  t=12s  Worker = READY
```

### OpenBao as System Service (not a Kubernetes pod)

OpenBao runs **below** Kubernetes at the OS level. This eliminates the chicken-and-egg problem: Vault exists before Kubernetes, so it can issue all certificates including the management cluster's own API server certs.

```
systemd dependency chain:
  network-online.target
    └── openbao.service (podman quadlet, TPM auto-unseal)
          └── openbao-agent.service (cert templating)
                └── kubelet.service (uses Vault-issued certs)
                      └── cilium
```

OpenBao on CoreOS runs as a **podman quadlet** (systemd-managed container), not a traditional binary. This respects the CoreOS philosophy: the OS is minimal and immutable, everything runs in containers.

---

## OpenBao PKI Hierarchy

```
OpenBao (systemd, root cluster masters)
└── PKI Secret Engine
    ├── Root CA "low-layer-root" (offline, 10y validity)
    │
    ├── Intermediate CA "management-cluster"
    │   ├── API server certs (root kube)
    │   ├── Kubelet certs (all root nodes)
    │   └── etcd certs
    │
    ├── Intermediate CA "org-{name}" (per organization, 1y, auto-renew)
    │   ├── API server certs (org kube)
    │   ├── Kubelet certs (org nodes)
    │   ├── Agent mTLS certs
    │   └── Application certs (via cert-manager Vault Issuer)
    │
    └── Auth backends
        ├── Kubernetes auth (root cluster)
        ├── Kubernetes auth (per org cluster)
        └── AppRole (for ll-operator)
```

Each organization gets its own intermediate CA. If an org is compromised, revoke its CA without impacting others.

---

## The Operator: `ll-operator`

Single Go binary running as a Kubernetes Deployment on the root cluster. Combines three roles:

```
ll-operator (single Go binary)
├── HTTP/REST API     → serves the Angular console
├── CRD Controllers   → watches LLOrganisation, LLEnvironment, LLPack, LLNode
├── gRPC Server       → agent connections (mTLS via OpenBao)
└── Helm SDK          → deploys charts to org clusters (via kubeconfig)
```

### Why a single binary

- No inter-process communication, no added latency
- CRD controllers and REST API share the same informer cache
- Single artifact to build, test, deploy, version
- Same pattern as ArgoCD (`argocd-server`)

### Internal architecture (hexagonal)

```
internal/
├── domain/          # Pure business types (Org, Env, Node, Pack)
├── ports/           # Interfaces (OrgRepository, AgentConnection)
├── operator/        # CRD controllers (implements ports)
├── api/             # REST handlers (implements ports)
├── agent/           # gRPC server + agent registry
└── helm/            # Helm SDK wrapper
```

### CRDs (on root cluster etcd)

| CRD | Purpose |
|-----|---------|
| `LLOrganisation` | Represents a customer organization |
| `LLEnvironment` | A Kubernetes cluster within an org |
| `LLNode` | Registered node (agent identity + license status) |
| `LLPack` | Available Helm chart package |
| `LLService` | Deployed service instance |

---

## The Agent: `ll-agent`

Lightweight DaemonSet running on every tagged node in org clusters. Two responsibilities only:

1. **License enforcement**: Proves the node is authorized (Vault-issued cert + gRPC heartbeat)
2. **Node monitoring**: Reports CPU, RAM, disk, network, kubelet status

### Agent is the security boundary

The agent is NOT a label tagger. Labels can be spoofed by anyone with `kubectl`. Instead, the operator controls node admission via **taints**, applied remotely through a privileged kubeconfig:

```
1. Node joins org cluster → automatic taint:
   low-layer.io/unlicensed:NoSchedule

2. Agent starts on node → connects to operator via gRPC (mTLS)

3. Operator verifies: valid cert? Org under node limit?

4. If OK → operator removes taint via kubeconfig:
   low-layer.io/unlicensed-

5. Node can now receive scheduled workloads

6. Agent disconnects / license expires →
   Operator re-applies taint → no new scheduling
```

Existing workloads continue running (taint only prevents new scheduling). The client cannot remove the taint because:
- RBAC on org cluster denies `nodes/patch` to client roles
- Or an admission webhook blocks removal of `low-layer.io/*` taints

### Chart deployment

Charts are deployed by the **operator** (not the agent). The operator has a kubeconfig to each org cluster's API server (obtained during CAPI provisioning). The agent's role is to authorize the node, not to deploy workloads.

### Resilience (kubelet pattern)

- Agent reconnects automatically with exponential backoff on operator restart
- On connection loss: continues with last known config
- Pending config changes stored as CRDs on the root cluster
- On reconnect: agent pulls delta, operator reconciles toward new desired state

---

## Organization Types

### Standard Organization (on OpenStack VMs)

The typical customer. Runs on VMs provisioned by OpenStack on the root cluster.

```
Org Kube (CAPI + OpenStack provider)
├── Control plane (3 nodes) + Workers (N nodes)
├── OpenBao tenant   (dedicated instance, org secrets)
├── Keycloak tenant  (dedicated realm, org users)
├── PostgreSQL       (backend for Keycloak)
├── Agents           (DaemonSet, license + monitoring)
└── Client workloads (deployed via LLPacks)
```

**Isolation**: Network (Neutron VPC) + Identity (dedicated Bao + Keycloak) + Storage (Ceph pools)

### Bare Metal / OpenStack Organization

A customer with their own bare metal. Installs Low-Layer to operate their own infrastructure.

```
Customer bare metal
├── Root cluster (same CoreOS images, same ll-operator)
├── OpenBao + Keycloak (host-level, NOT tenant)
├── OpenStack (if they want IaaS)
├── OR any CAPI provider (Hetzner, AWS, etc.)
└── Creates standard orgs on top (same model, recursively)
```

**No dedicated Bao/Keycloak tenant**: this org IS the root. OpenStack on bare metal means the customer operates the same stack as Low-Layer itself. The only difference: their `ll-operator` contacts the Low-Layer key server for licensing.

---

## Recursive Model

The same stack deploys at every level:

```
Low-Layer (datacenter)                    Customer "megacorp" (bare metal)
├── Root cluster (CoreOS)                 ├── Root cluster (same CoreOS)
├── OpenBao + Keycloak + PG              ├── OpenBao + Keycloak + PG
├── OpenStack                             ├── OpenStack (or CAPI Hetzner)
├── ll-operator → key server (self)       ├── ll-operator → key server (Low-Layer)
│                                         │
├── Org "acme" (standard)                 ├── Org "team-dev" (standard)
│   └── Bao tenant + KC tenant            │   └── Bao tenant + KC tenant
│                                         │
└── Org "beta" (standard)                 └── Org "team-prod" (standard)
    └── Bao tenant + KC tenant                └── Bao tenant + KC tenant
```

Customers with bare metal can also use multiple CAPI providers simultaneously:

```
megacorp root cluster
└── ll-operator
    ├── CAPI OpenStack → org kubes on their own OpenStack
    ├── CAPI Hetzner   → org kubes on Hetzner cloud
    └── CAPI AWS       → org kubes on AWS (if needed)
```

---

## Storage: Rook Ceph

Rook Ceph on the root cluster provides unified storage for everything:

| Consumer | Ceph Feature | Usage |
|----------|-------------|-------|
| Kubernetes PVCs | RBD (block) | PostgreSQL, Keycloak, etcd, stateful apps |
| OpenStack Cinder | RBD (block) | VM persistent disks |
| OpenStack Glance | RGW (S3) | VM images |
| Audit logs | RGW (S3) | Append-only export endpoint (ISO 27001) |
| Backups | RGW (S3) | etcd snapshots, org data |

---

## Product Tiers

### Free — "Kubernetes Native"

A visual interface for any existing Kubernetes cluster. No agent, no infrastructure provisioning.

```
helm install low-layer oci://registry.low-layer.io/charts/low-layer
```

What gets deployed (2 pods total):

```
├── ll-operator (mode: native)   → watches native K8s resources via ServiceAccount
└── ll-console                   → graph editor for visual orchestration
```

What the user gets:
- Console with graph editor to visualize and orchestrate native K8s resources (Deployment, StatefulSet, Service, Ingress, ConfigMap, Secret, Job, CronJob, PVC...)
- No agent on nodes (zero overhead)
- No Keycloak, no OpenBao, no PostgreSQL
- LLPacks catalog visible in the UI but locked ("upgrade to unlock")

What the user does NOT get:
- Managed services (PostgreSQL, Redis, Kafka...)
- Multi-org / multi-environment
- Node monitoring via agents
- CAPI provisioning

**Purpose**: Adoption hook. Users discover the interface for free, see the catalog, upgrade when they need managed services.

### Paid — "Platform Engineering"

Full platform with managed services, multi-org, and agents.

```
✓ Everything in Free
✓ LLPacks catalog (PostgreSQL, Redis, Kafka, RabbitMQ...)
✓ Multi-org, multi-environment
✓ OpenBao tenant + Keycloak tenant per org
✓ CAPI provisioning (OpenStack, Hetzner, AWS...)
✓ Agents on nodes (license enforcement + monitoring)
✓ 1 org + 5 nodes included, per-node billing after
```

### Enterprise — "Self-Hosted"

Full stack on customer bare metal. Same architecture as Low-Layer's own infrastructure.

```
✓ Everything in Paid
✓ Root cluster (CoreOS images)
✓ OpenStack integrated
✓ Recursive model (Low-Layer hosts Low-Layer)
✓ Unlimited nodes for Low-Layer's own infra
✓ Multiple CAPI providers simultaneously
```

### Operator Modes

The operator adapts its behavior based on the active tier. No reinstallation needed — upgrading the license activates additional controllers:

```go
type OperatorMode string

const (
    ModeNative     OperatorMode = "native"      // Free: K8s native resources only
    ModePlatform   OperatorMode = "platform"     // Paid: + LLPacks + multi-org + agents
    ModeEnterprise OperatorMode = "enterprise"   // Self-hosted: + OpenStack + CAPI
)
```

| Capability | native | platform | enterprise |
|------------|--------|----------|------------|
| Console + graph editor | yes | yes | yes |
| Native K8s orchestration | yes | yes | yes |
| LLPacks catalog | locked | yes | yes |
| Agents (DaemonSet) | no | yes | yes |
| Multi-org | no | yes | yes |
| Bao/KC tenant per org | no | yes | yes |
| CAPI provisioning | no | yes | yes |
| OpenStack integration | no | no | yes |
| CoreOS image management | no | no | yes |

---

## Licensing & Billing

### Enforcement (platform & enterprise tiers only)

- Each agent proves a node's identity via Vault-issued certificate (hardware fingerprint)
- The operator validates against the key server and maintains a node registry (LLNode CRDs)
- Node count = number of active, authenticated agent connections
- Grace period if key server is unreachable (cached authorization)
- Free tier has no agent and no node counting

---

## Sequence: Provisioning a New Organization

```
Client          Console        Operator        CAPI       OpenStack      Org Kube       Agent
  │                │               │              │           │              │             │
  │  "Create org"  │               │              │           │              │             │
  │───────────────▶│──────────────▶│              │           │              │             │
  │                │               │──"provision"─▶│           │              │             │
  │                │               │              │──"VMs"───▶│              │             │
  │                │               │              │           │──"create"───▶│             │
  │                │               │              │           │              │ (cluster up)│
  │                │               │◄─────────────│ kubeconfig│              │             │
  │                │               │                                         │             │
  │                │               │──deploy DaemonSet agents───────────────▶│             │
  │                │               │──deploy Bao tenant + KC tenant────────▶│             │
  │                │               │──apply taint low-layer.io/unlicensed──▶│             │
  │                │               │                                         │             │
  │                │               │                                         │──start─────▶│
  │                │               │◄───────────────────────────────gRPC register──────────│
  │                │               │                                         │             │
  │                │               │──verify license (key server)            │             │
  │                │               │──remove taint─────────────────────────▶│             │
  │                │               │                                  nodes ready          │
  │                │               │                                         │             │
  │  "Deploy PG"   │               │                                         │             │
  │───────────────▶│──────────────▶│                                         │             │
  │                │               │──helm install postgresql──────────────▶│             │
  │                │               │                                    scheduled         │
  │                │               │◄─────────metrics────────────────────────│◄────────────│
  │                │◄──────────────│                                         │             │
  │◄───────────────│  "PG ready"   │                                         │             │
```

---

## Key Decisions Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| State management | CRDs on etcd (no dedicated DB) | Kubernetes-native, no DB lifecycle to manage |
| Trust root | OpenBao at OS level (systemd, not pod) | Eliminates chicken-and-egg with Kubernetes certs |
| Auto-unseal | TPM per node | Hardware-bound, no cloud dependency |
| Cert management | OpenBao Agent on every node | Uniform rotation, single PKI source of truth |
| OS images | 2x Fedora CoreOS (master/worker) | Immutable, auditable, reproducible |
| Authentication | Keycloak clustered + PG, OIDC | Standard, federable |
| Storage | Rook Ceph | Unified: block (PVC/Cinder), object (S3/RGW) |
| IaaS | OpenStack minimal + Cinder | Bare metal → VMs for org clusters |
| Org provisioning | CAPI (OpenStack provider, or others) | Pluggable, supports multi-provider |
| Operator design | Single Go binary (REST + CRD + gRPC) | Simplicity, shared cache, single artifact |
| Agent design | Lightweight DaemonSet (license + monitoring) | Security boundary, not a deployer |
| Chart delivery | Helm SDK in operator, Low-Layer Helm repo | Standard OCI registry, simplified charts |
| Console | Angular, dedicated to its operator | No multi-management complexity |
| Org isolation | Dedicated Bao tenant + Keycloak tenant + PG | Full identity and secrets isolation per org |
| Bare metal orgs | Same stack, recursive | Customer runs same Low-Layer, key server for billing |
| Licensing | Agent = node proof, key server validation | Hardware-bound identity, grace period |
