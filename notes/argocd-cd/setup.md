# Argo CD — Project, Repository, and Application (Configuration as Code)

Everything here is declarative. You write a YAML manifest, `kubectl apply` it, and Argo CD
picks it up. The equivalent `argocd` CLI command is shown next to each step so you can verify
your work, but the **manifest is the source of truth**.

Ready-to-edit manifests live in [`manifests/`](./manifests/):

| File                                                             | What it creates                          |
| ---------------------------------------------------------------- | ---------------------------------------- |
| [`01-project.yaml`](./manifests/01-project.yaml)                 | The Argo CD project (`AppProject`)       |
| [`02-repo-ssh-secret.yaml`](./manifests/02-repo-ssh-secret.yaml) | Repository connection — **SSH**          |
| [`03-repo-token-secret.yaml`](./manifests/03-repo-token-secret.yaml) | Repository connection — **token (HTTPS)** |
| [`04-application.yaml`](./manifests/04-application.yaml)         | The Argo CD application (`Application`)  |
| [`kustomization.yaml`](./manifests/kustomization.yaml)           | Applies all of the above in order        |

Installation is covered in [`README.md`](./README.md).

---

# The Order

```text
1. Project      AppProject      -> what is allowed
2. Repository   Secret          -> how Argo CD authenticates to Git
3. Application  Application     -> what to deploy, and where
```

Three rules that cause most failures:

1. **`AppProject`, the repository `Secret`, and `Application` all live in the `argocd` namespace** —
   not in the namespace you are deploying to.
2. **The repo URL must match the credential type.** `git@github.com:...` needs an SSH key.
   `https://github.com/...` needs a token. They are not interchangeable.
3. **The repo URL in the `Application` must appear in the project's `sourceRepos`** — character for character.

---

# Step 0 — Log In

```bash
argocd login <ARGOCD_SERVER>        # e.g. argocd login 165.227.254.154
argocd admin initial-password -n argocd
```

The CLI is only needed for verification. Applying manifests with `kubectl` is enough on its own.

---

# Step 1 — Create the Project

Do not use the built-in `default` project. It permits every repo, every cluster, and every
namespace, which defeats the point of having projects.

`manifests/01-project.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: development
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: Development environment applications

  # Repos applications in this project may deploy FROM
  sourceRepos:
    - git@github.com:my-org/my-repo.git
    - https://github.com/my-org/my-repo.git

  # Clusters and namespaces they may deploy TO
  destinations:
    - server: https://kubernetes.default.svc
      namespace: development

  # Cluster-scoped resources allowed (needed for CreateNamespace=true)
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace

  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
```

Apply and verify:

```bash
kubectl apply -f manifests/01-project.yaml

kubectl get appproject -n argocd
argocd proj get development
```

| Field                       | Meaning                                                              |
| --------------------------- | -------------------------------------------------------------------- |
| `sourceRepos`               | Allowed repository URLs. `'*'` allows any (lab only).                |
| `destinations`              | Allowed `server` + `namespace` pairs.                                 |
| `clusterResourceWhitelist`  | Cluster-scoped kinds the project may create.                          |
| `namespaceResourceWhitelist`| Namespaced kinds the project may create.                              |

`https://kubernetes.default.svc` means "the cluster Argo CD itself is running in".

---

# Step 2 — Connect the Repository

Argo CD discovers repository credentials by looking for Secrets in the `argocd` namespace
carrying this label:

```yaml
labels:
  argocd.argoproj.io/secret-type: repository
```

That label is what makes it a repository connection. Without it the Secret is ignored.

There are two ways to authenticate. Pick one per repository — you do not need both.

---

## 2A — SSH

### Generate the key pair

```bash
ssh-keygen -t ed25519 -C "argocd" -f ~/.ssh/argocd_ed25519 -N ""
```

```text
~/.ssh/argocd_ed25519       # PRIVATE key -> goes into the Argo CD Secret
~/.ssh/argocd_ed25519.pub   # PUBLIC key  -> goes into GitHub
```

Never swap these. The private key never leaves your side; the public key never goes into Argo CD.

### Add the public key as a deploy key

```bash
cat ~/.ssh/argocd_ed25519.pub
```

