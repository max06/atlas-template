# atlas-template

A practical starting point for [ATLAS](https://github.com/max06/atlas)-based Kubernetes deployments with ArgoCD.

## What's included

- **ArgoCD** with a helmfile CMP (Config Management Plugin) sidecar, so ArgoCD can render helmfile-based repositories like ATLAS.
- **ATLAS ApplicationSet** that automatically discovers all deployments in this repository and creates ArgoCD Applications for each cluster/deployment pair.
- **Snapshot Review** workflow that shows rendered manifest diffs on every pull request.

## Getting started

1. **Use this template** to create your own repository (or fork it).

2. **Update `deployments/global.values.yaml`** — set `atlas.repoURL` to your repository URL.

3. **Review `deployments/in-cluster/`** — this matches ArgoCD's default cluster name for itself. Rename it if your cluster is registered under a different name.

4. **Render locally** to verify your setup:
   ```bash
   helmfile -f helmfile.yaml.gotmpl template
   ```

5. **Bootstrap** — apply the ArgoCD and ATLAS deployments manually for the first time:
   ```bash
   # Ensure your kubeconfig points to the target cluster
   helmfile -f helmfile.yaml.gotmpl apply \
     --selector bootstrap=true
   ```
   After this, ArgoCD takes over and manages all deployments via the ApplicationSet.

   Labels like `bootstrap` are defined per deployment in `deployment.yaml` and can be used to select any group of deployments. This is useful beyond initial setup — for example, you can label a one-time recovery job to restore a backup and apply it with a single selector.

6. **Push to main** to create the first snapshot baseline for PR reviews.

## Directory structure

```
helmfile.yaml.gotmpl                       # Entry point — references ATLAS remotely
deployments/
  global.values.yaml                       # Values for ALL clusters (git repo URL, etc.)
  apps/                                    # Global deployments (applied to every cluster)
  in-cluster/                                # ArgoCD's default self-cluster name
    cluster.values.yaml                    # Values for this cluster
    apps/
      argocd/
        deployment.yaml                    # ArgoCD + helmfile CMP sidecar
      atlas/
        deployment.yaml                    # ATLAS ApplicationSet for ArgoCD
templates/
  argocd/
    helmfile.yaml.gotmpl                   # ArgoCD Helm release with CMP config
  atlas-appset/
    helmfile.yaml.gotmpl                   # ApplicationSet that discovers deployments
```

## Adding deployments

Create a new directory under your cluster's `apps/` and add a `deployment.yaml`:

```bash
mkdir -p deployments/in-cluster/apps/my-app
```

```yaml
# deployments/in-cluster/apps/my-app/deployment.yaml
apps:
  - template: my-app
    namespace: my-namespace
settings:
  autosync: "true"
```

Then create the matching app template in `templates/my-app/helmfile.yaml.gotmpl`.

## SOPS / age encryption

ATLAS supports SOPS-encrypted value files (`*.values.sops.yaml`) at every hierarchy level. Secrets are encrypted per-key, so you can mix encrypted and plain values in the same file.

### Setting up age keys

You need **two separate age keys** — one for ArgoCD (cluster-side decryption) and one for CI (snapshot review):

1. **Generate keys:**
   ```bash
   # Key for ArgoCD — used at render time in the cluster
   age-keygen -o argocd-age-key.txt

   # Key for CI — used by the snapshot review workflow
   age-keygen -o ci-age-key.txt
   ```

2. **Add both public keys** as recipients in your `.sops.yaml`:
   ```yaml
   creation_rules:
     - path_regex: values\.sops\.yaml$
       age: >-
         age1...(argocd key)...,
         age1...(ci key)...
   ```

3. **Create the ArgoCD secret** on each cluster that needs to decrypt secrets:
   ```bash
   kubectl -n argocd create secret generic argocd-age-key \
     --from-file=keys.txt=argocd-age-key.txt
   ```
   The ArgoCD helmfile sidecar mounts this secret automatically and sets `SOPS_AGE_KEY_FILE`. The volume is marked `optional`, so ArgoCD starts fine without it — SOPS decryption simply fails at render time for encrypted files.

4. **Store the CI key** as a GitHub repository secret named `SOPS_AGE_KEY`, then uncomment the `secrets:` block in `.github/workflows/atlas-review.yml`.

### Encrypting values

```bash
# Create a new encrypted values file
sops --encrypt --in-place deployments/in-cluster/cluster.values.sops.yaml

# Edit an existing encrypted file
sops deployments/in-cluster/cluster.values.sops.yaml

# Re-encrypt after adding a new recipient
sops updatekeys deployments/in-cluster/cluster.values.sops.yaml
```

## CI / Snapshot Review

The included GitHub Actions workflow (`.github/workflows/atlas-review.yml`) automatically:

- **On push to main**: renders all manifests and stores a baseline snapshot.
- **On pull request**: renders manifests, diffs against the baseline, and posts a comment showing exactly what changed — organized by cluster and deployment.

If you use SOPS-encrypted values, see the [SOPS / age encryption](#sops--age-encryption) section above for key setup. Decrypted values are automatically redacted (`*REDACTED*`) in snapshot output — secrets are never exposed in PR comments.

## Learn more

See the [ATLAS documentation](https://github.com/max06/atlas) for the full reference on value inheritance, app template authoring, SOPS integration, and deployment automation.
