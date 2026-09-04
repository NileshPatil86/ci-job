# Voting App on AKS — Runbook

Operational guide for building, pushing, and deploying the voting app to AKS.

---

## 1. What this project is

A five-component demo voting app (the classic Docker "example voting app"), running on AKS with images built and served from Azure Container Registry.

| Component | Folder | Stack | Role |
|---|---|---|---|
| `vote` | `vote/` | Python 3.11 + Flask/gunicorn | Web page where you cast a vote |
| `result` | `result/` | Node 18 + Express + socket.io | Live results page |
| `worker` | `worker/` | .NET 7 | Background processor, no web UI |
| `redis` | *(public image)* | `redis:alpine` | Queue holding incoming votes |
| `db` | *(public image)* | `postgres:15-alpine` | Stores counted votes |

Only the **first three** are built by us and stored in ACR. `redis` and `db` use public images pulled straight from Docker Hub.

---

## 2. How the app and DB connect

### Data flow

```
  You ──vote──> [ vote ]
                   │  RPUSH to list "votes"
                   ▼
               [ redis ]
                   │  worker pops each vote
                   ▼
              [ worker ] ──INSERT/UPDATE──> [ db ]
                                               │  polled every second
                                               ▼
                                          [ result ] ──socket.io──> Your browser
```

- `vote` never talks to Postgres. It only pushes onto Redis.
- `result` never talks to Redis. It only reads Postgres.
- `worker` is the only component touching both — it is the bridge.

### How each service finds the database

Connection targets are **hardcoded in application source**, not injected as env vars. They work because Kubernetes Services named `db` and `redis` provide in-cluster DNS.

| Service | Connects to | Where it is set |
|---|---|---|
| `vote` | `Redis(host="redis", db=0)` | `vote/app.py` |
| `worker` | `Server=db;Username=postgres;Password=postgres;` | `worker/Program.cs` |
| `worker` | `redis` | `worker/Program.cs` |
| `result` | `postgres://postgres:postgres@db/postgres` | `result/server.js` |

Postgres credentials are set in `k8s-manifest/db-deployment.yaml`:

```yaml
env:
- name: POSTGRES_USER
  value: postgres
- name: POSTGRES_PASSWORD
  value: postgres
```

> **Important:** the hostnames `db` and `redis` must match the Service names in `k8s-manifest/`. Renaming a Service breaks the app with no config change to warn you.

---

## 3. Azure resources

| Thing | Value |
|---|---|
| Subscription | Ascendion Nitor |
| Resource group | `Nitor-Internal-POC` |
| AKS cluster | `nitor-dev-aks` |
| ACR | `vottingapp01.azurecr.io` (Standard SKU) |
| Namespace | `votting-app` |
| ACR repos | `vote`, `result`, `worker` |

---

## 4. Pipeline flow

### Workflow 1 — Build and push (`.github/workflows/app-ci-cd.yml`)

- **Triggers:** push to `main`, or manual run
- **Runs:** three parallel matrix legs, one per folder

Each leg:

1. Checks out the repo
2. Logs in to Azure via OIDC federated credentials
3. Logs in to ACR
4. Builds `<folder>/Dockerfile` with `<folder>` as build context
5. Pushes **two tags**: the git SHA and `latest`
6. Pulls the image back to prove the push worked

Result in ACR:

```
vottingapp01.azurecr.io/vote:<sha>     + :latest
vottingapp01.azurecr.io/result:<sha>   + :latest
vottingapp01.azurecr.io/worker:<sha>   + :latest
```

`fail-fast: false` — one broken Dockerfile will not cancel the other two builds.

### Workflow 2 — Deploy (`.github/workflows/deployment.yml`)

- **Trigger:** manual only, with an `image_tag` input (defaults to `latest`)

Steps in order:

1. Azure OIDC login
2. Fetch AKS credentials
3. Install kubectl
4. Rewrite the image tag in the three deployment manifests to the chosen tag
5. Create the namespace if missing
6. Ensure the `acr-pull-secret` pull secret is valid (see below)
7. `kubectl apply -f k8s-manifest/`
8. Wait for all five rollouts
9. Print final `deploy,svc,pods`

