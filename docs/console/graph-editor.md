# Graph Editor Architecture

The graph editor (`features/graph-editor/`) provides an interactive infrastructure topology designer built on **Rete.js v2**. It models real website/service architecture with typed connections between infrastructure components.

---

## Topology Model

The graph represents a Kubernetes-based infrastructure stack. Nodes are **color-coded by category**:

| Category | Color | Nodes |
|----------|-------|-------|
| **Network** | green | `InternetNode` (circle), `GatewayNode`, `VirtualServiceNode` (virtual), `K8sServiceNode` |
| **Workload** | dark | `ApplicationNode` (Node.js, Go, etc.), `DeploymentNode`, `StatefulSetNode`, `JobNode`, `CronJobNode` |
| **Database** | blue | `DatabaseNode` (PostgreSQL, etc.), `ConfigStoreNode` (etcd) |
| **CA** | yellow | `SecretNode` (OpenBao/Vault — PKI & Secrets) |
| **Config** | gray | `K8sSecretNode`, `ConfigMapNode` |
| **Storage** | pink | `PvcNode` |
| **Identity** | purple | `KeycloakNode` (IDP hub) |
| **Container** | orange | `OciImage` (sub-resource, via preview modal) |

Static nodes (Internet, Gateway, OpenBao, Keycloak) are placed by the system and not available in the user toolbar. The Internet→Gateway connection is a **pseudo-connection**: non-interactive, no visible pins, no delete icon — the network entry point is system-managed.

---

## Node Scope & Org-Level Graph

Every graph node carries a `scope: 'org' | 'env'` discriminator that determines ownership and editability:

| Scope | Stored In | Editable | Examples |
|-------|-----------|----------|----------|
| `org` | `OrgGraphDocument` (per-organization) | Locked (sub-graphs only) | Keycloak, OpenBao, dedicated DB, network chain, CA |
| `env` | `GraphDocument` (per-environment) | Full user control | User workloads, databases, config, PVCs |

**Bastion node types** (all `scope: 'org'`, system-managed):

| Classification | Node Types | User Can Edit |
|---------------|------------|---------------|
| Hub nodes | `keycloak`, `secret` (OpenBao) | Sub-graph resources only |
| Managed nodes | `database`, `internet`, `gateway`, `virtual-service`, `k8s-service` | No (locked) |

**Split storage model:**

```
Organisation
├── OrgGraphDocument (bastion-graph)     ← Shared across all environments
│   ├── spec: org nodes + org-to-org connections
│   ├── layout: positions for org nodes
│   └── status: operator feedback
│
└── Environment
    └── GraphDocument (env graph)        ← Per-environment
        ├── spec: env nodes + cross-scope connections (org→env)
        ├── layout: positions for env nodes
        └── status: operator feedback
```

**Workload OCI Image sub-resources:** All workload nodes (Application, Deployment, StatefulSet, Job, CronJob) support OCI Image sub-resources, managed through the node preview modal rather than the main graph canvas. OCI Image data is stored in `spec.subGraphs[workloadNodeId]` within the graph document. See the **Node Preview Modal** section below for details.

**Merged view:** At load time, `EditorService.loadGraphWithBastion()` calls `forkJoin` to fetch both graphs in parallel, then merges them into a single `GraphDocument` for the Rete.js editor. Org nodes get `scope: 'org'` and `locked: true`; env nodes default to `scope: 'env'`. Deduplication by node ID prevents conflicts with stale data.

**Scope-aware save:** On save, `exportGraphData()` filters OUT org nodes and org-to-org connections — only env-level data is sent to the env API. A separate `exportOrgGraphData()` exports org-only data to the org API.

**Cross-scope connections** (e.g., `gateway-1 → vs-app-1`, `kc-1 → app-1`) reference org node IDs from the env graph. They are stored in the env graph because they are environment-specific (each env may connect different workloads to the shared bastion).

