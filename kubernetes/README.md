# Deploying Conductor on Kubernetes

This guide covers deploying Conductor OSS on a Kubernetes cluster using plain manifests. An [OpenShift section](#openshift) at the bottom covers the small number of differences for that platform.

## Prerequisites

- A Kubernetes cluster (1.25+) with permission to create Namespaces, Deployments, StatefulSets, Services, ConfigMaps, Secrets, and PersistentVolumeClaims
- `kubectl` configured to talk to your cluster
- A default StorageClass that can provision `ReadWriteOnce` volumes (standard on most managed clusters and `kind`)

## Persistence backends

Conductor supports several persistence backends. **PostgreSQL is the recommended starting point** for Kubernetes deployments — it handles both workflow state and task queuing in a single service, which keeps the manifest simple and the operational footprint small.

| Backend | Persistence | Queuing | Notes |
|---|---|---|---|
| **PostgreSQL** ✓ | postgres | postgres (built-in) | Recommended. No Redis required. |
| Redis / Dynomite | Redis | Redis (built-in) | Good fit if Redis is already in your stack. |
| MySQL | MySQL | Redis (separate) | Requires a Redis deployment for task queues. |
| Cassandra | Cassandra | Redis (separate) | Suited for high-scale deployments. |

Tips for non-Postgres backends are in the [Other backends](#other-backends) section below.

## Quick start with PostgreSQL

`conductor.yaml` in this directory deploys a complete, working Conductor instance — server, UI, and an in-cluster PostgreSQL StatefulSet — in a single apply:

```shell
kubectl apply -f kubernetes/conductor.yaml
```

Wait for both pods to reach `Running`:

```shell
kubectl get pods -n conductor -w
```

Once ready, verify the server is healthy:

```shell
kubectl port-forward -n conductor svc/conductor-server 8080:8080 5000:5000
curl http://localhost:8080/health
# {"healthy":true,...}
```

The Conductor UI is available at [http://localhost:5000](http://localhost:5000).

## What the manifest creates

| Resource | Name | Purpose |
|---|---|---|
| Namespace | `conductor` | Isolates all resources |
| Secret | `conductor-postgres-secret` | PostgreSQL credentials |
| ConfigMap | `conductor-config` | Server `.properties` file |
| StatefulSet | `conductor-postgres` | PostgreSQL 16 with 10Gi PVC |
| Service | `conductor-postgres` | Internal DNS for the DB |
| Deployment | `conductor-server` | Conductor server + UI (combined image) |
| Service | `conductor-server` | Exposes API (8080) and UI (5000) |

The server image (`orkesio/orkes-conductor-community`) ships both the Conductor API and the UI in a single container — the API is served on port 8080 and the UI via nginx on port 5000.

## Configuration

Server configuration lives in the `conductor-config` ConfigMap as a `.properties` file. The Postgres key settings:

```properties
conductor.db.type=postgres          # activates Postgres for both persistence and queuing
conductor.indexing.enabled=false    # no Elasticsearch required
spring.datasource.url=jdbc:postgresql://conductor-postgres:5432/conductor
spring.datasource.username=conductor
spring.datasource.password=conductor
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=2
loadSample=true                     # loads a sample workflow on first boot
```

To change any property, edit the ConfigMap and restart the Deployment:

```shell
kubectl edit configmap conductor-config -n conductor
kubectl rollout restart deployment/conductor-server -n conductor
```

## Using an external PostgreSQL instance

Remove the `StatefulSet` and its `Service` from the manifest and update the ConfigMap's `spring.datasource.*` values to point at your external instance (RDS, Cloud SQL, Azure Database, etc.).

The database must exist before Conductor starts:

```sql
CREATE DATABASE conductor;
```

Conductor applies its own schema on first boot.

## Other backends

### Redis / Dynomite

Redis is a natural fit if it's already part of your infrastructure. Replace the Postgres StatefulSet and Service with a Redis deployment (or point at an external Redis), and update the ConfigMap:

```properties
conductor.db.type=redis_standalone
conductor.redis.hosts=<redis-service>:6379:us-east-1c
conductor.redis.workflowNamespacePrefix=conductor
conductor.redis.queueNamespacePrefix=conductor_queues
conductor.redis.queuesNonQuorumPort=6379
conductor.indexing.enabled=false
```

For Dynomite clusters, use `conductor.db.type=dynomite` and set `conductor.redis.hosts` to your Dynomite topology.

### MySQL

MySQL persistence requires a separate Redis deployment for task queuing. Add both a MySQL StatefulSet and a Redis StatefulSet (or use external services), then configure:

```properties
conductor.db.type=mysql
spring.datasource.url=jdbc:mysql://<mysql-service>:3306/conductor?useSSL=false&useUnicode=true&useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=UTC
spring.datasource.username=conductor
spring.datasource.password=conductor
conductor.queue.type=redis_standalone
conductor.redis.hosts=<redis-service>:6379:us-east-1c
conductor.indexing.enabled=false
```

### Cassandra

Cassandra is suited for high-throughput, horizontally-scaled deployments. Like MySQL, it requires Redis for task queuing:

```properties
conductor.db.type=cassandra
conductor.cassandra.cluster=<cassandra-service>
conductor.cassandra.keyspace=conductor
conductor.queue.type=redis_standalone
conductor.redis.hosts=<redis-service>:6379:us-east-1c
conductor.indexing.enabled=false
```

## Workflow search with OpenSearch

By default the guide sets `conductor.indexing.enabled=false`, which means the UI's workflow search and history features won't be available. If you need those, Conductor supports OpenSearch 2.x and 3.x as the indexing backend — it works alongside any persistence backend.

Add an OpenSearch StatefulSet and Service to your manifest:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: conductor-opensearch
  namespace: conductor
spec:
  serviceName: conductor-opensearch
  replicas: 1
  selector:
    matchLabels:
      app: conductor-opensearch
  template:
    metadata:
      labels:
        app: conductor-opensearch
    spec:
      initContainers:
        - name: sysctl
          image: busybox
          securityContext:
            privileged: true
          command: ["sysctl", "-w", "vm.max_map_count=262144"]
      containers:
        - name: opensearch
          image: opensearchproject/opensearch:2.15.0  # or 3.6.0 for v3
          env:
            - name: discovery.type
              value: single-node
            - name: OPENSEARCH_JAVA_OPTS
              value: "-Xms512m -Xmx512m"
            - name: DISABLE_SECURITY_PLUGIN
              value: "true"
          ports:
            - containerPort: 9200
          readinessProbe:
            httpGet:
              path: /_cluster/health
              port: 9200
            initialDelaySeconds: 20
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: conductor-opensearch
  namespace: conductor
spec:
  selector:
    app: conductor-opensearch
  ports:
    - port: 9200
      targetPort: 9200
```

Then update the ConfigMap to enable indexing. For OpenSearch 2.x:

```properties
conductor.indexing.enabled=true
conductor.indexing.type=opensearch2
conductor.opensearch.url=http://conductor-opensearch:9200
conductor.opensearch.indexPrefix=conductor
conductor.opensearch.indexReplicasCount=0
conductor.opensearch.clusterHealthColor=yellow
```

For OpenSearch 3.x, change only the type:

```properties
conductor.indexing.type=opensearch3
```

All other properties are identical between versions.

> **Note:** `clusterHealthColor=yellow` is required for single-node OpenSearch — a single-node cluster never reaches green status. For multi-node production clusters, use `green`.

> **OpenShift:** The `sysctl` init container requires a privileged SCC. Either grant it or set `vm.max_map_count=262144` at the node level via a `MachineConfig`.

## Production considerations

**Credentials** — Replace the plaintext `stringData` in the Secret with values from your secrets management system (Vault, AWS Secrets Manager, Sealed Secrets, etc.) before going to production.

**Replicas** — The Deployment defaults to 1 replica. Conductor server is stateless; scaling to 3+ replicas requires no additional configuration beyond `replicas: 3`.

**Resources** — The manifest sets no resource requests or limits. A production starting point:

```yaml
resources:
  requests:
    cpu: "1000m"
    memory: "1500Mi"
  limits:
    cpu: "2000m"
    memory: "2500Mi"
```

**Ingress** — The manifest omits an Ingress so it works out of the box with any cluster. Add an Ingress targeting `conductor-server` on port 5000 (UI) and 8080 (API) using your cluster's IngressClass.

**Image version** — Pin the image tag to a specific release in production. Check [Docker Hub](https://hub.docker.com/r/orkesio/orkes-conductor-community/tags) for available tags.

## OpenShift

The manifest works on OpenShift with two adjustments.

**1. Replace Ingress with Routes**

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: conductor-ui
  namespace: conductor
spec:
  to:
    kind: Service
    name: conductor-server
  port:
    targetPort: ui
  tls:
    termination: edge
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: conductor-api
  namespace: conductor
spec:
  to:
    kind: Service
    name: conductor-server
  port:
    targetPort: api
  tls:
    termination: edge
```

**2. SecurityContextConstraints**

The community image runs as **root (UID 0)**. The startup script launches nginx (which binds port 5000) and writes logs to `/app/logs/server.log` — both operations require root. This means the image is **incompatible with the `restricted` SCC** and with `runAsNonRoot: true`.

To run on OpenShift, grant the `anyuid` or `privileged` SCC to the service account in the `conductor` namespace:

```shell
oc adm policy add-scc-to-user anyuid -z default -n conductor
```

> **Do not** add a `securityContext: runAsNonRoot: true` block to the Conductor pod spec — it will prevent nginx from starting and the UI will be unavailable.

If the PostgreSQL pod fails to start due to volume permission issues, add `fsGroup: 999` to its pod spec (999 is the `postgres` GID in the official image).

> **Note on UI availability:** The UI is bundled in the community image and served by nginx on port 5000. If the UI Route returns "Application not available", the most common cause is nginx failing to start due to a restrictive security context — check the pod logs for `Permission denied` errors and verify the SCC grants above have been applied.

`oc` users: all `kubectl` commands in this guide work identically with `oc`.