**Step 6 is self-healing and non-destructive.** It behaves like this:

- If the secret already in the cluster authenticates against ACR → **leave it untouched**, whatever `ACR_PULL_TOKEN_PASSWORD` says.
- If there is no working secret → fall back to `ACR_PULL_TOKEN_PASSWORD`, but only write it after confirming ACR accepts it.
- If neither works → **fail the deploy without writing anything.**

This matters: an earlier version overwrote the secret on every run, so one wrong GitHub secret silently broke image pulls for every newly created pod. The current version cannot do that.

### Required GitHub repo secrets

| Secret | Purpose |
|---|---|
| `AZURE_CLIENT_ID` | OIDC federated login |
| `AZURE_TENANT_ID` | OIDC federated login |
| `AZURE_SUBSCRIPTION_ID` | OIDC federated login |
| `ACR_PULL_TOKEN_PASSWORD` | Fallback password for the ACR pull token. Only used when the cluster has no working pull secret — e.g. a brand new namespace. |

---

## 5. How image pull auth works

The AKS kubelet identity has **no `AcrPull` role** on the registry, so pods cannot pull from ACR on their own identity. As a workaround, pods authenticate with a pull-only ACR token.

- **Scope map** `votting-app-pull` — `content/read` + `metadata/read` on `vote`, `result`, `worker` only. No push rights, no access to other repos.
- **Token** `votting-app-pull-token` — bound to that scope map.
- **Secret** `acr-pull-secret` in namespace `votting-app`, type `kubernetes.io/dockerconfigjson`.
- Each app deployment references it:

```yaml
imagePullSecrets:
- name: acr-pull-secret
```

> **The permanent fix — needs an admin.** Grant `AcrPull` to the kubelet identity. That removes the token, the GitHub secret, the workflow step, and this entire class of failure.
>
> ```
> az aks update -g Nitor-Internal-POC -n nitor-dev-aks --attach-acr vottingapp01
> ```
>
> A **Contributor cannot do this.** Creating a role assignment needs `Microsoft.Authorization/roleAssignments/write`, which only Owner or User Access Administrator holds. Attempting it as a Contributor fails with:
>
> ```
> (AuthorizationFailed) ... does not have authorization to perform action
> 'Microsoft.Authorization/roleAssignments/write' over scope ...
> ```
>
> Kubelet identity object ID to hand the admin: `66aee99c-0e2b-4aaa-92fb-13e3e0cb9d46`

---

## 6. First-time setup

1. **Add the four GitHub secrets** listed above.

2. **Create the ACR scope map and token** (only if they do not already exist):

   ```
   az acr scope-map create --name votting-app-pull --registry vottingapp01 \
     --repository vote   content/read metadata/read \
     --repository result content/read metadata/read \
     --repository worker content/read metadata/read

   az acr token create --name votting-app-pull-token \
     --registry vottingapp01 --scope-map votting-app-pull
   ```

3. **Copy the generated `password1`** into the GitHub secret `ACR_PULL_TOKEN_PASSWORD`. It is shown only at creation time.

4. **Run the build workflow** (push to `main` or trigger manually) and confirm all three images land in ACR:

   ```
   az acr repository show-tags -n vottingapp01 --repository vote -o table
   ```

5. **Run the deploy workflow** from the Actions tab.

---

## 7. Routine deploy

1. Merge your change to `main` — the build workflow runs automatically.
2. Confirm all three matrix legs are green.
3. Go to **Actions → Deploy to AKS (k8s-manifest) → Run workflow**.
4. Enter the image tag:
   - `latest` for the newest build, or
   - a specific git SHA to deploy or roll back to an exact commit.
5. Watch the rollout steps finish.

### Deploying manually from your laptop

```
az aks get-credentials -g Nitor-Internal-POC -n nitor-dev-aks
kubectl apply -f k8s-manifest/ -n votting-app
kubectl rollout status deployment/vote -n votting-app
```

---