GitHub: repo → **Settings** → **Deploy keys** → **Add deploy key** → paste → **Add key**.
Leave *Allow write access* unticked; Argo CD only reads.

> GitLab: **Settings** → **Repository** → **Deploy keys**
> Bitbucket: **Repository settings** → **Access keys**

### The manifest

`manifests/02-repo-ssh-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: repo-my-repo-ssh
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  type: git
  name: my-repo-ssh
  url: git@github.com:my-org/my-repo.git
  project: development            # optional: scope to one project
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    REPLACE_WITH_YOUR_PRIVATE_KEY
    -----END OPENSSH PRIVATE KEY-----
```

Paste the whole key, `BEGIN` and `END` lines included, indented under the `|` block.

Build it from the key file instead of pasting by hand:

```bash
kubectl create secret generic repo-my-repo-ssh \
  -n argocd \
  --from-literal=type=git \
  --from-literal=url=git@github.com:my-org/my-repo.git \
  --from-literal=project=development \
  --from-file=sshPrivateKey=$HOME/.ssh/argocd_ed25519 \
  --dry-run=client -o yaml > manifests/02-repo-ssh-secret.yaml

# then add the label
kubectl label --local -f manifests/02-repo-ssh-secret.yaml \
  argocd.argoproj.io/secret-type=repository -o yaml
```

Apply:

```bash
kubectl apply -f manifests/02-repo-ssh-secret.yaml
```

If the Git host's SSH key is unknown to Argo CD, register it once:

```bash
ssh-keyscan github.com | argocd cert add-ssh --batch
```

The lab shortcut is `insecure: "true"` in the Secret. Do not do that outside a lab.

CLI equivalent, if you would rather not write the manifest:

```bash
argocd repo add git@github.com:my-org/my-repo.git \
  --ssh-private-key-path ~/.ssh/argocd_ed25519 \
  --project development
```

---

## 2B — Token (HTTPS)

### Create the token

GitHub: avatar → **Settings** → **Developer settings** → **Personal access tokens** →
**Tokens (classic)** → **Generate new token**

- **Note:** `argocd`
- **Expiration:** set one and write the date down — expiry is a common silent breakage
- **Scope:** `repo`

Copy it immediately; GitHub shows it once.

> Fine-grained token: **Contents → Read-only** on the repository.
> GitLab: **Settings** → **Access Tokens**, scope `read_repository`.

### The manifest

`manifests/03-repo-token-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: repo-my-repo-https
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  type: git
  name: my-repo-https
  url: https://github.com/my-org/my-repo.git
  project: development
  username: my-github-username
  password: REPLACE_WITH_YOUR_TOKEN     # the token goes HERE
```

The token goes in `password`. There is no `token` field, and putting it in `username` fails.

Apply:

```bash
kubectl apply -f manifests/03-repo-token-secret.yaml
```

CLI equivalent:

```bash
argocd repo add https://github.com/my-org/my-repo.git \
  --username my-github-username \
  --password <TOKEN> \
  --project development
```

---

## Verify the Connection

```bash
argocd repo list
```

```text
TYPE  NAME           REPO                                INSECURE  CONNECTION STATUS
git   my-repo-ssh    git@github.com:my-org/my-repo.git   false     Successful
```

`Successful` means Argo CD can read the repo. In the UI: **Settings** → **Repositories**,
green **Successful** badge.

```bash
kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=repository
```

---

## One Credential for a Whole Organization

To avoid one Secret per repo, use a **credential template**. Same shape, but the label is
`repo-creds` and the `url` is a **prefix** — any repository whose URL starts with it inherits
the credential.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: creds-my-org
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds     # <- repo-creds, not repository
type: Opaque
stringData:
  type: git
  url: git@github.com:my-org                       # prefix, matches every repo in the org
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```

---

# Step 3 — Create the Application

`manifests/04-application.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: development

  source:
    repoURL: git@github.com:my-org/my-repo.git    # SSH form  -> uses 02-repo-ssh-secret.yaml
    # repoURL: https://github.com/my-org/my-repo.git  # HTTPS form -> uses 03-repo-token-secret.yaml
    targetRevision: main
    path: k8s/overlays/dev

  destination:
    server: https://kubernetes.default.svc
    namespace: development

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

Apply and verify:

