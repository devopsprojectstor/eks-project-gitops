# GitOps with GitHub Actions + Argo CD

## How it works now

```
eks-project-app (code)          eks-project-gitops (desired state)       EKS cluster
──────────────────────          ──────────────────────────────────       ───────────
push to backend/ or frontend/
  │
  ▼
GitHub Actions
  1. test, build image
  2. push image to ECR
  3. commit new tag  ─────────────▶  apps/two-tier/kustomization.yaml
                                                   │
                                                   │  Argo CD polls every ~3 min
                                                   ▼
                                        Argo CD (inside cluster) ──▶ applies it
```

- **GitHub Actions = CI.** It builds and pushes images. It has no access to the cluster at all.
- **Argo CD = CD.** It runs inside the cluster and *pulls* changes from git.
- **The gitops repo is the source of truth.** Its git history is your deployment history.

Three repos:

| Repo | Holds | Changed by |
|---|---|---|
| `eks-project` | Terraform, Argo CD install, the Argo `Application` | you |
| `eks-project-app` | App code, Dockerfiles, build workflows | you |
| `eks-project-gitops` | Kubernetes manifests + image tags | Actions (tags), you (everything else) |

---

## Step 1: Create the gitops repo

On GitHub, create **`eks-project-gitops`** as **public** and empty.
It holds no secrets, and public means Argo CD needs no credentials to read it.

```bash
cd eks-project-gitops        # the folder from the zip
git init && git add . && git commit -m "Initial manifests"
git branch -M main
git remote add origin https://github.com/renusingh123/eks-project-gitops.git
git push -u origin main
```

## Step 2: Let the app repo push to the gitops repo (deploy key)

The app workflows need write access to the gitops repo, and only that repo.
A deploy key is an SSH key scoped to a single repo.

```bash
ssh-keygen -t ed25519 -C "gitops-bot" -f gitops_key -N ""
```

This makes two files:

- **`gitops_key.pub` (public key):** go to `eks-project-gitops` > Settings > **Deploy keys** > Add deploy key.
  Title it `app-ci`, paste the file contents, and **tick "Allow write access"**.
- **`gitops_key` (private key):** go to `eks-project-app` > Settings > Secrets and variables > Actions.
  - Secret `GITOPS_DEPLOY_KEY` = the whole file, including the `BEGIN` and `END` lines.
  - Variable `GITOPS_REPO` = `renusingh123/eks-project-gitops`

Then delete both key files from your laptop: `rm gitops_key gitops_key.pub`

## Step 3: Update the infra repo

Copy the files from the `eks-project` folder in the zip over your infra repo:

| File | Change |
|---|---|
| `platform/argocd/application.yaml` | **new.** Tells Argo CD to watch the gitops repo |
| `.github/workflows/terraform.yml` | installs Argo CD v3.4.9 and registers the app after apply |
| `.github/workflows/destroy.yml` | deletes the Argo app first, so it can't recreate the load balancer |
| `eks/main.tf` | removes the app pipeline's cluster access (keeps your `@*` OIDC fix) |
| `eks/outputs.tf` | drops `EKS_CLUSTER` from the app settings, which no longer needs it |

`eks/terraform.tfvars` is unchanged, so your `admin_principal_arns` (IAM user + root) stay.
If your `eks/main.tf` has other edits of your own, diff before overwriting.

```bash
git add . && git commit -m "Add Argo CD" && git push
```

This runs the Terraform workflow, ~15–20 min if the cluster is down, ~3 min if it's up.
Check that the steps **Install Argo CD** and **Register the app with Argo CD** go green.

## Step 4: Update the app repo

```bash
cd eks-project-app
git rm .github/workflows/deploy-backend.yml .github/workflows/deploy-frontend.yml
git rm -r k8s                                  # manifests now live in the gitops repo
cp <zip>/eks-project-app/.github/workflows/*.yml .github/workflows/
ls -a .github/workflows                        # build-backend.yml, build-frontend.yml, ci.yml
git add . && git commit -m "Switch to GitOps with Argo CD" && git push
```

The old deploy workflows must go. The app role has lost its cluster access, so they would fail.
You can also delete the `EKS_CLUSTER` variable from this repo now.

## Step 5: First deployment

Pushing in step 4 may not touch `backend/` or `frontend/`, so start both builds by hand:
Actions > **Build backend** > Run workflow, then **Build frontend** > Run workflow.

Each run ends by committing a new tag to the gitops repo. Check its commits; you'll see
`backend: deploy a1b2c3d` and `frontend: deploy a1b2c3d`.

Until both have run, pods show `ImagePullBackOff` (the tag is still `not-built-yet`).
That's expected. Argo CD fixes it on its own once the tags arrive.

## Step 6: Open the Argo CD UI

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo

kubectl port-forward svc/argocd-server -n argocd 8081:443
```

Open **https://localhost:8081** and accept the self-signed certificate warning.
Log in as `admin` with the password above.

You should see the **two-tier** app: Synced and Healthy, with a tree of Deployments, ReplicaSets, Pods and Services.

The UI is reached only through port-forward, with no public load balancer. That's safer and costs nothing.

App URL, as before:

```bash
kubectl get svc frontend -n two-tier     # http://<EXTERNAL-IP>
```

---

## Things to try (good interview demos)

**Deploy a change:** edit `frontend/index.html`, then push. Build frontend runs, and Argo CD syncs within ~3 min.
Click **Refresh** in the UI to sync immediately.

**Self-heal:**
```bash
kubectl scale deploy/frontend -n two-tier --replicas=5
```
Argo CD sees the drift from git and puts it back to 2.

**Change config through git:** in the gitops repo, set `replicas: 3` in `frontend.yaml`, then commit and push. Argo CD applies it.

**Rollback = git revert:**
```bash
cd eks-project-gitops && git pull
git log --oneline                 # find the bad "deploy" commit
git revert <commit> && git push   # Argo CD rolls back
```

**Prune:** delete a resource from the gitops repo. Argo CD deletes it from the cluster.

---

## Daily routine (unchanged)

- **Start:** infra repo > Actions > **Terraform** > Run workflow. Argo CD is installed and the app syncs on its own
  from the last tags in git, so there's **no need to re-run the builds** unless ECR was emptied.
- **Stop:** infra repo > Actions > **Destroy** > `two-tier-eks`.

One catch: Destroy deletes the ECR repos (`force_delete = true`), so after a rebuild the images
in the gitops tags are gone and pods show `ImagePullBackOff`. Re-run both build workflows once.
To avoid that, keep ECR outside the destroy. It costs only cents per month, so ask if you want that change.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Build: `Permission denied (publickey)` at "Check out gitops repo" | Deploy key missing or write access not ticked, or `GITOPS_DEPLOY_KEY` not the full private key |
| Build: `repository not found` | `GITOPS_REPO` variable wrong; it must be `owner/name`, not a URL |
| Argo app `Unknown` / `ComparisonError` | `repoURL` in `application.yaml` wrong, or gitops repo is private |
| Argo app `OutOfSync` and stays that way | Open the app in the UI > **App diff** / sync status message |
| Pods `ImagePullBackOff` | Tag in the gitops repo not in ECR yet; run the build workflow |
| Destroy: VPC `DependencyViolation` | The load balancer survived; delete it in EC2 > Load Balancers and re-run |