## 8. Verify a deployment

```
# everything should read Running
kubectl get pods -n votting-app -o wide

# get the public URLs
kubectl get svc -n votting-app
```

- `vote` is exposed on port **5000**, `result` on port **5001**, both via `LoadBalancer`.
- Open `http://<vote EXTERNAL-IP>:5000` and cast a vote.
- Open `http://<result EXTERNAL-IP>:5001` — the tally should update within a second or two.
- If `EXTERNAL-IP` shows `<pending>`, Azure is still provisioning the load balancer. Wait a minute or two.

Confirm the pipeline end to end:

```
kubectl logs deployment/worker -n votting-app --tail=20
```

Expect `Connected to db`, `Found redis at …`, then `Processing vote for '…'` lines as votes come in.

---

## 9. Troubleshooting

### `ImagePullBackOff` — read the error text carefully

The wording tells you which of two different problems you have.

**Error says `failed to fetch anonymous token … 401`**

- Meaning: **no credentials were sent at all.** The kubelet fell back to an anonymous pull and the registry rejected it.
- Cause: the `imagePullSecrets` block is missing from the deployment, or the `acr-pull-secret` secret does not exist in the namespace.
- Check:

  ```
  kubectl get secret acr-pull-secret -n votting-app
  kubectl get deploy vote -n votting-app -o jsonpath='{.spec.template.spec.imagePullSecrets[*].name}'
  ```

**Error says `failed to fetch oauth token … 401`**

- Meaning: **credentials were sent but rejected.** The secret exists and is wired up, but holds the wrong password.
- Cause: usually the secret was overwritten by a deploy run where `ACR_PULL_TOKEN_PASSWORD` was stale or wrong.
- Fix: correct the GitHub secret, then re-run the deploy workflow. To repair the cluster immediately:

  ```
  kubectl create secret docker-registry acr-pull-secret \
    --namespace votting-app \
    --docker-server=vottingapp01.azurecr.io \
    --docker-username=votting-app-pull-token \
    --docker-password='<password1>' \
    --dry-run=client -o yaml | kubectl apply -f -

  kubectl delete pods -n votting-app --field-selector=status.phase=Pending
  ```

> **Watch out:** existing Running pods keep running with a broken secret, because their image is already pulled. Only *new* pods fail. A scale-up or node move can surface a credential problem that has been latent for hours.

### Test the token without deploying anything

```
az acr token show -n votting-app-pull-token -r vottingapp01 --query status
```

Returns `enabled` if the token itself is healthy — which means any 401 is about the *password*, not the token.

### Rotate the pull token

```
az acr token credential generate -n votting-app-pull-token -r vottingapp01 --password1
```

Then update the GitHub secret **and** the in-cluster secret with the new value.

### Pods stuck `Pending`

```
kubectl describe pod <pod-name> -n votting-app
```

Usually insufficient node CPU/memory. Reduce replicas or scale the node pool.

### App loads but votes never appear in results

Check `worker` first — it is the only link between Redis and Postgres.

```
kubectl logs deployment/worker -n votting-app --tail=50
kubectl get pods -n votting-app -l app=worker
```

---

## 10. Known limitations

These are acceptable for a POC but must be fixed before any real use.

- **Data is lost on restart.** `db` and `redis` use `emptyDir` volumes. Every vote disappears when a pod restarts or moves nodes. Fix with a PersistentVolumeClaim (`managed-csi`).
- **Postgres password is `postgres`, in plain text** in the manifest and hardcoded in two source files. Move to a Kubernetes Secret.
- **No health probes.** No `readinessProbe` or `livenessProbe` on any deployment, so traffic can reach a pod before it is ready, and a hung pod is never restarted.
- **`worker` is a single replica** with no probes — if it wedges, votes queue in Redis and results silently stop updating.
- **Pull auth is a shared token,** not managed identity. Needs manual rotation. See the `AcrPull` note in section 5.
- **Services are `LoadBalancer` with no TLS.** Traffic is plain HTTP on a public IP. Put an ingress controller with a certificate in front before exposing anything real.
