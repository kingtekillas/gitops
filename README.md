# GitOps Platform Operations Guide

This repository manages a Kubernetes platform with Argo CD using an **app-of-apps** pattern. It deploys core platform services (ingress, certificates, monitoring), workloads (Nextcloud), and operational resources (backups + alerts).

## 1) Architecture overview

### Argo CD app-of-apps (`root-apps.yaml`)

`root-apps.yaml` is the bootstrap `Application` applied manually once. It points Argo CD at `applications/`, where each file defines a child `Application`.

```text
root-apps (argocd/Application)
└── path: applications/
    ├── cert-manager
    ├── local-issuers
    ├── ingress-nginx
    ├── monitoring-kube-prometheus-stack
    ├── monitoring-alerts
    ├── nextcloud
    └── nextcloud-backups
```

### Deployed components and namespaces

| Argo CD Application | Namespace | What it deploys |
|---|---|---|
| `cert-manager` | `cert-manager` | Jetstack cert-manager Helm chart + CRDs |
| `local-issuers` | `cert-manager` | ClusterIssuer manifests (`infrastructure/issuers/`) |
| `ingress-nginx` | `ingress-nginx` | NGINX ingress controller Helm chart |
| `monitoring-kube-prometheus-stack` | `monitoring` | Prometheus, Alertmanager, Grafana |
| `monitoring-alerts` | `monitoring` | Custom PrometheusRule alerts |
| `nextcloud` | `nextcloud` | Nextcloud Helm chart, MariaDB, Redis, ingress |
| `nextcloud-backups` | `nextcloud` | Backup PVC + CronJobs for DB/files |

---

## 2) Bootstrap order and sync-wave intent

### Intended order

1. **cert-manager** (`sync-wave: "0"`)
   - Installs cert-manager controller/CRDs first.
2. **issuers** (`local-issuers`, `sync-wave: "1"`)
   - Applies `ClusterIssuer` resources only after cert-manager exists.
3. **ingress controller** (`ingress-nginx`)
   - Should be synced after certificate stack is present so ingress TLS references can resolve cleanly.

### Practical sync-wave guidance

- Current repo already annotates:
  - `cert-manager`: wave `0`
  - `local-issuers`: wave `1`
- `ingress-nginx` currently has no explicit wave annotation. Operationally, set it to wave `2` if you want strict deterministic ordering during full bootstrap/reconcile events.

Suggested annotation:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "2"
```

---

## 3) Ingress controller decision: Traefik vs NGINX

### Current state in this repo

- The platform deploys **NGINX Ingress Controller** (`applications/ingress-nginx.yaml`) with controller class configured as `nginx` and marked default in Helm values.
- Some app values still declare ingress class as **`traefik`** (`apps/nextcloud/values.yaml`, `monitoring/kube-prometheus-stack/values.yaml`).

This mismatch can cause Ingress resources to be ignored (or picked by the wrong controller in mixed clusters).

### Recommended decision

Pick one controller and align all manifests.

- If using **NGINX** (recommended for this repo as currently bootstrapped):
  - Use `ingressClassName: nginx` / `className: nginx` in all workload values.
- If switching to **Traefik**:
  - Stop deploying `ingress-nginx` and set all ingresses to `traefik`.

### Expected ingress class values

| Controller choice | Ingress class value to use |
|---|---|
| NGINX | `nginx` |
| Traefik | `traefik` |

---

## 4) Secret management (no plaintext in Git)

### Problem to avoid

Do **not** commit live credentials directly in Helm values or manifests. This repo currently contains placeholder-style credential fields in app/monitoring values and backup jobs; treat these as examples only.

### Recommended provisioning patterns

Use one of these production-safe approaches:

1. **SOPS + age/GPG + Argo CD plugin**
   - Store encrypted YAML in Git.
   - Decrypt only in-cluster during sync.
2. **External Secrets Operator**
   - Keep secrets in Vault/AWS Secrets Manager/1Password/etc.
   - Sync into Kubernetes Secrets at runtime.
3. **Sealed Secrets**
   - Commit encrypted `SealedSecret` resources that only cluster controller can decrypt.

### Operational baseline

- Put secret references in chart values (`existingSecret`, envFrom secret refs, or mounted secret names).
- Keep cleartext out of:
  - `apps/*/values.yaml`
  - `monitoring/*/values.yaml`
  - `backups/*/*.yaml`
- Enforce via CI policy checks (for example: denylist patterns like `password:`, `token:`, `bot_token:` unless templated from secret refs).

---

## 5) Backup/restore and alerting notes

### Current backup setup (Nextcloud)

`nextcloud-backups` deploys:
- `nextcloud-backup-pvc` (`10Gi`, `local-path`)
- `nextcloud-db-backup` CronJob (every 6h, `mysqldump`)
- `nextcloud-files-backup` CronJob (every 6h offset, copies `/data`)

### Required variables and prerequisites

At minimum verify these before relying on backups:

- DB credentials available to backup job:
  - `MYSQL_PASSWORD` must match MariaDB user password.
- PVCs and storage classes exist:
  - `nextcloud-backup-pvc`
  - Nextcloud data PVC (`nextcloud-nextcloud` claim in file backup job)
  - `local-path` storage class on cluster
- Enough disk space for backup retention window.

### How to test restore path (runbook drill)

Run this periodically in non-production (or controlled maintenance window):

1. **Create an on-demand backup**
   - Trigger both CronJobs manually (`kubectl create job --from=cronjob/...`).
2. **Validate backup artifacts**
   - Confirm fresh `db-<timestamp>.sql` and `data-<timestamp>` in backup PVC.
3. **Restore DB to scratch instance**
   - Launch temporary MariaDB pod.
   - Import SQL dump.
   - Run validation query (`show tables;`, row counts).
4. **Restore files to scratch PVC/pod**
   - Copy a `data-<timestamp>` snapshot to temporary target.
   - Verify key app directories and file counts.
5. **Application-level verification**
   - Point a temporary Nextcloud instance at restored DB/data, or run targeted integrity checks.
6. **Record RTO/RPO and gaps**
   - Capture restore duration, missing dependencies, manual steps.

If restore drill is not repeatable in < runbook target time, adjust backup format/frequency/storage.

### Alerting setup notes

Current monitoring stack includes:
- kube-prometheus-stack with Alertmanager routing to Telegram receiver.
- Custom `LowDiskSpace` alert (`<15%` free on `/` for `5m`).

Before production cutover:
- Move Alertmanager credentials (`bot_token`, `chat_id`) to Kubernetes Secret flow (see Secret Management section).
- Add a synthetic alert test (for example a short-lived test rule) to validate notification path end-to-end.
- Verify on-call destinations and repeat intervals (`group_wait`, `group_interval`, `repeat_interval`) meet incident policy.

---

## Bootstrap quickstart

1. Install Argo CD in `argocd` namespace.
2. Apply root app:

```bash
kubectl apply -f root-apps.yaml
```

3. Watch child apps:

```bash
kubectl -n argocd get applications
kubectl -n argocd get applications -w
```

4. Validate core services:

```bash
kubectl get ns cert-manager ingress-nginx monitoring nextcloud
kubectl -n cert-manager get pods
kubectl -n ingress-nginx get pods
kubectl -n monitoring get pods
kubectl -n nextcloud get pods
```