**Org-level auto-arrange:** Org-level nodes are automatically positioned using ELK (`elkjs`) via `rete-auto-arrange-plugin`. The layout algorithm is **layered** with **left-to-right (LR)** direction, using 80px spacing between nodes and 120px spacing between layers. Auto-arrange runs on every graph load — org-level node positions are not persisted. Env-level nodes retain manual drag-and-drop positioning with saved positions. Dependencies: `elkjs@^0.8.2`, `rete-auto-arrange-plugin@^2.0.2`.

**Visual indicators:** Org nodes display a blue "ORG" badge, dashed border, and slightly reduced opacity. The node preview modal shows an info banner: "Shared org-level service — configuration applies to all environments."

**System-managed connections:** Connections between two org-level nodes carry `isSystemManaged: true`. These connections are visually rendered (retaining socket-based color: green, yellow, purple, etc.) but are completely inert — no hover effect, no configure button, no click handler. This reflects that bastion infrastructure wiring is platform-managed, not user-editable.

**API endpoints:**

| Endpoint | Scope | Purpose |
|----------|-------|---------|
| `GET/POST /api/environments/:envId/graph` | env | User workload graph |
| `GET/POST /api/organizations/:orgId/bastion-graph` | org | Service Bastion graph |
| `GET/POST /api/organizations/:orgId/bastion-graph/subgraphs/:hubNodeId` | org | Hub node sub-graph resources |

---

## Typed Socket/Pin System

Connections are **type-safe**: only pins of the same socket type can connect. The system uses 7 socket types organized into 6 visual groups:

```
Socket Types           Visual Groups
───────────────────    ─────────────────────────
network  ─────────┐   network  (#22c55e, solid)
route    ─────────┘
data     ─────────── data     (#3b82f6, solid)
cert     ─────────── cert     (#eab308, dashed)
config   ─────────── config   (#6b7280, solid)
storage  ─────────── storage  (#f472b6, solid)
identity ─────────── identity (#a855f7, solid)
```

- **Network group** (green): `network` socket (Internet→Gateway) and `route` socket (Gateway→VirtualService→K8sService→Workload)
- **Data group** (blue): `data` socket (Workload→Database, Workload→ConfigStore)
- **Cert group** (yellow, dashed): `cert` socket (Secret→Workload)
- **Config group** (gray): `config` socket (K8sSecret/ConfigMap→Workload)
- **Storage group** (pink): `storage` socket (PVC→Workload)
- **Identity group** (purple): `identity` socket (Keycloak→Workload, OIDC credentials + endpoints)

All workload nodes (Application, Deployment, StatefulSet, Job, CronJob) accept `config`, `storage`, and `identity` inputs.

---

## Connection Validation — 5-Layer Model

Connection attempts are validated by `canMakeTypedConnection()` through 5 successive layers. Each layer can reject the connection independently:

| Layer | Check | Rejects |
|-------|-------|---------|
| 1. **Same-side** | Output→Output or Input→Input | Wiring two outputs together |
| 2. **Socket group** | Source group ≠ Target group | Green pin to blue pin |
| 3. **Exact socket type** | `network` ≠ `route` (same group, different type) | Gateway:out(route) to Internet:in(network) |
| 4. **Cardinality** | Output or input with card. `1` already connected | Second connection from Internet:out, second route into a VirtualService |
| 5. **Node compatibility** | `NODE_PIN_COMPATIBILITY` map lookup | Application:out(data) → K8sSecret:config-out |

If layer 5 rejects but an **auto-chain** exists for the socket group, the system triggers automatic intermediate node creation instead (see Auto-Chain below).

---

## Node-Level Pin Compatibility — Per-Service Pin Typing

Each input pin declares which **source node types** it accepts via `NODE_PIN_COMPATIBILITY`. This enforces service-level typing beyond socket colors:

```
NODE_PIN_COMPATIBILITY: Map<targetPin, Set<sourceNodeType>>

  gateway:in          → { internet }
  virtual-service:in  → { gateway }
  k8s-service:in      → { virtual-service }
  workload:in         → { k8s-service }
  database:in         → { application, deployment, statefulset, keycloak }
  config-store:in     → { application, deployment, statefulset, keycloak }
  workload:cert       → { secret }
  workload:config     → { k8s-secret, configmap }
  workload:storage    → { pvc }
  workload:identity   → { keycloak }
```

