# kubernetes-genguardx

Kubernetes manifests for deploying GenGuardX with Kustomize.

The deployment architecture is based on the existing Corridor GenGuardX setup:

- `corridor-app`: primary API and web application
- `corridor-worker`: background worker process
- `corridor-jupyter`: Jupyter/JupyterHub-facing service
- shared persistent volumes for app data, uploads, notebooks, and databases
- a single ingress routing `/` to the app and `/jupyter` to Jupyter

This repo is cloud agnostic. It can be used on any Kubernetes cluster, including managed Kubernetes offerings such as GKE, EKS, and AKS, as long as the cluster provides:

- an ingress controller
- a TLS issuer or existing TLS secret
- a `ReadWriteMany`-capable storage class
- an image pull secret for the container registry you use

## Layout

```text
base/               Reusable application manifests
overlays/example/   Minimal deployable overlay with placeholder configuration
```

## Configure

Update these files before deployment:

- `overlays/base/kustomization.yaml`
  - set the namespace
  - set the application image
  - set the public hostname
- `overlays/base/configs/api_config.py`
  - set database, Redis, and product-specific application settings
- `overlays/base/configs/jupyter_server_config.py`
  - set Jupyter runtime settings if needed
- `overlays/base/configs/jupyterhub_config.py`
  - set API connectivity and any JupyterHub-specific settings

If your cluster uses a different RWX storage class, update the PVC patches in `overlays/base/kustomization.yaml`.

## Deploy

Create the image pull secret in the target namespace (if required):

```bash
kubectl create namespace genguardx
kubectl create secret docker-registry registry-secret \
  --docker-server=<registry-host> \
  --docker-username=<username> \
  --docker-password=<password> \
  --namespace genguardx
```

Render and apply the overlay:

```bash
kubectl apply -k overlays/base
```

Verify rollout:

```bash
kubectl get pods -n genguardx
kubectl get svc -n genguardx
kubectl get ingress -n genguardx
```

## Notes

- The base manifests assume the application image contains the `corridor-api`, `corridor-worker`, and `corridor-jupyter` entrypoints.
- The app deployment runs a database migration in an init container before the main API starts.
- The ingress manifest references `cert-manager.io/cluster-issuer: letsencrypt-prod`; adjust or remove that annotation if your cluster handles TLS differently.
