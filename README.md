# jenkins-pipeline

Jenkinsfile común para todos mis proyectos. Construye la imagen con **Kaniko** (sin Docker ni Podman), la sube a **GHCR** y la despliega en **PRE** con **Helm**.

Producción (PRO) queda fuera de este pipeline: se desplegará más adelante con Argo CD.

## Flujo

```text
Prepare  ->  Checkout  ->  Build and Push (Kaniko -> GHCR)  ->  Deploy PRE (helm upgrade)
```

1. **Prepare**: valida los parámetros y calcula imagen, carpeta y release.
2. **Checkout**: descarga la rama `BRANCH` de `GIT_URL`.
3. **Build and Push**: construye el `Dockerfile` de la carpeta de la app y sube `ghcr.io/juanfranvelilla/<imagen>:<BUILD_NUMBER>`.
4. **Deploy PRE** (si `DEPLOY_PRE` está activo): `helm upgrade --install` del chart `k8s/` con `image.tag=<BUILD_NUMBER>`. Si el despliegue no arranca en 5 minutos, Helm hace rollback automático (`--atomic`).

## Parámetros del job

Se definen en la configuración de cada job de Jenkins (*This project is parameterized*), **no** en el Jenkinsfile, para que cada job conserve sus valores por defecto.

| Parámetro    | Tipo    | Ejemplo                                                  | Descripción |
|--------------|---------|----------------------------------------------------------|-------------|
| `GIT_URL`    | String  | `https://github.com/juanFRANvelilla/finance-portfolio.git` | Repo del proyecto. Obligatorio. |
| `BRANCH`     | String  | `main`                                                   | Rama a construir y desplegar. Por defecto `main`. |
| `APP_NAME`   | String  | `backend`                                                | Subcarpeta con el `Dockerfile` y el chart `k8s/`. Obligatorio. |
| `BUILD_ROOT` | Boolean | `false`                                                  | `true` si el `Dockerfile` y `k8s/` están en la raíz del repo. |
| `DEPLOY_PRE` | Boolean | `true`                                                   | `false` para solo construir y subir la imagen. |

## Convenciones

Con `GIT_URL` y `APP_NAME` se calcula todo lo demás:

| Qué              | `BUILD_ROOT=false`                           | `BUILD_ROOT=true`           |
|------------------|----------------------------------------------|-----------------------------|
| Carpeta          | `<APP_NAME>/`                                | `./`                        |
| Imagen           | `ghcr.io/juanfranvelilla/<repo>-<APP_NAME>`  | `ghcr.io/juanfranvelilla/<repo>` |
| Tag              | `<BUILD_NUMBER>`                             | `<BUILD_NUMBER>`            |
| Release de Helm  | `<repo>-<APP_NAME>`                          | `<repo>`                    |
| Namespace        | campo `namespace` de `k8s/values.yaml`       | igual                       |

Ejemplo, finance-portfolio backend: imagen `ghcr.io/juanfranvelilla/finance-portfolio-backend:12`, release `finance-portfolio-backend`, namespace `pre-finance-portfolio-back`.

Solo se publica el tag numérico: no hay `latest` ni `dev`, así cada despliegue apunta a una imagen concreta.

### Namespaces

Uno por entorno y app, con el entorno delante:

```text
pre-<proyecto>-back    pre-<proyecto>-front
pro-<proyecto>-back    pro-<proyecto>-front
```

### Estructura esperada en cada proyecto

```text
<APP_NAME>/
  Dockerfile
  k8s/
    Chart.yaml
    values.yaml        # PRE
    values-pro.yaml    # PRO (más adelante, Argo CD)
    templates/
```

`values.yaml` debe tener al menos:

```yaml
namespace: pre-<proyecto>-<back|front>
registry: ghcr.io/juanfranvelilla/<imagen>
image:
  tag: ""   # lo sobrescribe Jenkins con el BUILD_NUMBER
```

Y las plantillas deben usar `{{ .Values.namespace }}` en `metadata.namespace` y la etiqueta `app.kubernetes.io/instance: {{ .Release.Name }}`.

## Requisitos en el clúster

Una sola vez, en el namespace `jenkins`:

- **Secret `regcred`** (`kubernetes.io/dockerconfigjson`) con acceso de escritura a `ghcr.io`. Kaniko lo usa para subir la imagen.
- **ServiceAccount `jenkins-deployer`**: es la identidad del pod de build para desplegar con Helm.
  ```bash
  kubectl create serviceaccount jenkins-deployer -n jenkins
  ```
- **Caché de Kaniko** en el nodo: `/home/juanfran/jenkins-cache/kaniko` (hostPath).

Por cada namespace de PRE (`pre-<proyecto>-<app>`):

```bash
NS=pre-finance-portfolio-back

kubectl create namespace $NS

# Permiso para que Jenkins despliegue en este namespace (y solo en este)
kubectl create rolebinding jenkins-deployer -n $NS \
  --clusterrole=edit \
  --serviceaccount=jenkins:jenkins-deployer

# Credenciales para que el clúster descargue la imagen de GHCR
kubectl create secret docker-registry ghcr-secret -n $NS \
  --docker-server=ghcr.io \
  --docker-username=juanfranvelilla \
  --docker-password=<PAT con read:packages>
```

Más los secrets propios de la app (por ejemplo `backend-secrets`).

## Crear un job nuevo

1. **New Item** → *Pipeline*.
2. Marcar *This project is parameterized* y añadir los parámetros de la tabla con los valores del proyecto.
3. **Pipeline** → *Pipeline script from SCM*:
   - Repository URL: la de este repo.
   - Credentials: `github-token-podio`.
   - Branch: `*/main`.
   - Script Path: `Jenkinsfile`.
4. Guardar y lanzar con **Build with Parameters**.
