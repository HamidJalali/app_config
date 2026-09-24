# app_config

This repository contains the Kubernetes configuration used to deploy the application from the related `app_code` repository.

## Related repository

- `app_code`: Flask application source and Docker image definition
- `app_config`: deployment manifests, namespace setup, and image pull secret configuration

The deployment flow is designed so that the Jenkins pipeline in `app_code` builds and pushes a Docker image, then applies the Kubernetes manifests in this repository to run the app in a `demo` namespace.

## Repository contents

```text
.
├── README.md                      # Deployment documentation
├── image-pull-secret.yaml         # Kubernetes Docker registry secret
├── myapp-deployment.yaml          # Deployment definition for the application
├── myapp-service.yaml             # Service exposing the application inside the cluster
```

## Files

### `image-pull-secret.yaml`
Creates a Kubernetes secret of type `kubernetes.io/dockerconfigjson` using the value provided by the environment variable `DOCKER_CONFIG_SECRET_VALUE`.

This secret is used so the cluster can authenticate to the Docker registry and pull the image for the app.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: docker-registry-secret
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: ${DOCKER_CONFIG_SECRET_VALUE}
```

### `myapp-deployment.yaml`
Defines the application deployment in the `demo` namespace.

Key points:

- `replicas: 1`
- container image: `hjcontainer/myapp:${IMAGE_TAG}`
- container port: `8080`
- image pull secret: `docker-registry-secret`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp-deploy-label
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: hjcontainer/myapp:${IMAGE_TAG}
        ports:
        - containerPort: 8080
      imagePullSecrets:
      - name: docker-registry-secret
```

### `myapp-service.yaml`
Exposes the application internally through a Kubernetes `Service`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: demo
spec:
  selector:
    app: myapp
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```

## Deployment flow

1. The application image is built in the `app_code` repository.
2. The image is tagged using the version from `VERSION`.
3. The Jenkins pipeline logs in to the Docker registry.
4. The image is pushed to Docker Hub.
5. The Kubernetes manifests in this repository are applied to the target cluster.
6. The app runs in the `demo` namespace and is exposed through `myapp-service`.

## Namespace

All deployment resources are configured for the `demo` namespace.

## Prerequisites

Before applying these manifests, ensure:

- a Kubernetes cluster is available
- `kubectl` is configured to connect to the cluster
- the Docker registry secret is populated correctly
- the referenced image `hjcontainer/myapp:<version>` exists in the target registry

## Example usage

Apply the manifests with:

```bash
kubectl apply -f image-pull-secret.yaml
kubectl apply -f myapp-deployment.yaml
kubectl apply -f myapp-service.yaml
```

## Notes

This repository is intentionally lightweight and focused on deployment configuration. It assumes the application image and CI/CD pipeline are managed by the `app_code` repository.