**Design rationale**: Socket types enforce **category-level** compatibility (green only connects to green), while `NODE_PIN_COMPATIBILITY` enforces **service-level** constraints (within the green category, a `gateway:in` pin only accepts connections from `internet` nodes, not from `k8s-service`). This two-tier system allows fine-grained topology rules without multiplying socket types.

**Workload types** (`WORKLOAD_NODE_TYPES`) group: `application`, `deployment`, `statefulset`, `keycloak`, `job`, `cronjob`. Database and ConfigStore accept connections from any workload except Job/CronJob (stateless batch nodes). Note: `keycloak` is visually in the Identity category (purple) but remains in the workload type group for data/database compatibility. All workload nodes (Application, Deployment, StatefulSet, Job, CronJob) have `hasOciImages: true` and display OCI Image cards with bottom orange pins in the node preview modal.

---

## Pin Cardinality — CRD Block / Nested Block

Each pin carries a **cardinality** (`1` or `n`) displayed as a small colored label next to the socket circle. This models the Kubernetes CRD field structure:

| Cardinality | CRD Concept | Behavior | Example |
|-------------|-------------|----------|---------|
| `1` | **block** (single object) | Max 1 connection | `spec.selector`, `spec.template` |
| `n` | **nested block** (array) | Unlimited connections | `volumes[]`, `volumeClaimTemplates[]` |

Cardinality is declared per pin via `BaseNode.addTypedInput(key, socket, cardinality)` and `addTypedOutput(key, socket, cardinality)`. These helpers synchronize `pinCardinality` map with Rete's `multipleConnections` flag on inputs.

**Cardinality by node type:**

| Node | Pin | Dir | Card. | CRD Mapping |
|------|-----|-----|-------|-------------|
| Internet | out | O | 1 | Single egress |
| Gateway | in / out | I/O | 1 / n | block / listeners[] |
| VirtualService | in / out | I/O | 1 / 1 | block / block |
| K8sService | in / out | I/O | 1 / 1 | block / block |
| Workloads | in | I | n | nested: service refs |
| Workloads | out | O | n | nested: data connections |
| Workloads | cert | I | n | nested: volumes[].secret |
| Workloads | config | I | n | nested: volumes[].configMap |
| Workloads | storage | I | n | nested: volumeMounts[] |
| Workloads | identity | I | n | nested: OIDC client bindings |
| Keycloak | identity-out | O | n | nested: OIDC clients |
| Database / ConfigStore | in | I | n | nested: clients |
| Secret (OpenBao) | cert-out | O | n | nested: cert consumers |
| K8sSecret / ConfigMap | config-out | O | n | nested: consumers |
| PVC | storage-out | O | 1 | block: RWO mount |

**Enforcement model:**
- **Output `1`**: Blocked by `checkCardinality(side='output')` — user must delete existing connection first
- **Input `1`**: Blocked by `checkCardinality(side='input')` — user must delete existing connection first
- **Input/Output `n`**: No limit, multiple connections allowed

> **Note**: Rete's native `multipleConnections: false` on inputs is not relied upon for enforcement — the removal guard (`setupConnectionRemovalGuard`) would silently bypass it. All cardinality enforcement goes through `checkCardinality()` in `canMakeTypedConnection()`.

**Future**: Cardinality tables will be populated dynamically from CRD OpenAPI schemas when the `ll-operator` CRDs are finalized.

---

## Connection Spread

When multiple connections share the same source output or target input (cardinality `n`), bezier curves would overlap visually. The **spread system** offsets control points to separate them:

- `getConnectionSpread(connectionId)` computes sibling index among connections sharing the same source/target
- Source siblings offset `c1y`, target siblings offset `c2y` (bezier control points)
- Spread: 30px per connection, centered around 0 (e.g., 2 connections → -15px / +15px)
- Endpoints remain fixed on sockets — curves fan out at the midpoint only