```bash
kubectl apply -f manifests/04-application.yaml

kubectl get application -n argocd
argocd app get my-app
```

> The manifest above is the minimum that works. The file on disk,
> [`manifests/04-application.yaml`](./manifests/04-application.yaml), is a full annotated
> reference: every field below plus Helm/Kustomize/directory/plugin rendering, the complete
> `syncOptions` list, `ignoreDifferences`, `managedNamespaceMetadata`, multi-source apps, and an
> `ApplicationSet` example — each one commented with what it does. Optional settings are commented
> out there, so the file still applies as-is.

## Field Reference

### Identity

| Field                  | Meaning                                                                 |
| ---------------------- | ------------------------------------------------------------------------ |
| `metadata.name`        | The application's name in the UI and CLI. Not the name of anything deployed. |
| `metadata.namespace`   | Must be `argocd`. Putting it in the target namespace is why the app never shows up. |
| `finalizers`           | Cascading delete — removing the Application removes what it deployed.    |
| `sync-wave` annotation | Orders this app against other apps. Lower syncs first.                   |

### Source — what to deploy

| Field                    | Meaning                                                              |
| ------------------------ | --------------------------------------------------------------------- |
| `source.repoURL`         | Must match the repository Secret's `url` **and** the project's `sourceRepos`, exactly. |
| `source.targetRevision`  | Branch (`main`), tag (`v1.4.2`), or commit SHA. A tag or SHA pins the deployment. |
| `source.path`            | Folder inside the repo. Relative to the root, case sensitive. `.` = root. |
| `source.helm`            | `valueFiles`, inline `values`, `parameters`, `releaseName`, `ignoreMissingValueFiles`. |
| `source.kustomize`       | `images` (tag overrides), `namePrefix`, `commonLabels`, `replicas`.    |
| `source.directory`       | `recurse`, `include`/`exclude` globs, jsonnet vars — for plain YAML folders. |
| `source.plugin`          | A custom renderer registered as a repo-server sidecar.                |
| `sources` (plural)       | Multiple repos in one app — e.g. a public chart plus your private values via `$ref`. Replaces `source`. |

### Destination — where it goes

| Field                     | Meaning                                                             |
| ------------------------- | -------------------------------------------------------------------- |
| `destination.server`      | Cluster API URL. `https://kubernetes.default.svc` = the local cluster. |
| `destination.name`        | Alternative to `server` — the cluster's registered name. Never set both. |
| `destination.namespace`   | Default namespace. A resource with its own `metadata.namespace` keeps it. |

### Sync policy — how it stays in sync

| Field                          | Meaning                                                        |
| ------------------------------ | --------------------------------------------------------------- |
| `syncPolicy.automated`         | Deploy on every Git change. Omit the whole block for manual sync. |
| `automated.prune`              | Delete cluster resources removed from Git.                       |
| `automated.selfHeal`           | Revert manual `kubectl` changes back to Git.                     |
| `automated.allowEmpty`         | Keep `false` — stops a rendering bug that yields zero resources from pruning the app away. |
| `retry.limit` / `backoff`      | Retry a failed sync with exponential backoff instead of getting stuck. |
| `managedNamespaceMetadata`     | Labels/annotations for the namespace `CreateNamespace=true` creates (mesh injection, pod security). |

### Sync options

| Option                          | Effect                                                          |
| ------------------------------- | ---------------------------------------------------------------- |
| `CreateNamespace=true`          | Create the destination namespace. Needs `Namespace` in the project's `clusterResourceWhitelist`. |
| `PrunePropagationPolicy=`       | `foreground` (wait for dependents), `background`, or `orphan`.    |
| `PruneLast=true`                | Prune only after everything else is synced and Healthy.           |
| `RespectIgnoreDifferences=true` | Apply `ignoreDifferences` during sync, not just comparison. Without it, sync overwrites the ignored fields. |
| `ApplyOutOfSyncOnly=true`       | Only re-apply resources that actually differ. Faster on large apps. |
| `ServerSideApply=true`          | Needed for very large CRDs; cleaner field ownership.              |
| `Validate=false`                | Skip dry-run validation — for CRs whose CRD lands in the same sync. |
| `Replace=true`                  | `kubectl replace` instead of `apply`. Disruptive; last resort for immutable fields. |
| `FailOnSharedResource=true`     | Fail instead of fighting another app over the same resource.      |

