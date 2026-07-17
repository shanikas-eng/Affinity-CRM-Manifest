# Argo CD manifests

**Purpose:** Have Argo CD track this git repo and sync the two Node-3-migrated stacks (`crm-new`, `crm-dev`) into the cluster.

**Assumption:** Argo CD is installed in a namespace called `argocd`. If not yet installed, see § "Installing Argo CD" below.

---

## Files

| File | What it does |
|---|---|
| `00-project.yaml` | `AppProject crm` — scopes Applications to only these two namespaces and only these resource kinds. Prevents an accidentally-misconfigured Application from touching cluster-scoped resources or other namespaces. |
| `10-app-crm-new.yaml` | `Application crm-new` — points at `k8s/crm-new/` in this repo → syncs Deployments + Services + HTTPRoute into namespace `crm-new`. |
| `11-app-crm-dev.yaml` | `Application crm-dev` — same, for the `docker-composer` (QA) stack. |

---

## What Argo does NOT manage (do NOT let it)

These stay out of Argo's control because losing them via a stray `prune` would be catastrophic:

- **Namespaces** (`00-namespace.yaml` files are `exclude`d in each Application `source.directory.exclude`)
- **Secrets** — `dockerhub` (Docker Hub PAT), `mariadb` (DB creds). Created via `kubectl create secret` per runbook § 3.3–3.4. Store the raw values in a password manager, **not** in git.
- **Gateway API `Gateway` + `Certificate` + `ClusterIssuer`** — Phase 1 resources, live in `gateway-system` / `cert-manager`, out of scope for this Argo project.

If later you want secrets in git, use [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) or [External Secrets Operator](https://external-secrets.io/) — do NOT commit raw `Secret` YAMLs.

---

## First-time bootstrap

1. **Fill in `repoURL` in both `10-app-crm-new.yaml` and `11-app-crm-dev.yaml`.**
   ```yaml
   source:
     repoURL: https://gitlab.com/affiniti/k8s-manifests.git   # ← your actual URL
     targetRevision: main
   ```

2. **Also fill in `sourceRepos` in `00-project.yaml`** to the same URL (currently `"*"` — tighten it once the repo URL is known).

3. **If repo is private, add repo creds to Argo:**
   ```bash
   kubectl -n argocd create secret generic repo-affiniti-k8s \
     --from-literal=type=git \
     --from-literal=url=https://gitlab.com/affiniti/k8s-manifests.git \
     --from-literal=username=<user> \
     --from-literal=password=<PAT>
   kubectl -n argocd label secret repo-affiniti-k8s \
     argocd.argoproj.io/secret-type=repository
   ```

4. **Prerequisites — namespaces + secrets must exist first** (from Phase 3 runbook § 3):
   ```bash
   kubectl apply -f k8s/crm-new/00-namespace.yaml
   kubectl apply -f k8s/crm-dev/00-namespace.yaml
   # + dockerhub secret + mariadb secret in each ns (see runbook)
   ```

5. **Apply the Argo objects:**
   ```bash
   kubectl apply -f k8s/argocd/00-project.yaml
   kubectl apply -f k8s/argocd/10-app-crm-new.yaml
   kubectl apply -f k8s/argocd/11-app-crm-dev.yaml
   ```

6. **First sync — manually** (safer than auto for first run):
   ```bash
   argocd app sync crm-new
   argocd app sync crm-dev
   # or from the UI: Applications → crm-new → SYNC
   ```

7. **Enable auto-sync** once the first manual sync is green — uncomment `automated:` in each Application manifest and re-apply.

---

## What happens after auto-sync is on

- Push to `main` → Argo detects drift within ~3 min → applies the change.
- Deployment image tags are **ignored** by Argo (`ignoreDifferences`). CI still runs `kubectl set image` (or updates git via a bot) to roll new images. This way a deploy pipeline that only bumps images doesn't fight Argo.
- If someone `kubectl delete deploy/pd-ms-auth-service`, Argo's `selfHeal` re-creates it. If someone `kubectl edit` changes replicas, Argo reverts it — desired state is git.

---

## Installing Argo CD (if not already installed)

Not part of Phase 3 — write this into `runbook/07-phase7-observability-ci.md` when scheduled. Sketch:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# Expose UI via HTTPRoute:
#   host: argocd.affiniti-kube.duckdns.org  (uses existing kube wildcard cert)
# Initial admin password:
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
```

---

## Gotchas

1. **`CreateNamespace=false`** in `syncOptions`. Namespace pre-created out-of-band. If we let Argo create it, a `prune` sweep can delete the namespace and take the secrets with it.
2. **`ServerSideApply=true`.** Handles field ownership cleanly when both CI (`kubectl set image`) and Argo touch the same Deployment.
3. **`ignoreDifferences` on `image`.** Without this, every CI image-tag push shows as `OutOfSync` in Argo and — with auto-sync — Argo would revert the pod to the git-listed image.
4. **The `crm` `AppProject` whitelist is deliberately narrow.** If you later add a resource kind to the stack (e.g. `HorizontalPodAutoscaler`, `NetworkPolicy`), you must also add it to `namespaceResourceWhitelist` — otherwise Argo will refuse to apply it.