---

## Relation Editor — Link Blueprint System

Clicking the configure button on a connection opens the **Relation Editor**, a 3-column slide-in overlay for defining how two nodes relate at the infrastructure level. The system maps visual graph connections to concrete Kubernetes resource creation workflows.

**3-column layout:**
- **Left**: Source node info (type, label, existing connections)
- **Center**: Blueprint selection → multi-step resolution wizard
- **Right**: Target hub node resource pool (tree hierarchy for OpenBao/Keycloak)

### Blueprint Model

```
LinkBlueprint {
  id: string                           # e.g. "app-openbao-cert"
  label: string                        # "Certificate Request"
  socketGroup: SocketGroup             # cert, config, storage, data
  source/target: NodeTypeName          # Directional matching
  description: string
  generatesResources: number           # How many K8s resources this creates
  steps: ResolutionStep[]              # Multi-step creation workflow
  sections: Section[]                  # UI grouping for steps
  demoPool: DemoResource[]             # Pre-populated sample resources
}
```

### Multi-Step Resolution Workflow

Each blueprint defines a sequence of **resolution steps** — resources to select or create in order. Steps form a hierarchy (level 0 → level N):

```
ResolutionStep {
  level: number              # Depth in the resource hierarchy (0 = root)
  resourceKind: string       # e.g. "PKIEngine", "IntermediateCA", "Role"
  action: ResolutionAction   # select-or-create | create | reference
  shared: boolean            # Can be reused across connections
  autoGenerate: boolean      # System generates automatically (no user input)
  configFields: ConfigField[]  # Form schema for this step
}
```

**Resolution actions:**
| Action | Behavior |
|--------|----------|
| `select-or-create` | User picks an existing resource OR creates a new one (shared resources) |
| `create` | Always creates a new resource (unique per connection) |
| `reference` | Links to an existing resource without modification |

### Blueprint Catalog (6 Blueprints)

| Blueprint | Socket | Direction | Steps | Resources | Description |
|-----------|--------|-----------|-------|-----------|-------------|
| **Volume Mount** (`statefulset-pvc`) | storage | PVC → Workload | 1 | 1 | Mount persistent volume claim |
| **Config Mount** (`app-configmap`) | config | ConfigMap → Workload | 1 | 1 | Mount configuration data |
| **Secret Mount** (`app-k8ssecret`) | config | K8sSecret → Workload | 1 | 1 | Mount secret data |
| **Certificate Request** (`app-openbao-cert`) | cert | Secret → Workload | 5 | 5 | Full PKI chain (PKI Engine → Intermediate CA → Role → Certificate → Policy) |
| **OIDC Identity** (`keycloak-identity`) | identity | Keycloak → Workload | 4 | 4 | OIDC client credentials + delivery (Realm → Client → Scopes → Credential Delivery) |
| **Database Access** (`app-database`) | data | Workload → Database | 2 | 2 | Configure database access (Database → User) |

**OIDC Identity credential delivery model**: The `keycloak-identity` blueprint's 4th step (Credential Delivery) configures how OIDC credentials reach the workload. Public config (issuer URL, client ID, realm) is delivered via **ConfigMap**. The `client_secret` is delivered via **VSO** (Vault Secrets Operator): Keycloak → ll-operator → OpenBao → `VaultStaticSecret` → K8s Secret → workload env vars. The `vaultPath` field in the delivery step references the OpenBao path without requiring a visual Keycloak↔OpenBao connection in the graph — this is handled internally by the relation editor.

**Example — Certificate Request (5 steps):**

```
Level 0: PKI Engine        [select-or-create, shared]
Level 1: Intermediate CA   [select-or-create, shared]
Level 2: Role              [create]
Level 3: Certificate       [create]
Level 4: Policy            [create, autoGenerate]
```

Shared steps (PKI Engine, Intermediate CA) can be reused across multiple certificate connections. The Policy step is auto-generated without user input.

