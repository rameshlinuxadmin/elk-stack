# ELK Stack on Kubernetes (ECK)

A self-hosted logging pipeline for Kubernetes, built on the **Elastic Cloud on Kubernetes (ECK) operator**, running locally on **Docker Desktop Kubernetes**. Collects container logs cluster-wide, ships them through Logstash, and makes them searchable in Kibana — enriched with Kubernetes metadata (pod, namespace, container name, etc.).

## Architecture

```
Kubernetes Pods (all namespaces)
        │  writes to /var/log/containers/*.log (symlinks → /var/lib/docker/containers)
        ▼
   Filebeat (DaemonSet, one pod per node)
        │  filestream input + add_kubernetes_metadata processor
        │  Beats protocol, port 5044
        ▼
     Logstash
        │  beats input → elasticsearch output (TLS)
        ▼
   Elasticsearch
        │  indices: filebeat-YYYY.MM.dd
        ▼
      Kibana
        │  Discover / Security → Explore → Hosts → Events
        ▼
    Searchable, filterable logs
```

## Components

| Component | Version | Purpose |
|---|---|---|
| ECK Operator | latest | Manages all Elastic custom resources |
| Elasticsearch | 9.1.2 | Storage and search (single node) |
| Logstash | 9.1.2 | Receives from Filebeat, forwards to Elasticsearch over TLS |
| Kibana | 9.1.2 | UI for querying and visualizing logs |
| Filebeat | 9.1.2 | DaemonSet, tails container logs on every node |

All resources are deployed in the `elastic-system` namespace, managed as ArgoCD Applications sourced from this repo: [rameshlinuxadmin/elk-stack](https://github.com/rameshlinuxadmin/elk-stack/).

## Prerequisites

- Kubernetes cluster (tested on Docker Desktop Kubernetes, single node, Docker runtime)
- ECK operator installed
- ArgoCD installed and configured with access to this repo

![ArgoCD](ELKimages/argocd.png)

## Deployment

All resources in this repo are managed via **ArgoCD (GitOps)**. Manifests are not applied manually with `kubectl` — changes are made in this repo and synced by ArgoCD.

Typical workflow:

```bash
# 1. Edit the manifest in this repo (e.g. filebeat.yaml)
# 2. Commit and push
git add filebeat.yaml
git commit -m "fix: enable filestream symlink following"
git push

# 3. ArgoCD syncs automatically (or manually trigger a sync)
Check sync/health status:

```bash
kubectl get applications -n argocd
```

Confirm pods are running after a sync:

```bash
kubectl get pods -n elastic-system -w
```

> **Note on applying fixes:** every fix described below (RBAC, filestream symlinks/fingerprinting, Logstash TLS) was originally diagnosed by editing manifests directly and applying with `kubectl apply` for a fast feedback loop, then committing the working version back into this repo for ArgoCD to track as the source of truth. If ArgoCD is set to auto-sync with self-heal enabled, manual `kubectl apply`/`edit` changes will be reverted on the next reconciliation — disable auto-sync temporarily or edit the repo directly when iterating.

## Filebeat RBAC

The ECK `Beat` CRD does **not** create a ServiceAccount automatically. `add_kubernetes_metadata` needs API access to list/watch pods and namespaces, so a dedicated ServiceAccount, ClusterRole, and ClusterRoleBinding are required (`filebeat-rbac.yaml`), and `serviceAccountName: filebeat` must be set on the DaemonSet pod spec.

> **Note:** on some setups the service account token is not automounted by default. If `add_kubernetes_metadata` silently produces no `kubernetes.*` fields, verify `automountServiceAccountToken: true` is explicitly set and that `/var/run/secrets/kubernetes.io/serviceaccount/token` exists inside the Filebeat container.

## Filestream input configuration

Reading `/var/log/containers/*.log` (which are symlinks into `/var/lib/docker/containers/` on Docker-runtime nodes) requires two non-default filestream settings:

```yaml
filebeat.inputs:
  - type: filestream
    id: kubernetes-container-logs
    paths:
      - /var/log/containers/*.log
    prospector.scanner.symlinks: true          # filestream does not follow symlinks by default
    prospector.scanner.fingerprint.enabled: false  # fingerprinting needs 1024+ byte files; disable for small/new logs
    file_identity.native: ~
```

Without `symlinks: true`, filestream never opens any files (0 harvesters, 0 events, no errors). Without disabling fingerprinting, small or newly created log files are skipped until they grow past 1024 bytes.

## Logstash → Elasticsearch TLS

ECK enables TLS on Elasticsearch by default. Logstash must connect over `https://` and trust the cluster's CA, mounted from the auto-generated `<cluster-name>-es-http-certs-public` secret:

```ruby
elasticsearch {
  hosts => ["https://elasticsearch-es-http.elastic-system.svc:9200"]
  user => "elastic"
  password => "${ELASTIC_PASSWORD}"
  ssl_enabled => true
  ssl_certificate_authorities => ["/usr/share/logstash/config/certs/ca.crt"]
  index => "filebeat-%{+YYYY.MM.dd}"
}
```

The Elasticsearch password is read from the operator-managed secret `elasticsearch-es-elastic-user`.

## Verifying the pipeline

**Check indices exist and are receiving documents:**

```bash
kubectl exec -n elastic-system elasticsearch-es-default-0 -- \
  curl -s -k -u elastic:$ELASTIC_PASSWORD "https://localhost:9200/_cat/indices/filebeat-*?v"
```

**Check Filebeat harvester activity:**

```bash
kubectl logs -n elastic-system -l beat.k8s.elastic.co/name=filebeat | grep -i harvester | tail -5
```

**Query in Kibana (Discover, data view `filebeat-*`):**

```
kubernetes.container.name : "envoy-gateway"
```
![Kibana logs](ELK/images/Kibana-logs.png)

## Known limitations

- **Single-node Elasticsearch** — indices show `yellow` health because the configured replica shard has no second node to be allocated to. Not a functional problem locally; set `number_of_replicas: 0` if you want a clean `green` status.
- **Daily indices, not data streams** — `filebeat-%{+YYYY.MM.dd}` is a classic rollover-free index pattern with the default ECS mapping template installed by Logstash. There is no ILM policy attached, so indices will accumulate indefinitely with no automatic retention or rollover.
- **No `dockercontainers` dependency assumption** — the `/var/lib/docker/containers` hostPath mount is only needed because this cluster uses the Docker runtime. On containerd/CRI-O clusters, `/var/log/pods` is sufficient and this mount can be dropped.
- **`hostNetwork: true` on Filebeat** — required for this setup; be aware it exposes the pod directly on the node's network namespace.

## Roadmap / not yet wired up

- ILM policy + rollover for `filebeat-*` indices
- Data stream migration (`logs-filebeat.generic-default`) for proper ECS field typing
- Metrics/APM collection alongside logs
- Alerting rules on error-level log patterns
- Documented integration between this stack, Jenkins, and Envoy Gateway deployed elsewhere in this cluster (currently deployed but not yet documented here)

## Related repos / stack

- **Jenkins** — CI (repo/location not yet documented here)
- **ArgoCD** — GitOps delivery for this repo and others in the cluster
- **Envoy Gateway** — ingress (repo/location not yet documented here)