### Drift control and housekeeping

| Field                    | Meaning                                                              |
| ------------------------ | --------------------------------------------------------------------- |
| `ignoreDifferences`      | Fields another controller owns (HPA replicas, injected CA bundles). Without it the app flaps OutOfSync forever. |
| `revisionHistoryLimit`   | How many syncs stay available for `argocd app rollback`. Default 10.  |
| `info`                   | Key/value links shown on the app page — runbook, dashboard, owner.    |

Healthy output:

```text
Health Status:      Healthy
Sync Status:        Synced to main (a1b2c3d)
```

Day-to-day:

```bash
argocd app list
argocd app sync my-app          # only needed with manual sync
argocd app logs my-app
argocd app history my-app
argocd app rollback my-app <ID>
argocd app delete my-app
```

---

# Apply Everything at Once

```bash
cd notes/argocd-cd/manifests
kubectl apply -k .
```

Or file by file, in order:

```bash
kubectl apply -f 01-project.yaml
kubectl apply -f 02-repo-ssh-secret.yaml      # or 03-repo-token-secret.yaml
kubectl apply -f 04-application.yaml
```

The project must exist before the application, or the application is rejected.

---

# Keeping Secrets Out of Git

Steps 2A and 2B produce Secrets holding a private key or a token. `stringData` is **plain text**,
and Kubernetes Secrets are only base64-encoded, not encrypted. Committing those files as-is
publishes the credential.

The manifests in `manifests/` ship with `REPLACE_WITH_...` placeholders so they are safe to commit.
Fill them in locally and never commit the filled version, or use one of these:

| Approach              | How it works                                                                 |
| --------------------- | ----------------------------------------------------------------------------- |
| **Apply out of band** | Keep the real Secret on your machine, `kubectl apply` it, commit only the placeholder file. |
| **Sealed Secrets**    | `kubeseal` encrypts the Secret; only the in-cluster controller can decrypt it. The `SealedSecret` is safe to commit. |
| **SOPS + age/KMS**    | Encrypt the values in place; decrypt at apply time.                           |
| **External Secrets**  | The Secret is pulled from Vault / AWS Secrets Manager / etc. Git holds only a reference. |

A quick local guard:

```bash
echo 'manifests/*-secret.yaml' >> .gitignore
```

---

# Troubleshooting

| Symptom                                                       | Cause and fix                                                                        |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `repository not accessible` / `authentication required`        | URL and credential type do not match, the Secret is missing the `secret-type: repository` label, or the token expired. |
| `Permission denied (publickey)`                                | Public key was not added as a deploy key, or the wrong key went into `sshPrivateKey`. |
| `error creating SSH agent` / host key errors                   | Run `ssh-keyscan github.com \| argocd cert add-ssh --batch`.                          |
| `application repo ... is not permitted in project ...`         | `repoURL` is not in the project's `sourceRepos`. Compare the two strings exactly.     |
| `application destination ... is not permitted in project ...`  | Cluster/namespace is not in the project's `destinations`.                             |
| `Namespace ... is not permitted in project`                    | Add `Namespace` to `clusterResourceWhitelist`.                                        |
| `Unable to resolve 'HEAD' to a commit SHA`                     | Wrong `targetRevision`. Check `main` vs `master`.                                     |
| `app path does not exist`                                      | `path` is relative to the repo root and case sensitive.                               |
| App stays `OutOfSync`                                          | No `syncPolicy.automated`. Run `argocd app sync my-app`.                              |
| Changes revert on their own                                    | Expected with `selfHeal: true`. Change Git, not the cluster.                          |

```bash
kubectl describe application my-app -n argocd
kubectl logs -n argocd deploy/argocd-repo-server        # repo auth and fetch errors
kubectl logs -n argocd deploy/argocd-application-controller
kubectl get events -n development --sort-by=.lastTimestamp
```

---

# Reference

- Declarative setup: https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/
- Private repositories: https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#repositories
- Projects: https://argo-cd.readthedocs.io/en/stable/user-guide/projects/
- Application spec: https://argo-cd.readthedocs.io/en/stable/operator-manual/application.yaml
- Sync options: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/