### Blueprint Resolution

`resolveBlueprint(sourceType, targetType, socketType)` matches blueprints bidirectionally:
- **Forward match**: source→target matches blueprint direction
- **Reverse match**: target→source matches (connection wired backwards)
- Returns `{ blueprint, reversed: boolean }` or `null`

**Data flow:**

```
Connection click  → openRelationEditor(connectionId)
                  → RelationEditorComponent reads source/target nodes
                  → resolveBlueprint() finds matching blueprint
                  → User walks through resolution steps (select-or-create / create)
                  → Each step generates a resource in the target hub's resource pool
                  → saveLinkConfig({ connectionId, blueprintId, resolvedSteps })
                  → Stored in EditorService.linkConfigs Map
```

**Context display**: The editor shows existing connections for both endpoints (excluding the current one), helping the user understand the node's relation context before configuring.

---

## Auto-Chain — Automatic Intermediate Node Creation

When a user drags a connection between two incompatible nodes that **share the same socket group**, the system can automatically create intermediate nodes to complete the path. This avoids requiring the user to manually place every node in a chain.

**Predefined chain sequences:**

| Chain | Group | Sequence | Terminal |
|-------|-------|----------|----------|
| **Network** | network | `internet` → `gateway` → `virtual-service` → `k8s-service` → ... | Any workload node |
| **Certificate** | cert | `secret` → ... | Any workload node |

**How it works:**

1. User attempts to connect e.g. `InternetNode` → `ApplicationNode` (fails layer 5)
2. `canMakeTypedConnection()` calls `findAutoChain(sourceType, targetType, socketGroup)`
3. `findAutoChain()` walks the chain sequence, finds both source and target positions
4. Returns `AutoChainResult` with `intermediateSteps[]` (nodes to create between source and target)
5. `createAutoChain()` positions intermediate nodes linearly between source and target
6. All connections are wired automatically: source → intermediate₁ → ... → intermediateₙ → target
7. Success notification: "Created N intermediate node(s)"

**Example**: Internet → Application triggers creation of Gateway + VirtualService + K8sService, positioned evenly between the two endpoints, all wired automatically.

**Design constraint**: Auto-chains only trigger when no direct compatibility exists AND a chain sequence covers both source and target for the given socket group. Unrecognized pairs simply fail with an error notification.

---

## Connection Filter UI

A filter panel displays live connection counts per group (6 groups). Selecting a group hides all nodes and connections that don't belong to that group using CSS `display:none` (no rete state mutation). This provides simplified topology views per concern (network, data, certificates, config, storage, identity).

---

## Properties Panel — Draft/Dirty Pattern

Node properties are edited via a **buffered draft** — changes are local until the user explicitly validates:

1. On node selection, `toConfig()` is snapshotted into a `panelDraft` signal
2. All edits update the draft only (node is unchanged)
3. **Validate** button applies draft to node (with confirmation modal)
4. **Delete** button removes node + all its connections (with confirmation modal)
5. Switching nodes or closing the panel while dirty prompts "abandon changes?"

This prevents accidental modifications to the infrastructure topology.

---

## Node Preview Modal — OCI Image Sub-Resources

Workload nodes (Application, Deployment, StatefulSet, Job, CronJob) support OCI Image sub-resources managed through a **sub-editor modal**. This follows the same UX pattern as the OpenBao/Keycloak sub-graph editors — a dedicated editing surface scoped to the owning workload, keeping container image configuration off the main graph canvas.

**Sub-editor modal (workload nodes):**
- Workload nodes open a **larger dialog** (92vw × 82vh, max 1200px) to accommodate the sub-editor experience
- The origin workload card is displayed at the top
- Below the origin card, OCI Image cards are rendered as sub-resource nodes
- Bottom orange pins on the workload card connect to OCI Image cards via vertical bezier curves
- The layout models a parent-child relationship: workload owns its container images

**Non-workload nodes** keep the standard smaller preview modal with an always-visible properties panel (no sub-editor pattern).

