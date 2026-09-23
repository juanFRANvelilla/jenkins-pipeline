# jenkins-pipeline

Shared Jenkinsfile for all my projects. It builds the image with **Kaniko** (no Docker or Podman), pushes it to **GHCR** and deploys it to **PRE** with **Helm**.

Production (PRO) is out of scope for this pipeline: it will be deployed later with Argo CD.

## Flow

```text
Prepare  ->  Checkout  ->  Build and Push (Kaniko -> GHCR)  ->  Deploy PRE (helm upgrade)
```

1. **Prepare**: validates the parameters and works out the image, directory and release names.
2. **Checkout**: fetches branch `BRANCH` from `GIT_URL`.
3. **Build and Push**: builds the app's `Dockerfile` and pushes `ghcr.io/juanfranvelilla/<image>:<BUILD_NUMBER>`.
4. **Deploy PRE** (when `DEPLOY_PRE` is enabled): `helm upgrade --install` of the `k8s/` chart with `image.tag=<BUILD_NUMBER>`. If the rollout doesn't become ready within 5 minutes, Helm rolls back automatically (`--atomic`).

## Job parameters

They are defined in each Jenkins job's configuration (*This project is parameterized*), **not** in the Jenkinsfile, so every job keeps its own defaults.

| Parameter    | Type    | Example                                                    | Description |
|--------------|---------|------------------------------------------------------------|-------------|
| `GIT_URL`    | String  | `https://github.com/juanFRANvelilla/finance-portfolio.git` | Project repository. Required. |
| `BRANCH`     | String  | `main`                                                     | Branch to build and deploy. Defaults to `main`. |
| `APP_NAME`   | String  | `backend`                                                  | Subdirectory containing the `Dockerfile` and the `k8s/` chart. Required. |
| `BUILD_ROOT` | Boolean | `false`                                                    | `true` if the `Dockerfile` and `k8s/` live at the repository root. |
| `DEPLOY_PRE` | Boolean | `true`                                                     | `false` to only build and push the image. |

## Conventions

Everything else is derived from `GIT_URL` and `APP_NAME`:

| What           | `BUILD_ROOT=false`                          | `BUILD_ROOT=true`                |
|----------------|---------------------------------------------|----------------------------------|
| Directory      | `<APP_NAME>/`                               | `./`                             |
| Image          | `ghcr.io/juanfranvelilla/<repo>-<APP_NAME>` | `ghcr.io/juanfranvelilla/<repo>` |
| Tag            | `<BUILD_NUMBER>`                            | `<BUILD_NUMBER>`                 |
| Helm release   | `<repo>-<APP_NAME>`                         | `<repo>`                         |
| Namespace      | `namespace` field in `k8s/values.yaml`      | same                             |

Example, finance-portfolio backend: image `ghcr.io/juanfranvelilla/finance-portfolio-backend:12`, release `finance-portfolio-backend`, namespace `pre-finance-portfolio-back`.

Only the numeric tag is published: no `latest` or `dev`, so every deployment points to a specific image.

### Namespaces

One per environment and app, environment first:

```text
pre-<project>-back    pre-<project>-front
pro-<project>-back    pro-<project>-front
```

### Expected layout in each project

```text
<APP_NAME>/
  Dockerfile
  k8s/
    Chart.yaml
    values.yaml        # PRE
    values-pro.yaml    # PRO (later, Argo CD)
    templates/
```

`values.yaml` must contain at least:

```yaml
namespace: pre-<project>-<back|front>
registry: ghcr.io/juanfranvelilla/<image>
image:
  tag: ""   # overridden by Jenkins with the BUILD_NUMBER
```

Templates must use `{{ .Values.namespace }}` in `metadata.namespace` and the label `app.kubernetes.io/instance: {{ .Release.Name }}`.

## Cluster requirements

Once, in the `jenkins` namespace:

- **Secret `regcred`** (`kubernetes.io/dockerconfigjson`) with write access to `ghcr.io`. Kaniko uses it to push the image.
- **ServiceAccount `jenkins-deployer`**: the identity the build pod uses to deploy with Helm.
  ```bash
  kubectl create serviceaccount jenkins-deployer -n jenkins
  ```
- **Kaniko cache** on the node: `/home/juanfran/jenkins-cache/kaniko` (hostPath).

For each PRE namespace (`pre-<project>-<app>`):

```bash
NS=pre-finance-portfolio-back

kubectl create namespace $NS

# Allow Jenkins to deploy to this namespace (and only this one)
kubectl create rolebinding jenkins-deployer -n $NS \
  --clusterrole=edit \
  --serviceaccount=jenkins:jenkins-deployer

# Credentials for the cluster to pull the image from GHCR
kubectl create secret docker-registry ghcr-secret -n $NS \
  --docker-server=ghcr.io \
  --docker-username=juanfranvelilla \
  --docker-password=<PAT with read:packages>
```

Plus the app's own secrets (for example `backend-secrets`).

## Creating a new job

1. **New Item** → *Pipeline*.
2. Tick *This project is parameterized* and add the parameters from the table with the project's values.
3. **Pipeline** → *Pipeline script from SCM*:
   - Repository URL: this repository's URL.
   - Credentials: `github-personal-token`.
   - Branch: `*/main`.
   - Script Path: `Jenkinsfile`.
4. Save and run it with **Build with Parameters**.
