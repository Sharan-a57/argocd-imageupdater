# ArgoCD Image Updater Guide

A comprehensive guide to setting up ArgoCD with Image Updater for automated container image updates in Kubernetes.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Running the Python Application](#running-the-python-application)
- [ArgoCD Installation](#argocd-installation)
- [Accessing ArgoCD](#accessing-argocd)
- [ArgoCD Image Updater Installation](#argocd-image-updater-installation)
- [Project Structure and Kustomize Setup](#project-structure-and-kustomize-setup)
- [Git Credentials Configuration](#git-credentials-configuration)
- [Application Configuration](#application-configuration)
- [How It Works](#how-it-works)
- [Accessing Your Application](#accessing-your-application)
- [Troubleshooting](#troubleshooting)

## Prerequisites

- Kubernetes cluster (Minikube, Kind, or production cluster)
- kubectl CLI installed and configured
- Git repository for storing manifests
- Container registry with your application images

## Running the Python Application

To run the Flask application locally:

```bash
venv/bin/python app.py
```

## ArgoCD Installation

Install ArgoCD in your Kubernetes cluster:

```bash
# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## Accessing ArgoCD

### Port Forwarding

Expose the ArgoCD server locally:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Access the UI at: `https://localhost:8080`

### Getting Admin Password

Retrieve the initial admin password:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 --decode && echo
```

Login credentials:
- **Username:** `admin`
- **Password:** (output from above command)

---

## ArgoCD Image Updater Installation

Install ArgoCD Image Updater to enable automatic image updates:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

### What is ArgoCD Image Updater?

ArgoCD Image Updater is an automation tool that monitors container registries for new image versions and automatically updates your ArgoCD applications when new images are available.

---

## Project Structure and Kustomize Setup

### Why Kustomize?

ArgoCD Image Updater **requires** either Kustomize or Helm to function with Git write-back. It does not support direct modification of plain YAML files in a Directory source type.

**Key Benefits:**
- Image Updater modifies only `kustomization.yaml`, keeping deployment files clean
- Kustomize overlays the image version at build time
- Separates configuration (deployment.yaml) from runtime state (current image)
- Maintains GitOps best practices

### Directory Structure

Organize your manifests as follows:

```
./k8s/
├── application.yaml      # ArgoCD Application definition
├── configmap.yaml        # Application configuration
├── deployment.yaml       # Kubernetes Deployment
├── kustomization.yaml    # Kustomize configuration
└── service.yaml          # Kubernetes Service
```

### Creating kustomization.yaml

The `kustomization.yaml` file tells Kustomize which resources to include:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
```

**What happens:**
- ArgoCD Image Updater will automatically add an `images:` section to this file
- This section specifies image overrides that Kustomize applies at build time
- Your original `deployment.yaml` remains unchanged in Git

**Example after Image Updater modifies it:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
images:
  - name: aantonn/flask-app
    newTag: v1.2
```

---

## Git Credentials Configuration

### Why Git Credentials are Needed

ArgoCD Image Updater needs Git credentials to:
1. Clone your repository
2. Modify `kustomization.yaml` with new image versions
3. Commit and push changes back to your Git repository

Without these credentials, Image Updater cannot persist changes to Git.

### Creating the Secret

Create a Kubernetes secret containing your Git credentials:

```bash
kubectl -n argocd create secret generic git-creds \
  --from-literal=username=your-github-username \
  --from-literal=password=your-personal-access-token
```

**Important Notes:**
- For GitHub, use a **Personal Access Token (PAT)**, not your account password
- The token needs `repo` permissions (read/write access to repositories)
- The secret must be created in the `argocd` namespace
- Secret name (`git-creds`) must match the annotation in `application.yaml`

### Generating a GitHub Personal Access Token

1. Go to GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Click "Generate new token"
3. Select scopes: `repo` (full control of private repositories)
4. Generate and copy the token
5. Use this token as the password in the secret

---

## Application Configuration

### The application.yaml File

The `application.yaml` defines an ArgoCD Application resource with Image Updater annotations:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: flask-app
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: flask-app=aantonn/flask-app
    argocd-image-updater.argoproj.io/flask-app.update-strategy: semver
    argocd-image-updater.argoproj.io/write-back-method: git:secret:argocd/git-creds
    argocd-image-updater.argoproj.io/write-back-target: kustomization
spec:
  project: default
  source:
    repoURL: https://github.com/Sharan-a57/argocd-imageupdater.git
    targetRevision: main
    path: k8s
    kustomize: {}
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### Understanding the Annotations

#### 1. `image-list`
```yaml
argocd-image-updater.argoproj.io/image-list: flask-app=aantonn/flask-app
```

**What it does:** Defines which container images to monitor for updates.

**Format:** `<alias>=<image-name>`
- `flask-app` is the alias (used to reference this image in other annotations)
- `aantonn/flask-app` is the actual image name in the registry

**Why it's needed:** Tells Image Updater which images to track in your container registry.

#### 2. `update-strategy`
```yaml
argocd-image-updater.argoproj.io/flask-app.update-strategy: semver
```

**What it does:** Defines how Image Updater selects new image versions.

**Available Strategies:**
- `semver` - Semantic versioning (v1.0.0 → v1.1.0 → v2.0.0)
  - Picks the highest version number
  - Best for versioned releases
- `newest-build` - Picks the most recently built/pushed image
  - Based on image creation timestamp
  - Not dependent on tag naming
- `digest` - Uses image digest/SHA
- `name` - Alphabetical/lexical tag ordering

**Why it's needed:** Without a strategy, Image Updater wouldn't know which version to choose when multiple tags exist.

#### 3. `write-back-method`
```yaml
argocd-image-updater.argoproj.io/write-back-method: git:secret:argocd/git-creds
```

**What it does:** Specifies how and where Image Updater should persist changes.

**Format:** `git:secret:<namespace>/<secret-name>`
- `git` - Write changes back to the Git repository
- `secret` - Use credentials from a Kubernetes secret
- `argocd/git-creds` - Namespace and name of the secret

**Why it's needed:** Enables GitOps workflow by committing image updates back to your repository.

**Alternative:** Without Git write-back, changes would only exist in the cluster and be lost on next sync.

#### 4. `write-back-target`
```yaml
argocd-image-updater.argoproj.io/write-back-target: kustomization
```

**What it does:** Specifies that updates should be written to `kustomization.yaml`.

**Why it's needed:**
- Tells Image Updater to modify the Kustomize `images:` section
- Keeps deployment files untouched
- Follows Kustomize best practices

### Understanding the Application Spec

#### Source Configuration
```yaml
source:
  repoURL: https://github.com/Sharan-a57/argocd-imageupdater.git
  targetRevision: main
  path: k8s
  kustomize: {}
```

- **repoURL:** Your Git repository containing the manifests
- **targetRevision:** Git branch to track (usually `main` or `master`)
- **path:** Directory within the repo containing manifests
- **kustomize:** Enables Kustomize processing (required for Image Updater)

#### Destination Configuration
```yaml
destination:
  server: https://kubernetes.default.svc
  namespace: dev
```

- **server:** Target Kubernetes cluster (default is the cluster where ArgoCD runs)
- **namespace:** Where your application pods will be deployed

#### Sync Policy
```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
  - CreateNamespace=true
```

- **automated:** Enables automatic syncing when Git changes are detected
- **prune:** Deletes resources removed from Git
- **selfHeal:** Corrects manual changes back to Git state
- **CreateNamespace:** Automatically creates the target namespace if it doesn't exist

### Deploying the Application

Apply the application configuration:

```bash
kubectl apply -f k8s/application.yaml
```

**What happens:**
1. Creates an ArgoCD Application resource in the `argocd` namespace
2. ArgoCD detects the new Application
3. ArgoCD syncs manifests from Git to the `dev` namespace
4. Image Updater starts monitoring the configured image

---

## How It Works

### The Complete Workflow

1. **Image Updater polls** your container registry (every 2 minutes by default)
2. **Detects new version** based on the update strategy (e.g., v1.2 → v1.3)
3. **Clones Git repository** using the provided credentials
4. **Modifies kustomization.yaml** by adding/updating the `images:` section:
   ```yaml
   images:
     - name: aantonn/flask-app
       newTag: v1.3
   ```
5. **Commits and pushes** the change to Git
6. **ArgoCD detects** the Git change
7. **Kustomize builds** manifests, applying the image override
8. **ArgoCD syncs** the updated manifests to the cluster
9. **Kubernetes rolls out** the new image version

### Why Deployment Files Stay Unchanged

Your `deployment.yaml` might show:
```yaml
image: aantonn/flask-app:v1.1
```

But Kustomize transforms it before deployment:
```yaml
image: aantonn/flask-app:v1.3  # Overridden by kustomization.yaml
```

**Benefits:**
- Clean separation of base configuration and runtime state
- Deployment files remain version-controlled and stable
- Image versions are managed declaratively
- Easy rollbacks via Git history

### Verifying It's Working

Check Image Updater logs:
```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater -f
```

Look for messages like:
```
Setting new image to aantonn/flask-app:v1.3
Successfully updated image
Committing 1 parameter update(s)
```

Check kustomization.yaml changes:
```bash
cat k8s/kustomization.yaml
```

You should see an `images:` section with the new tag.

---

## Accessing Your Application

Once deployed, access your application using one of these methods:

### Method 1: Minikube Service (Minikube only)
```bash
minikube service flask-app-service -n dev
```

### Method 2: Port Forwarding
```bash
kubectl port-forward svc/flask-app-service 5000:5000 -n dev
```

Then open `http://localhost:5000` in your browser.

---

## Troubleshooting

### Image Updater Not Detecting Applications

**Symptom:** Logs show `considering 0 annotated application(s)`

**Solutions:**
1. Ensure annotations are on the **Application resource**, not the Deployment
2. Verify the Application is in the `argocd` namespace
3. Check that `kustomize: {}` is specified in the Application source

### "Directory" Source Type Not Supported

**Symptom:** `skipping app of type 'Directory' because it's not of supported source type`

**Solution:** Add `kustomize: {}` to the Application source spec and create `kustomization.yaml`

### Git Push Failures

**Symptom:** `failed to push changes` in Image Updater logs

**Solutions:**
1. Verify Git credentials secret exists: `kubectl get secret git-creds -n argocd`
2. Ensure PAT has `repo` permissions
3. Check secret namespace matches annotation (`argocd`)

### Images Not Updating

**Symptoms:** New images available but not deployed

**Checklist:**
1. Verify update strategy matches your tagging scheme
2. Check Image Updater can access your registry (may need pull secrets)
3. Review Image Updater logs for errors
4. Ensure ArgoCD Application sync is not paused

### Viewing Logs

```bash
# Image Updater logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater -f

# ArgoCD Application Controller logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller -f
```

---

## Summary

This setup provides:
- ✅ Automated image updates from your container registry
- ✅ GitOps workflow with all changes persisted in Git
- ✅ Clean separation of configuration and runtime state
- ✅ Easy rollbacks via Git history
- ✅ Automatic deployment of new versions

For more information, visit:
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ArgoCD Image Updater Documentation](https://argocd-image-updater.readthedocs.io/)