**Floating toolbar:**
- A floating toolbar (bottom-left, semi-transparent backdrop blur) replaces the previous inline "+ Image" button
- Shows available sub-resource types (currently: OCI Image)
- Extensible for future sub-resource types without layout changes

**Contextual right panel:**
- Hidden by default when the workload sub-editor modal opens
- Click the origin (workload) card → workload properties panel appears
- Click an OCI Image card → `SubGraphDetail` form appears (registry, tag, pull policy, etc.)
- Click elsewhere (empty canvas area) → panel hides
- This keeps the sub-editor canvas uncluttered while providing on-demand property editing

**Data storage:**
- OCI Image data is stored as `SubGraphResource` entries in `spec.subGraphs[workloadNodeId]` within the graph document
- Each workload node has its own sub-graph key, scoping images to their owning workload
- This follows the same sub-graph pattern used by hub nodes (OpenBao, Keycloak) but at the env level

**Registry:**
- `oci-image-subgraph.registry.ts` provides `getOciImageDefinition()` — the `SubGraphDefinition` describing OCI Image resource types, config schema, and hierarchy
- This registry follows the same pattern as `getOpenBaoDefinition()` / `getKeycloakDefinition()` for hub node sub-graphs

---

## Toolbar — Horizontal Inline with Dropdowns

The toolbar is a horizontal bar (top-right) with section headers arranged in a row. Each section header toggles a dropdown below it. Dropdowns **persist until the header is re-clicked** — adding items does not close the dropdown. This UX supports bulk node creation from the same category.

5 sections: Network, Workload, Database, Config, Storage. Centrer + Export buttons at the end. CA (OpenBao) and Identity (Keycloak) are excluded — these are system-managed bastion nodes not available for user placement. OCI Images are managed through the workload sub-editor modal, not the toolbar.

---

## Rete.js v2 Integration Patterns

| Pattern | Implementation |
|---------|---------------|
| Custom rendering | `AngularPlugin` with `AngularArea2D` presets |
| Node components | `CustomNodeComponent`, `InternetNodeComponent` registered via `customize` |
| Connection styling | `connectionStyles` Map populated during `connectioncreated` event, read by `CustomConnectionComponent` |
| Socket rendering | `CustomSocketComponent` with color derived from socket name via `SOCKET_TYPE_REGISTRY` |
| Connection validation | `ClassicFlow.canMakeConnection()` resolves socket type + checks both input/output cardinality via `checkCardinality()` |
| Pin cardinality | `BaseNode.pinCardinality` Map + `addTypedInput`/`addTypedOutput` sync cardinality with Rete's `multipleConnections` |
| Pseudo-connection cleanup | `dropPseudoConnection()` wrapper around `ConnectionPlugin.drop()` — forces cleanup when `canMakeConnection` rejects or removal guard blocks |
| Connection spread | `getConnectionSpread()` offsets bezier control points (c1y/c2y) for overlapping connections sharing same source/target |
| Connection configure | SVG pencil icon at bezier midpoint (t=0.5), CSS sibling hover, opens `RelationEditorComponent` overlay via `openRelationEditor()` |
| Pseudo-connections | `isPseudo` flag: no hit-area, no delete icon, non-interactive (system-managed links) |
| Hidden sockets | `BaseNode.hiddenSockets` getter hides pins visually (opacity:0) while keeping DOM anchor for rete |
| Filter/visibility | CSS `display:none` on `area.nodeViews` and `area.connectionViews` elements |
| Node removal | `removeNode()` cascades: removes all connections first, then the node |
| Custom modals | Signal-driven async modal (`Promise<boolean>`) replaces native `confirm()` to avoid blocking event loop drag bugs |
| Drag feedback | Colored box-shadow on dragged node matching its border color |

