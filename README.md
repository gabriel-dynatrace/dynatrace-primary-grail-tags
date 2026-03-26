# Dynatrace Primary Grail Fields & Tags — Reference Guide

> **Sources:** [Primary Grail Fields](https://docs.dynatrace.com/docs/semantic-dictionary/tags/primary-fields) · [Global Field Reference](https://docs.dynatrace.com/docs/semantic-dictionary/fields) · [D1COE Metadata Enrichment](https://dt-rnd.atlassian.net/wiki/spaces/d1coe/pages/1246757730) · [D1COE Enrichment: Kubernetes](https://dt-rnd.atlassian.net/wiki/spaces/d1coe/pages/1229849653) · [D1COE Enrichment: OneAgent](https://dt-rnd.atlassian.net/wiki/spaces/d1coe/pages/1373569857)
> **Audience:** Customer Success Engineers, Services Consultants, Solution Architects

---

## Why This Matters — 2nd Gen vs 3rd Gen

In **2nd Gen**, observability signals (logs, spans, metrics) were tightly coupled to entities. Access control, alerting, and dashboards were built around **Management Zones** and entity-level **Classic Tags**.

In **3rd Gen**, every datapoint is treated independently. Each signal must carry its own metadata so it can be:

| Purpose | 3rd Gen Mechanism |
|---------|------------------|
| Grouped / partitioned | **Buckets** |
| Filtered / visualized | **Segments** |
| Access controlled | **IAM Policies** |
| Cost allocated | **DPS Cost Allocation** |

**Metadata enrichment is the foundation.** Without it, none of the above work correctly. Primary Grail Fields and Primary Grail Tags are the two mechanisms Dynatrace provides to get the right metadata onto every signal.

---

## Two Concepts: Fields vs Tags

### Primary Grail Fields

Pre-defined infrastructure attributes that Dynatrace **automatically enriches** on all telemetry signals — no additional configuration required. They propagate across metrics, logs, spans, events, and Smartscape topology.

> Think of them as **Dynatrace-managed** metadata. You get them for free when your infrastructure is monitored correctly.

**Examples:** `k8s.cluster.name`, `k8s.namespace.name`, `dt.host_group.id`, `aws.account.id`

### Primary Grail Tags

**Customer-defined** key-value attributes that you add to your infrastructure or workloads. They use the `primary_tags.` prefix and propagate across all signals the same way Primary Grail Fields do.

> Think of them as **your-managed** metadata. You define them, you own the naming convention.

**Examples:** `primary_tags.stage`, `primary_tags.team`, `primary_tags.application`

### Key Rule

> **Primary Grail Fields don't always align with how customers want to slice their data.** A host group like `onPrem_multi_staging` might host both `easytravel` and `hipstershop` — two teams sharing the same infrastructure. Primary Grail Fields can't separate them. Primary Grail Tags can.

---

## Primary Grail Fields Reference

These fields are automatically propagated across all signals when the corresponding infrastructure is monitored by Dynatrace.

| Field | Description | Example Value |
|-------|-------------|---------------|
| `k8s.cluster.name` | Kubernetes cluster name | `acme-prod10` |
| `k8s.namespace.name` | Kubernetes namespace name | `checkout`, `kube-system` |
| `dt.host_group.id` | OneAgent host group identifier | `onPrem_easytravel_prod` |
| `aws.account.id` | AWS account identifier | `123456789012` |
| `aws.region` | AWS geographical region | `us-east-1` |
| `azure.subscription` | Azure subscription ID | `27e9b03f-04d2-...` |
| `azure.resource.group` | Azure resource group | `demo-backend-rg` |
| `azure.location` | Azure geographical location | `westeurope` |
| `gcp.project.id` | GCP project identifier | `dynatrace-gcp-ext` |
| `gcp.region` | GCP geographical region | `europe-west3` |

### What "Primary" Means

Fields tagged `primary-field` in the Dynatrace semantic dictionary propagate across **every signal type**. Non-primary fields (e.g. `java.jar.file`) only exist on the signal type where they originate — a segment built on `java.jar.file` only filters spans, not logs or metrics.

> **Use Primary Grail Fields for IAM, Buckets, and Segments whenever possible.** They require zero enrichment effort and work across all data types.

---

## Special Fields — Not Primary Grail Tags, But Critical

These fields are not `primary_tags.*` prefixed, but are treated as special primary fields and are foundational to 3rd Gen configuration:

| Field | Purpose |
|-------|---------|
| `dt.security_context` | Data-level access control — scopes what telemetry a user/group can see |
| `dt.cost.costcenter` | Cost allocation by cost center |
| `dt.cost.product` | Cost allocation by product |
| `dt.owner` | Ownership — used in Ownership app (Classic model) |

These must be **explicitly set** during enrichment — they are not auto-populated. They should be part of every enrichment strategy.

---

## Enrichment Strategy — Decision Guide

Use this to decide which enrichment approach to recommend per technology:

```
Is the customer on Kubernetes?
  ├─ YES → Are k8s.cluster.name and k8s.namespace.name granular enough?
  │           ├─ YES → Rely on Primary Grail Fields (zero effort)
  │           └─ NO  → Use Namespace Annotations & Labels
  │                     └─ Need workload-level granularity?
  │                           └─ YES → Add Manual Pod Annotations
  │
  └─ NO → Is it OneAgent on dedicated infrastructure (1 app per host)?
              ├─ YES → Rely on Host Group as Primary Grail Field
              │         └─ Add dt.security_context + dt.cost.* + primary_tags.* at host level
              └─ NO  → Shared infrastructure (multiple apps per host)
                         └─ Set dt.security_context + primary_tags.* as environment
                            variables per process
```

---

## Configuration by Technology

### Kubernetes — Three Approaches

#### Approach 1: Primary Grail Fields (Zero Config)

`k8s.cluster.name` and `k8s.namespace.name` are automatically enriched by the Dynatrace Operator on all telemetry signals. No additional configuration needed.

**Best for:** Simple setups where namespace-level granularity is sufficient for IAM, Buckets, and Segments.

**Limitation:** Two fields only. Cannot differentiate workloads within the same namespace.

---

#### Approach 2: Namespace Annotations & Labels (Recommended)

Map existing Kubernetes namespace labels and annotations to Dynatrace enrichment fields using the Dynatrace Operator settings. This is the recommended approach for customers who already follow cloud-native tagging conventions.

**Configure in:** Dynatrace Settings → Cloud and Virtualization → Kubernetes → your cluster → Metadata enrichment

**How it works:** The Dynatrace Operator mutates pod definitions and adds the mapped metadata. Labels/annotations defined at the **namespace level** are applied to all pods in that namespace.

**Example namespace definition:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: checkout
  labels:
    app.kubernetes.io/name: checkout
    app.kubernetes.io/part-of: ecommerce
    environment: production
    team: platform-checkout
```

**Example mapping result on telemetry:**
```
primary_tags.name = checkout
primary_tags.part-of = ecommerce
primary_tags.environment = production
primary_tags.team = platform-checkout
```

> **Important:** New enrichment rules may take up to **45 minutes** to become effective — the Dynatrace Operator queries the Settings API once every 45 minutes. To apply immediately: `kubectl -n dynatrace rollout restart deployment dynatrace-operator`. After adding enrichment config, **restart the application pods** to pick up the new metadata.

**Limitation:** Namespace-level only — all workloads in a namespace get the same values.

---

#### Approach 3: Manual Pod Annotations (Maximum Granularity)

Add Dynatrace metadata directly as pod annotations in the workload YAML. This gives workload-level granularity, overriding namespace-level values.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broker-service
spec:
  template:
    metadata:
      annotations:
        metadata.dynatrace.com/dt.security_context: "ecommerce-brokerservice"
        metadata.dynatrace.com/dt.cost.costcenter: "platform-team"
        metadata.dynatrace.com/dt.cost.product: "checkout"
        metadata.dynatrace.com/primary_tags.stage: "production"
```

**Priority rule:** Manual Pod Annotations **override** Namespace Annotations & Labels when both are set. You can run both simultaneously — pod annotations win.

> **Limitation:** Manual Pod Annotations do **not** enrich Kubernetes Platform Metrics or Kubernetes Events. For those signal types, namespace-level enrichment is still needed. Use both approaches together.

---

### OneAgent — Dedicated Infrastructure

When a VM hosts a single application or belongs to one team, the **host group** is the primary enrichment signal. Add additional metadata at the host level.

**Navigate to:** Settings → Monitored technologies → OneAgent → your host

Set the following custom properties:

```
# Data access control
dt.security_context=easytravel

# Cost allocation
dt.cost.costcenter=easytravel
dt.cost.product=easytravel

# Custom primary grail tags
primary_tags.stage=prod
primary_tags.team=platform-easytravel
```

> Use the `primary_tags.<name>` naming convention — this ensures the field is treated as a primary grail tag and propagates correctly across signals.

**Signal restart behavior:**
- **Logs** — no application restart required; the OneAgent log module handles enrichment on its own restart
- **Spans** — requires a **restart of the monitored application** to pick up new metadata

---

### OneAgent — Shared Infrastructure

When multiple applications run on the same host (e.g. in Docker), host-level metadata alone is insufficient — every process needs its own enrichment values.

Set enrichment as **environment variables per container/process:**

```yaml
# docker-compose.yaml
services:
  broker-service:
    environment:
      - DT_CUSTOM_PROP=dt.security_context=ecommerce-broker;dt.cost.product=broker;primary_tags.team=checkout
  login-service:
    environment:
      - DT_CUSTOM_PROP=dt.security_context=ecommerce-login;dt.cost.product=login;primary_tags.team=identity
```

> This is the only way to assign **different** `dt.security_context` values to different processes on the same host.

---

### OpenTelemetry / Custom Instrumentation

Set tags via OTel resource attributes:

```bash
OTEL_RESOURCE_ATTRIBUTES="dt.security_context=ecommerce,primary_tags.team=platform,primary_tags.stage=prod,dt.cost.product=checkout"
```

Or in code:

```javascript
const { Resource } = require('@opentelemetry/resources');

const resource = new Resource({
  'dt.security_context': 'ecommerce',
  'primary_tags.team': 'platform',
  'primary_tags.stage': 'prod',
  'dt.cost.product': 'checkout',
});
```

---

## Use Cases

### 1. Segmentation

Segments are global filters applied across all Dynatrace apps. Build them on Primary Grail Fields for low maintenance — they auto-apply to all signal types.

**Navigate to:** Settings → Environment segmentation → New segment

**By host group:**
```
Filter: dt.host_group.id = "$host_group_name"
Applies to: All data
```

**By K8s cluster (pattern match for environment stage):**
```
Filter: k8s.cluster.name startsWith "PROD-"
Applies to: All data, Security Events
```

**By K8s namespace:**
```
Filter: k8s.namespace.name = "$namespace"
Applies to: All data
```

**By custom primary tag (team):**
```
Filter: primary_tags.team = "checkout"
Applies to: All data
```

**By dt.security_context (fine-grained):**
```
Filter: dt.security_context startsWith "team-xyz"
Applies to: Logs, Security Events
```

> Built-in segments for `dt.host_group.id`, `k8s.cluster.name`, and `k8s.namespace.name` already exist in Dynatrace. They apply to **All data** by default — if you need to scope them to Security Events only, create a custom segment with the same field.

---

### 2. IAM Policies — Data Access Control

Use Primary Grail Fields in IAM policy `WHERE` conditions to restrict which telemetry data a user or group can see.

**Read-only access to logs for a specific K8s cluster:**
```
ALLOW storage:logs:read
  WHERE storage:k8s.cluster.name = "acme-prod10";
```

**Read access to logs and spans scoped to a host group:**
```
ALLOW storage:logs:read, storage:spans:read
  WHERE storage:dt.host_group.id = "onPrem_easytravel_prod";
```

**Access scoped by security context (team-based):**
```
ALLOW storage:logs:read, storage:spans:read, storage:events:read
  WHERE storage:dt.security_context = "checkout-team";
```

**Wildcard match for a department (MATCH operator):**
```
ALLOW storage:logs:read
  WHERE storage:dt.host_group.id MATCH ("teamA-*");
```

**Multi-field: AWS account scoped metrics:**
```
ALLOW storage:metrics:read
  WHERE storage:aws.account.id = "123456789012";
```

> `dt.security_context` gives the most flexibility for access control. Primary Grail Fields are better for broad, deployment-topology-based scoping. Use both — fields for coarse access, `dt.security_context` for fine-grained team-level control.

---

### 3. Bucket Routing (OpenPipeline)

Route telemetry into dedicated Grail buckets based on Primary Grail Fields — this controls retention, cost, and access at a bucket level.

**Example matching condition in OpenPipeline:**
```
k8s.cluster.name = "acme-prod10"
```

```
dt.host_group.id startsWith "onPrem_easytravel"
```

```
primary_tags.stage = "production"
```

---

### 4. DQL Querying

Query using `primary_tags.*` fields the same way you query any other field:

```dql
fetch logs
| filter primary_tags.team == "checkout"
| summarize count(), by: {k8s.namespace.name, loglevel}
```

```dql
fetch spans
| filter k8s.cluster.name == "acme-prod10"
| filter dt.security_context == "checkout-team"
| summarize avg(duration), by: {service.name}
```

```dql
fetch metrics
| filter aws.account.id == "123456789012"
| filter aws.region == "us-east-1"
```

---

## Fields Comparison Table

| Field | Auto-Enriched? | Propagates Cross-Signal? | Use For |
|-------|---------------|--------------------------|---------|
| `k8s.cluster.name` | Yes (Operator) | Yes | IAM, Buckets, Segments |
| `k8s.namespace.name` | Yes (Operator) | Yes | IAM, Buckets, Segments |
| `dt.host_group.id` | Yes (OneAgent) | Yes | IAM, Buckets, Segments |
| `aws.account.id` | Yes (AWS connector) | Yes | IAM, Segments |
| `azure.subscription` | Yes (Azure connector) | Yes | IAM, Segments |
| `gcp.project.id` | Yes (GCP connector) | Yes | IAM, Segments |
| `primary_tags.*` | No — you configure it | Yes (once set) | Custom IAM, Segments, Buckets |
| `dt.security_context` | No — you set it | Yes (once set) | Fine-grained access control |
| `dt.cost.costcenter` | No — you set it | Yes (once set) | Cost allocation |
| `dt.cost.product` | No — you set it | Yes (once set) | Cost allocation |

---

## Gotchas & Common Pitfalls

### 1. Not all fields propagate cross-signal
Only `primary-field` tagged fields in the semantic dictionary propagate across all signal types. Fields like `java.jar.file` or `k8s.workload.name` are local to the signal where they originate.

> **Test cross-signal propagation before building IAM or Segment rules on a field.** Use the [Enrichment Overview Notebook](https://guu84124.apps.dynatrace.com/ui/document/v0/#share=cce784b4-98e3-4a2a-97e3-56bc9ad7501e) from D1COE to verify.

---

### 2. Entities are not automatically in segments
With 3rd Gen, Smartscape entities are NOT automatically included when filtering by Primary Grail Field in Segments. Until New Smartscape ships for all entity types, you must **explicitly add entity types** (Hosts, Processes, Services) via `Related entity` to your segment if needed.

---

### 3. Namespace Annotations & Labels only work at namespace scope
The K8s Operator enrichment via namespace labels/annotations applies to **all pods in the namespace uniformly**. It does not support per-workload differences. For workload-level granularity, you must use Manual Pod Annotations.

---

### 4. Manual Pod Annotations miss K8s Platform Metrics and Events
Pod-level annotations do not enrich Kubernetes Platform Metrics or Kubernetes Events. Use **both** Namespace Annotations and Manual Pod Annotations together — namespace handles platform metrics/events, pods handle workload-level signals.

---

### 5. Operator rule changes take up to 45 minutes
New or modified enrichment rules for the Dynatrace Operator (namespace labels/annotations) take up to 45 minutes to apply. Force an immediate update with:
```bash
kubectl -n dynatrace rollout restart deployment dynatrace-operator
```
Then restart the application pods to pick up the new metadata on spans.

---

### 6. `primary_tags.*` naming convention must be exact
When defining custom tags via environment variables or OneAgent custom properties, always use the `primary_tags.` prefix:

```
# Correct
primary_tags.stage=production

# Wrong — will not be treated as a primary grail tag
stage=production
```

---

### 7. `dt.security_context` is not auto-populated
Unlike Primary Grail Fields, `dt.security_context` must be **explicitly set** via OneAgent custom properties, K8s pod annotations, OTel attributes, or OpenPipeline processors. If it's missing, IAM policies scoped by `dt.security_context` will silently deny access (implicit deny with no match).

---

### 8. Classic Tags and Management Zones are not the same
Classic Tags are entity-level metadata — they do not attach to raw telemetry (logs, spans, metrics). They cannot be used in Grail IAM conditions, OpenPipeline routing, or DQL queries on raw data. They are still valid for entity-level use cases (classic dashboards, alerting profiles), but should not be relied on for 3rd Gen platform configuration.

---

### 9. ServiceNow CMDB integration (SGC) still requires Classic Tags until SGC v3
The Service Graph Connector v2 syncs Smartscape entities using Classic Tags and Management Zone IDs. Until SGC v3 ships (planned), any CMDB integration that requires Classic Tags must be maintained separately from your 3rd Gen IAM/Segment configuration.

---

## Enrichment Checklist — Per Technology

### Kubernetes
- [ ] Dynatrace Operator deployed — `k8s.cluster.name` and `k8s.namespace.name` auto-enriched
- [ ] Namespace labels/annotations mapped if additional granularity needed
- [ ] `dt.security_context` set per namespace (or per pod for workload-level)
- [ ] `dt.cost.costcenter` and `dt.cost.product` set
- [ ] `primary_tags.*` defined per your naming convention
- [ ] Application pods restarted after enrichment changes

### OneAgent (Dedicated)
- [ ] Host group defined with consistent naming convention
- [ ] `dt.security_context` set as host custom property
- [ ] `dt.cost.costcenter` and `dt.cost.product` set
- [ ] `primary_tags.*` set as host custom properties

### OneAgent (Shared / Multi-App)
- [ ] `dt.security_context` set as environment variable per process/container
- [ ] `dt.cost.costcenter` and `dt.cost.product` set per process
- [ ] `primary_tags.*` set per process via environment variable

### OpenTelemetry
- [ ] Resource attributes include `dt.security_context`
- [ ] Resource attributes include `dt.cost.costcenter` and `dt.cost.product`
- [ ] Resource attributes include `primary_tags.*` fields per convention

---

## Recommended Naming Conventions

Consistent naming is critical — IAM conditions and Segment filters depend on predictable values.

**Host groups (OneAgent):**
```
<provider>_<application>_<environment>
onPrem_easytravel_prod
aws_checkout_staging
```

**K8s namespaces:**
```
<application>-<component>
easytravel-frontend
checkout-api
```

**dt.security_context:**
```
<team>_<environment>_<sensitivity>
team-checkout_production_sensitive
team-platform_staging_standard
```

**primary_tags.*:**
```
primary_tags.stage = prod | staging | dev
primary_tags.team  = checkout | identity | platform
primary_tags.app   = easytravel | hipstershop | easytrade
```

> Agree on naming conventions **before** deploying. Changing values after IAM policies and Segments are built around them is painful — every downstream rule must be updated.

---

## Useful Links

| Resource | Link |
|----------|------|
| Primary Grail Fields reference | [docs.dynatrace.com](https://docs.dynatrace.com/docs/semantic-dictionary/tags/primary-fields) |
| Global field reference (semantic dictionary) | [docs.dynatrace.com](https://docs.dynatrace.com/docs/semantic-dictionary/fields) |
| K8s metadata & telemetry enrichment | [docs.dynatrace.com](https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/guides/metadata-automation/k8s-metadata-telemetry-enrichment) |
| Configure advanced permissions (security context) | [docs.dynatrace.com](https://docs.dynatrace.com/docs/platform/grail/organize-data/advanced-permission-setup) |
| Segments upgrade guide (MZ → Segments) | [docs.dynatrace.com](https://docs.dynatrace.com/docs/manage/segments/upgrade-guide-segments) |
| D1COE — Metadata Enrichment | [Confluence](https://dt-rnd.atlassian.net/wiki/spaces/d1coe/pages/1246757730) |
| D1COE — Enrichment: Kubernetes | [Confluence](https://dt-rnd.atlassian.net/wiki/spaces/d1coe/pages/1229849653) |
| D1COE — Enrichment: OneAgent | [Confluence](https://dt-rnd.atlassian.net/wiki/spaces/d1coe/pages/1373569857) |
| D1COE — Segment Security Data Best Practices | [Confluence](https://dt-rnd.atlassian.net/wiki/spaces/d1coe/pages/1953038337) |
