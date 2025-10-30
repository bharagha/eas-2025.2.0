# Deploy with Helm

Use Helm to deploy Smart Intersection to a Kubernetes cluster.
This guide will help you:

- Add the Helm chart repository.
- Configure the Helm chart to match your deployment needs.
- Deploy and verify the application.

Helm simplifies Kubernetes deployments by streamlining configurations and
enabling easy scaling and updates. For more details, see
[Helm Documentation](https://helm.sh/docs/).

## Prerequisites

Before You Begin, ensure the following:

- **Kubernetes Cluster**: Ensure you have a properly installed and
configured Kubernetes cluster.
- **System Requirements**: Verify that your system meets the [minimum requirements](./system-requirements.md).
- **Tools Installed**: Install the required tools:
  - Kubernetes CLI (kubectl)
  - Helm 3 or later
- **cert-manager**: Will be installed as part of the deployment process (instructions provided below)

## Steps to Deploy

To deploy the Smart Intersection Sample Application, copy and paste the entire block of following commands into your terminal and run them:


### Step 1: Clone the Repository

Before you can deploy with Helm, you must clone the repository:

```bash
# Clone the repository
git clone https://github.com/open-edge-platform/edge-ai-suites.git

# Navigate to the Metro AI Suite directory
cd edge-ai-suites/metro-ai-suite/metro-vision-ai-app-recipe/
```

### Step 2: Configure Proxy Settings (If behind a proxy)

If you are deploying in a proxy environment, update the values.yaml file with your proxy settings before installation:

```bash
# Edit the values.yml file to add proxy configuration
nano ./smart-intersection/chart/values.yaml
```

Update the existing proxy configuration in your values.yaml with following values:

```yaml
http_proxy: "http://your-proxy-server:port"
https_proxy: "http://your-proxy-server:port"
no_proxy: "localhost,127.0.0.1,.local,.cluster.local"
```

Replace `your-proxy-server:port` with your actual proxy server details.

### Install cert-manager

The Smart Intersection application requires cert-manager for TLS certificate management. Install cert-manager before deploying the application:

```bash
# Install cert-manager using YAML manifests (recommended for reliability)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.2/cert-manager.yaml

# Wait for cert-manager to be ready
kubectl wait --for=condition=Available --timeout=60s deployment/cert-manager -n cert-manager
kubectl wait --for=condition=Available --timeout=60s deployment/cert-manager-webhook -n cert-manager
```

### Setup Storage Provisioner (For Single-Node Clusters)

Check if your cluster has a default storage class with dynamic provisioning. If not, install a storage provisioner:

```bash
# Check for existing storage classes
kubectl get storageclass

# If no storage classes exist or none are marked as default, install local-path-provisioner
# This step is typically needed for single-node bare Kubernetes installations
# (Managed clusters like EKS/GKE/AKS already have storage classes configured)

# Install local-path-provisioner for automatic storage provisioning
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

# Set it as default storage class
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Verify storage class is ready
kubectl get storageclass
```

### Step 3: Deploy the application

Now you're ready to deploy the Smart Intersection application. To avoid cert-manager webhook issues that can cause database initialization problems, we'll temporarily disable the webhooks during deployment:

```bash
# Temporarily remove webhook configurations to prevent deployment issues
kubectl delete validatingwebhookconfiguration cert-manager-webhook --ignore-not-found
kubectl delete mutatingwebhookconfiguration cert-manager-webhook --ignore-not-found

# Install the chart (works on both single-node and multi-node clusters)
helm upgrade --install smart-intersection ./smart-intersection/chart \
  --create-namespace \
  --set grafana.service.type=NodePort \
  --set global.storageClassName="" \
  -n smart-intersection

# Restore webhook configurations after successful deployment
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.2/cert-manager.yaml
```

> **Note**: Using `global.storageClassName=""` makes the deployment use whatever default storage class exists on your cluster. This works for both single-node and multi-node setups.

> **Note**: The webhook bypass steps above prevent cert-manager webhook validation issues that can cause incomplete deployments and database initialization problems. The webhooks are restored after successful deployment.

## Access Application Services

The Smart Intersection application provides multiple access methods, similar to the Docker Compose setup:

### Nginx Reverse Proxy Access (HTTPS - Recommended)

All services are accessible through the nginx reverse proxy with TLS encryption:

- **Smart Intersection (Main UI)**: `https://<HOST_IP>:30443/` - Fully functional
- **Grafana Dashboard**: `https://<HOST_IP>:30443/grafana/` - Fully functional  
- **NodeRED Editor**: `https://<HOST_IP>:30443/nodered/` - Fully functional
- **DL Streamer API**: `https://<HOST_IP>:30443/api/pipelines/` - Fully functional
- **InfluxDB (Proxy)**: `https://<HOST_IP>:30443/influxdb/` - Basic functionality + API access

### Direct Service Access

Some services also provide direct access on dedicated ports:

- **InfluxDB (Direct)**: `http://<HOST_IP>:30086/` - Fully functional
- **HTTP Redirect**: `http://<HOST_IP>:30080/` - Redirects to HTTPS

### Service Credentials

#### Smart Intersection Web UI
- **Username**: `admin`
- **Password**: Get from secrets:
  ```bash
  kubectl get secret smart-intersection-supass-secret -n smart-intersection -o jsonpath='{.data.supass}' | base64 -d && echo
  ```

#### Grafana Dashboard  
- **Username**: `admin`
- **Password**: `admin`

#### InfluxDB (Both Direct and Proxy Access)
- **Username**: `admin`
- **Password**: Get from secrets:
  ```bash
  kubectl get secret smart-intersection-influxdb-secrets -n smart-intersection -o jsonpath='{.data.influxdb2-admin-password}' | base64 -d && echo
  ```

#### NodeRED Editor
- **No login required** - Visual programming interface

#### DL Streamer Pipeline Server
- **API Access**: No authentication required for status endpoints

> **Note**: InfluxDB provides both direct access (port 30086) for full functionality and proxy access through nginx for basic functionality and API access, matching the Docker Compose setup.

## Uninstall the Application

To uninstall the application, run the following command:

```bash
helm uninstall smart-intersection -n smart-intersection
```

## Delete the Namespace

To delete the namespace and all resources within it, run the following command:

```bash
kubectl delete namespace smart-intersection
```

## Complete Cleanup (Optional)

If you want to completely remove all infrastructure components installed during the setup process, including cert-manager and storage provisioner:

### Remove cert-manager
```bash
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.2/cert-manager.yaml
```

### Remove local-path-provisioner (if installed)
```bash
kubectl delete -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

### Remove additional storage classes (if created)
```bash
kubectl delete storageclass hostpath local-storage standard
```

> **Note**: This complete cleanup will remove all certificate management capabilities and storage provisioning from your cluster. You'll need to reinstall these components for future deployments.

## What to Do Next

- **[Troubleshooting Helm Deployments](./support.md#troubleshooting-helm-deployments)**: Consolidated troubleshooting steps for resolving issues during Helm deployments.
- **[Get Started](./get-started.md)**: Ensure you have completed the initial setup steps before proceeding.

## Supporting Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [Helm Documentation](https://helm.sh/docs/)
