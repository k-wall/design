<!--
PROPOSAL WORKFLOW:
1. Copy this template to: proposals/000-<descriptive-name>.md
2. Fill in your proposal content
3. Open a PR on GitHub
4. Rename the file to use your PR number: proposals/<PR#>-<descriptive-name>.md
   Example: git mv proposals/000-my-feature.md proposals/105-my-feature.md
5. Update the heading below to include your PR number: # <PR#> - <Title>
6. Push the rename and updated title to your PR

See proposals/README.md for complete instructions.
-->

# 142 - Kubernetes CRD Changes for Routing API

The [routing API proposal (070)](070-routing-api.md) introduces Routers as top-level plugins that
direct client requests across multiple Kafka clusters via named Routes. This proposal specifies the
CRD schema changes needed so that the Kroxylicious Kubernetes operator can configure routing
declaratively. It addresses [kroxylicious/kroxylicious#4430](https://github.com/kroxylicious/kroxylicious/issues/4430).

## Current situation

The current operator API ([proposal 001](001-kroxylicious-operator-api-v1alpha.md)) defines a
`VirtualKafkaCluster` (VKC) that links a proxy, a set of ingresses, an ordered filter chain, and
**exactly one** upstream target via `targetKafkaServiceRef`:

```yaml
apiVersion: kroxylicious.io/v1alpha1
kind: VirtualKafkaCluster
metadata:
  name: my-cluster
spec:
  proxyRef:
    name: my-proxy
  targetKafkaServiceRef:
    name: upstream-kafka
  ingresses:
    - ingressRef:
        name: my-ingress
  filterRefs:
    - group: kroxylicious.io
      kind: KafkaProtocolFilter
      name: my-filter
```

There is no concept of a router or multi-cluster routing in the CRD model. Each VKC maps to a
single `KafkaService`.

## Motivation

Routing API proposal 070 enables a single VKC to fan traffic across multiple upstream `KafkaService`
instances, with per-route filter chains and optional router chaining (forming a DAG). The CRDs must
express this structure so that the operator can generate the corresponding `RouterDefinition` and
`RouteDefinition` entries in the proxy configuration.

Specific capabilities that require CRD support:

- Associating a VKC with a router rather than a single `KafkaService`
- Declaring named routes within a router, each with its own filter chain and upstream target
- Supporting router chaining (a route's target can be another router)
- Stable node-ID mapping across restarts and YAML reordering
- Reporting cycles in the router DAG as conditions on the affected resources

## Proposal

### Out of scope

- **`KroxyliciousSidecarConfig`** — the sidecar injection admission webhook and its configuration
  resource are not extended to support router configuration. This is deferred to a separate proposal.

### New CRD: `KafkaProtocolRouter`

A `KafkaProtocolRouter` is a namespace-scoped resource that declares a router plugin and its
routes. It follows the same pattern as `KafkaProtocolFilter` (standalone, reusable, no
`proxyRef`). Multiple VKCs in the same namespace can reference the same router.

```yaml
apiVersion: kroxylicious.io/v1alpha1
kind: KafkaProtocolRouter
metadata:
  name: tenant-router
spec:
  type: io.kroxylicious.proxy.router.HeaderBasedRouter  # passed opaquely to the proxy; resolved by the proxy runtime (FQCN or unambiguous simple name)
  configTemplate:                                       # router-specific config; supports secret/configmap interpolation e.g. ${secret:my-secret:key}
    headerName: X-Tenant-Id
  routes:
    - name: tenant-a
      id: 0
      filterRefs:
        - group: kroxylicious.io
          kind: KafkaProtocolFilter
          name: tenant-a-encryption
      targetRef:
        group: kroxylicious.io
        kind: KafkaService
        name: kafka-cluster-a
    - name: tenant-b
      id: 1
      targetRef:
        group: kroxylicious.io
        kind: KafkaService
        name: kafka-cluster-b
```

#### Route fields

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Human-readable name; used as the route name in `RouteDefinition` |
| `id` | integer | yes | Stable zero-based identifier within this router; used in the node-ID mapping formula |
| `filterRefs` | list | no | Ordered list of `KafkaProtocolFilter` references applied on this route |
| `targetRef` | object | yes | Upstream target — either a `KafkaService` or a `KafkaProtocolRouter` |

#### Route `targetRef`

`targetRef` is a typed cross-namespace reference using `group`, `kind`, and `name`. Only two
`kind` values are valid:

- `KafkaService` — leaf target, maps to a `clusterDefinition` entry in the proxy config
- `KafkaProtocolRouter` — chains to another router, enabling nested routing DAGs

A CEL validation rule in the CRD schema enforces that `targetRef.kind` is one of these two values.

#### Route `id` and node-ID mapping

The routing API uses the formula:

```
virtualNodeId = route.id + (numberOfRoutes × targetBrokerNodeId)
```

Route `id` values MUST be:
- Non-negative integers
- Unique within the `KafkaProtocolRouter`
- Stable — reordering routes in the YAML MUST NOT change their `id` values

Implicit assignment (deriving `id` from list position) was considered and rejected; see
[Rejected alternatives](#rejected-alternatives).

Together these three constraints — non-negative, less than route count, and unique — guarantee that
`id` values form exactly the set `{0, 1, …, S-1}`, which is required for the node-ID mapping
formula to work correctly. They are enforced by CEL validation rules on the `routes` array in the
CRD schema:

```
# non-negative
self.routes.all(r, r.id >= 0)

# less than route count (no gaps)
self.routes.all(r, r.id < self.routes.size())

# unique within router
self.routes.all(r, self.routes.filter(s, s.id == r.id).size() == 1)
```

---

### Changes to `VirtualKafkaCluster`

#### New field: `targetRef`

A new optional field `targetRef` replaces the role of `targetKafkaServiceRef`. It accepts the same
discriminated union as a route's `targetRef` (`KafkaService` or `KafkaProtocolRouter`):

```yaml
spec:
  proxyRef:
    name: my-proxy
  targetRef:
    group: kroxylicious.io
    kind: KafkaProtocolRouter
    name: tenant-router
  ingresses:
    - ingressRef:
        name: my-ingress
  filterRefs:
    - group: kroxylicious.io
      kind: KafkaProtocolFilter
      name: global-audit-filter
```

#### Deprecation of `targetKafkaServiceRef`

`targetKafkaServiceRef` is deprecated. It continues to function — the operator treats it as
shorthand for `targetRef: {group: kroxylicious.io, kind: KafkaService, name: <same-name>}`.

When a VKC uses the deprecated field, the operator MUST set a condition on the VKC:

```
type:    DeprecatedField
status:  "True"
reason:  TargetKafkaServiceRefDeprecated
message: "spec.targetKafkaServiceRef is deprecated; migrate to spec.targetRef"
```

#### Validation

The mutual exclusion of `targetRef` and `targetKafkaServiceRef` is expressed as a `oneOf`
constraint in the CRD OpenAPI schema, requiring exactly one of the two fields to be present.

---

### Status conditions

#### On `KafkaProtocolRouter` — cycle detection

The operator resolves router references transitively when reconciling a `KafkaProxy`. If a cycle
is detected in the router DAG, the operator MUST set the following condition on **each
`KafkaProtocolRouter` that participates in the cycle**:

```
type:    ResolvedRefs
status:  "False"
reason:  RouterDAGCycle
message: "Router forms a cycle with: [router-b, router-c]"
```

Routers in a cycle are not deployed. The associated VKCs are also marked unready.

#### On `VirtualKafkaCluster` — unresolvable target

If the `targetRef` (or a router it references) cannot be resolved (not found, wrong kind, cycle),
the VKC MUST be marked:

```
type:    ResolvedRefs
status:  "False"
reason:  InvalidTarget          # or: RouterDAGCycle / TargetNotFound
message: "<human-readable explanation>"
```

---

### Complete example: multi-tenant routing

```yaml
# Upstream clusters
apiVersion: kroxylicious.io/v1alpha1
kind: KafkaService
metadata:
  name: kafka-cluster-a
spec:
  bootstrapServers: kafka-a.internal:9092
---
apiVersion: kroxylicious.io/v1alpha1
kind: KafkaService
metadata:
  name: kafka-cluster-b
spec:
  bootstrapServers: kafka-b.internal:9092
---
# Router with two routes
apiVersion: kroxylicious.io/v1alpha1
kind: KafkaProtocolRouter
metadata:
  name: tenant-router
spec:
  type: io.kroxylicious.proxy.router.HeaderBasedRouter
  configTemplate:
    headerName: X-Tenant-Id
  routes:
    - name: tenant-a
      id: 0
      targetRef:
        group: kroxylicious.io
        kind: KafkaService
        name: kafka-cluster-a
    - name: tenant-b
      id: 1
      targetRef:
        group: kroxylicious.io
        kind: KafkaService
        name: kafka-cluster-b
---
# VirtualKafkaCluster using the router
apiVersion: kroxylicious.io/v1alpha1
kind: VirtualKafkaCluster
metadata:
  name: multi-tenant
spec:
  proxyRef:
    name: my-proxy
  targetRef:
    group: kroxylicious.io
    kind: KafkaProtocolRouter
    name: tenant-router
  ingresses:
    - ingressRef:
        name: my-ingress
```

---

### Operator reconciliation sketch

The proposal covers CRD schema only; reconciler implementation is out of scope. At a high level,
reconciliation of a `KafkaProxy` must:

1. Resolve all `VirtualKafkaCluster` resources associated with the proxy
2. For each VKC, follow `targetRef` transitively, collecting the set of reachable
   `KafkaProtocolRouter` and `KafkaService` resources
3. Detect cycles in the resulting DAG and set conditions accordingly
4. Generate `routerDefinitions`, `clusterDefinitions`, and the VKC `target` in the proxy config
   from the resolved graph

## Affected/not affected projects

| Project | Affected |
|---|---|
| `kroxylicious-kubernetes/kroxylicious-kubernetes-api` | Yes — new `KafkaProtocolRouter` CRD; changes to `VirtualKafkaCluster` spec |
| `kroxylicious-kubernetes/kroxylicious-operator` | Yes — reconciler must resolve router refs, detect cycles, generate config |
| `kroxylicious-kubernetes/kroxylicious-admission` | No — out of scope; see [Out of scope](#out-of-scope) |
| `kroxylicious-api` | No — router SPI unchanged |
| `kroxylicious-runtime` | No — `RouterDefinition`/`RouteDefinition` model unchanged |
| `kroxylicious-filters` | No |
| `kroxylicious-kms` | No |
| `kroxylicious-docs` | Yes — operator user guide needs updating |

## Compatibility

### Backwards compatibility

Existing `VirtualKafkaCluster` resources using `targetKafkaServiceRef` continue to work without
change. The operator will emit a deprecation condition but will not break existing deployments.
No timeline for removing `targetKafkaServiceRef` is set by this proposal; a separate deprecation
notice will govern its removal.

### Forward compatibility

`KafkaProtocolRouter` is introduced at `v1alpha1`, consistent with the rest of the operator API.
Breaking changes to its schema require the usual alpha→beta→GA graduation process defined in
proposal 001.

Route `id` values are explicitly assigned to decouple virtual node-ID assignments from YAML
ordering. This means adding a new route at any position in the list does not invalidate
node-ID assignments for existing routes, provided existing route `id` values are not changed.
Changing a route's `id` or removing a route changes the effective `numberOfRoutes` value used
in the mapping formula and invalidates existing node-ID assignments — clients will need to
reconnect (the routing proposal describes this as requiring connection draining).

## Rejected alternatives

### Implicit route IDs derived from list position

Making `id` implicit (i.e., `id = index in routes list`) was considered for simplicity. It was
rejected because reordering routes in the YAML silently changes the node-ID mapping formula,
potentially reassigning virtual broker node IDs across restarts or configuration updates. This
would force client reconnections unexpectedly. Explicit `id` values make this change intentional
and reviewable in a diff.

### Inline router configuration in `VirtualKafkaCluster`

Embedding the router spec directly inside the VKC spec was considered. It was rejected because:

- A router cannot be shared across multiple VKCs
- The VKC spec becomes large and hard to review
- Consistency with `KafkaProtocolFilter` — which is a standalone CRD — argues for the same
  pattern for routers

### Replacing `targetKafkaServiceRef` with a discriminated `targetRef` only (no deprecation path)

Replacing the existing field outright (rather than deprecating it alongside the new field) would
be a breaking schema change for existing users. The deprecation-with-condition approach preserves
backward compatibility while clearly signalling the migration path.

### Separate `KafkaProxyRoute` CRD per route

Making each route a standalone CRD (referenced from `KafkaProtocolRouter`) was considered for
maximum RBAC granularity. It was rejected because routes have no meaningful independent lifecycle
— they are always created, updated, and deleted together with their router — and the additional
resource count adds operational overhead without proportionate benefit.