**Key Rete.js caveats**:
- `rete-angular-plugin`'s `ConnectionWrapperComponent` passes `data` and `path` as **separate** `@Input()` properties — not nested inside `data`
- Rete assigns properties **directly on component instances** (not via Angular bindings) — `ngOnChanges` is unreliable for connection props, use getters instead
- Blocking `confirm()` during `nodepicked` event freezes the event loop, preventing `pointerup` from clearing drag state — always use async modals

---

## Sub-Graph Editor (Resource Management)

Hub nodes (OpenBao, Keycloak) open a **modal mini graph editor** — the same rete.js node system as the main chart, scoped to that hub's API resources. Double-clicking a hub node opens the modal; closing it returns the user to the main graph. Since hub nodes are org-scoped, sub-graph saves route to the org-level API (`/api/organizations/:orgId/bastion-graph/subgraphs/:hubNodeId`) — changes propagate to all environments.

**Architecture decisions:**

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Type system | `ResourceNode` extends `ClassicPreset.Node` (NOT `BaseNode`) | Sub-graph nodes have different fields (resourceId, resourceKind, config, status, allowedChildren) — separate type hierarchy avoids polluting main editor nodes |
| Service isolation | `SubGraphEditorService` provided at **component level** (`providers: [SubGraphEditorService]`) | Each modal gets its own rete.js editor instance, destroyed on close. No shared singleton state between modals or with main editor |
| Socket model | Single `subGraphSocket` type + `allowedChildren` validation | Sub-graph connections model API resource hierarchy (parent→child), not infrastructure topology. Type-safety comes from `allowedChildren` arrays per resource type, not multiple socket types |
| Hub node | Immovable anchor, always present, cannot be deleted | Models the hub service itself (e.g., OpenBao). Position locked via area pipe intercepting `nodetranslate` |
| Resource schema | Registry pattern: `getOpenBaoDefinition()` / `getKeycloakDefinition()` | Each hub type has a `SubGraphDefinition` describing resource types, categories, config schemas, and hierarchy. Extensible to new hub types |

**Data flow:**

```
SubGraphResource[]     ──► loadResources()   ──► Rete.js nodes + connections
(persisted model)           (hub + recursive        (interactive editor)
                             tree walk)

Rete.js nodes          ──► exportToResources() ──► SubGraphResource[]
(after user edits)          (connection graph        (saved back)
                             traversal)
```

**Component structure:**

```
subgraph-modal/                  # Modal shell (backdrop, header, footer)
├── SubGraphModalComponent       # Orchestrates editor lifecycle
├── subgraph-node/               # Rete.js node renderer (icon, label, status dot)
│   └── SubGraphNodeComponent
├── subgraph-connection/         # Simplified connection (SVG path, colored, no configure button)
│   └── SubGraphConnectionComponent
├── subgraph-detail/             # Properties panel (slides in on node selection)
│   └── SubGraphDetailComponent
└── services/
    └── SubGraphEditorService    # Per-modal rete.js lifecycle (create, layout, export, destroy)
```

**Layout:** Left-to-right hierarchical auto-layout (COL_GAP=260px, ROW_GAP=110px). Hub node at depth 0, children at increasing depths. Orphan nodes positioned below the tree. `zoomAt()` fits all nodes in viewport after layout.

**Connection validation:** `canMakeConnection(from, to)` checks `sourceNode.allowedChildren.includes(targetNode.resourceKind)`. Same-side connections (output→output) are blocked. Hub node's `allowedChildren` lists top-level resource kinds (auto-computed: kinds not listed as children of any other type).

---

## Output Pin Color Resolution

Sub-graph nodes use `ColoredSocket` (extends `ClassicPreset.Socket`) to carry
per-pin colors independent of the socket type registry.

| Node type | Output pin color | Source |
|-----------|-----------------|--------|
| **Hub** (multi-section) | Section color from toolbar definition | `hubSections[].color` |
| **Regular** (single output) | First allowed child type's color | `outputColor` resolved from registry |
| **Leaf** (no children) | _No output pin_ | — |

The connection line inherits its color from the source output socket
(`setConnectionColor` reads `ColoredSocket.color`), ensuring visual consistency
from pin to connection to target node.
