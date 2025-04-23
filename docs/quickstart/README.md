# Tornjak Deployment with SPIRE on Kubernetes Quickstart

This tutorial demonstrates how to deploy [Tornjak](https://github.com/spiffe/tornjak) with [SPIRE](https://github.com/spiffe/spire) on a Kubernetes cluster using Minikube, inspired by the [SPIRE Quickstart for Kubernetes](https://spiffe.io/docs/latest/try/getting-started-k8s/). It covers setting up a local SPIRE server with a co-located Tornjak backend, deploying a SPIRE agent, and accessing the Tornjak API and optional frontend.

## Overview

**SPIRE** (SPIFFE Runtime Environment) is an open-source tool for managing SPIFFE identities in distributed systems. It issues SPIFFE IDs and SVIDs (SPIFFE Verifiable Identity Documents) via the SPIFFE Workload API, enabling mutual trust between workloads.

**Tornjak** is a management plane and GUI for SPIRE, simplifying administration across multiple clusters with an intuitive interface for managing and visualizing SPIFFE identities.

This guide walks through:
- Setting up prerequisites
- Configuring deployment files
- Deploying SPIRE and Tornjak
- Accessing the Tornjak backend and optional frontend
- Cleaning up resources

## Contents
- [Step 0: Prerequisites](#step-0-prerequisites)
- [Step 1: Setup Deployment Files](#step-1-setup-deployment-files)
- [Step 2: Deploy SPIRE and Tornjak](#step-2-deploy-spire-and-tornjak)
- [Step 3: Access Tornjak](#step-3-access-tornjak)
- [Cleanup](#cleanup)
- [Troubleshooting](#troubleshooting)

## Step 0: Prerequisites

Ensure you have the following installed:
- **Minikube**: Version 1.33.0 or later. [Install Minikube](https://minikube.sigs.k8s.io/docs/start/).
- **Docker**: Version 24.0.7 or later. [Install Docker](https://docs.docker.com/get-docker/).
- **kubectl**: Compatible with Kubernetes 1.28 or later. [Install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/).
- **Git**: For cloning the Tornjak repository. [Install Git](https://git-scm.com/downloads).

**Note**: This tutorial uses Minikube for simplicity, but you can adapt it for an existing Kubernetes cluster. Ensure Docker Desktop is running before starting Minikube.

## Step 1: Setup Deployment Files

### Start Minikube

Launch a Minikube cluster with the Docker driver:

```bash
minikube start --driver=docker
```

Example output:
```
😄  minikube v1.33.0 on Darwin 14.5
✨  Using the docker driver
👍  Starting control plane node minikube in cluster minikube
🔥  Creating docker container (CPUs=2, Memory=4000MB) ...
🐳  Preparing Kubernetes v1.28.3 on Docker 24.0.7 ...
🔎  Verifying Kubernetes components...
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster
```

Verify the cluster is running:

```bash
kubectl get nodes
```

```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   2m    v1.28.3
```

If Minikube fails to start, see [Troubleshooting: Minikube Fails to Start](#troubleshooting).

### Clone the Tornjak Repository

Clone the Tornjak repository and navigate to the quickstart directory:

```bash
git clone https://github.com/spiffe/tornjak.git
cd tornjak/docs/quickstart
```

This directory contains configuration files similar to the [SPIRE Kubernetes Quickstart](https://spiffe.io/docs/latest/try/getting-started-k8s/), with additions for Tornjak, such as `tornjak-configmap.yaml`.

### Review Tornjak Configuration

Inspect the Tornjak backend configuration:

```bash
cat tornjak-configmap.yaml
```

Example content:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tornjak-agent
  namespace: spire
data:
  server.conf: |
    server {
      spire_socket_path = "unix:///tmp/spire-server/private/api.sock"
      http {
        enabled = true
        port = 10000
      }
    }
    plugins {
      DataStore "sql" {
        plugin_data {
          drivername = "sqlite3"
          filename = "/run/spire/data/tornjak.sqlite3"
        }
      }
    }
```

For detailed configuration options, see the [Tornjak configuration documentation](https://github.com/spiffe/tornjak/blob/main/docs/config-tornjak-server.md).

### Configure the StatefulSet

This tutorial uses a deployment where the Tornjak backend runs as a sidecar container in the same pod as the SPIRE server, sharing a socket for communication. Copy the appropriate StatefulSet configuration:

```bash
cp server-statefulset-examples/backend-sidecar-server-statefulset.yaml server-statefulset.yaml
```

Verify the StatefulSet configuration:

```bash
cat server-statefulset.yaml
```

Key differences from the SPIRE quickstart:
- Adds a `tornjak-backend` container using `ghcr.io/spiffe/tornjak-backend:2.1.0`.
- Mounts a shared `socket` volume for SPIRE and Tornjak communication.
- Includes a `tornjak-config` volume from the `tornjak-agent` ConfigMap.

Example excerpt:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: spire-server
  namespace: spire
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spire-server
  template:
    spec:
      containers:
        - name: spire-server
          image: ghcr.io/spiffe/spire-server:1.9.0
          args:
            - -config
            - /run/spire/config/server.conf
          volumeMounts:
            - name: socket
              mountPath: /tmp/spire-server/private
        - name: tornjak-backend
          image: ghcr.io/spiffe/tornjak-backend:2.1.0
          args:
            - --tornjak-config
            - /run/spire/tornjak-config/server.conf
          volumeMounts:
            - name: socket
              mountPath: /tmp/spire-server/private
      volumes:
        - name: socket
          emptyDir: {}
        - name: tornjak-config
          configMap:
            name: tornjak-agent
```

**Security Note**: For production, secure the SQLite database and SPIRE socket with appropriate access controls and consider using a managed database instead of SQLite.

## Step 2: Deploy SPIRE and Tornjak

Apply the configuration files to deploy SPIRE and Tornjak:

```bash
kubectl apply -f spire-namespace.yaml -f server-account.yaml -f spire-bundle-configmap.yaml -f tornjak-configmap.yaml -f server-cluster-role.yaml -f server-configmap.yaml -f server-statefulset.yaml -f server-service.yaml
```

Example output:
```
namespace/spire created
serviceaccount/spire-server created
configmap/spire-bundle created
configmap/tornjak-agent created
clusterrole.rbac.authorization.k8s.io/spire-server-trust-role created
clusterrolebinding.rbac.authorization.k8s.io/spire-server-trust-role-binding created
configmap/spire-server created
statefulset.apps/spire-server created
service/spire-server created
```

**Note for Windows Users**: Replace backslashes (`\`) with backticks (`` ` ``) in the `kubectl apply` command for compatibility with Windows terminals:

```bash
kubectl apply -f spire-namespace.yaml `-f server-account.yaml `-f spire-bundle-configmap.yaml `-f tornjak-configmap.yaml `-f server-cluster-role.yaml `-f server-configmap.yaml `-f server-statefulset.yaml `-f server-service.yaml
```

Verify the SPIRE server is running:

```bash
kubectl get statefulset -n spire
```

```
NAME           READY   AGE
spire-server   1/1     30s
```

If the status shows `0/1`, wait a few minutes and check again.

### Deploy the SPIRE Agent

Apply the agent configurations:

```bash
kubectl apply -f agent-account.yaml -f agent-cluster-role.yaml -f agent-configmap.yaml -f agent-daemonset.yaml
```

Example output:
```
serviceaccount/spire-agent created
clusterrole.rbac.authorization.k8s.io/spire-agent-cluster-role created
clusterrolebinding.rbac.authorization.k8s.io/spire-agent-cluster-role-binding created
configmap/spire-agent created
daemonset.apps/spire-agent created
```

Verify the agent is running:

```bash
kubectl get daemonset -n spire
```

```
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   AGE
spire-agent   1         1         1       1            1           20s
```

### Create Registration Entries

Register the node:

```bash
kubectl exec -n spire spire-server-0 -c spire-server -- /opt/spire/bin/spire-server entry create \
    -spiffeID spiffe://example.org/ns/spire/sa/spire-agent \
    -selector k8s_sat:cluster:demo-cluster \
    -selector k8s_sat:agent_ns:spire \
    -selector k8s_sat:agent_sa:spire-agent \
    -node
```

Example output:
```
Entry ID         : 03d0ec2b-54b7-4340-a0b9-d3b2cf1b041a
SPIFFE ID        : spiffe://example.org/ns/spire/sa/spire-agent
Parent ID        : spiffe://example.org/spire/server
Selector         : k8s_sat:cluster:demo-cluster
Selector         : k8s_sat:agent_ns:spire
Selector         : k8s_sat:agent_sa:spire-agent
```

Register the workload:

```bash
kubectl exec -n spire spire-server-0 -c spire-server -- /opt/spire/bin/spire-server entry create \
    -spiffeID spiffe://example.org/ns/default/sa/default \
    -parentID spiffe://example.org/ns/spire/sa/spire-agent \
    -selector k8s:ns:default \
    -selector k8s:sa:default
```

Example output:
```
Entry ID         : 11a367ab-7095-4390-ab89-34dea5fddd61
SPIFFE ID        : spiffe://example.org/ns/default/sa/default
Parent ID        : spiffe://example.org/ns/spire/sa/spire-agent
Selector         : k8s:ns:default
Selector         : k8s:sa:default
```

### Deploy a Test Workload

Deploy a client workload to test the SPIRE Workload API:

```bash
kubectl apply -f client-deployment.yaml
```

Verify the workload can fetch an SVID:

```bash
kubectl exec -it $(kubectl get pods -o=jsonpath='{.items[0].metadata.name}' -l app=client) -- \
    /opt/spire/bin/spire-agent api fetch -socketPath /run/spire/sockets/agent.sock
```

Example output:
```
Received 1 svid after 10ms
SPIFFE ID: spiffe://example.org/ns/default/sa/default
SVID Valid After: 2025-04-23 12:00:00 +0000 UTC
SVID Valid Until: 2025-04-23 13:00:10 +0000 UTC
```

Verify the pod images:

```bash
kubectl -n spire describe pod spire-server-0 | grep "Image:"
```

Example output:
```
Image: ghcr.io/spiffe/spire-server:1.9.0
Image: ghcr.io/spiffe/tornjak-backend:2.1.0
```

## Step 3: Access Tornjak

### Step 3a: Access the Tornjak Backend

Forward the Tornjak backend port to your local machine:

```bash
kubectl -n spire port-forward spire-server-0 10000:10000
```

Example output:
```
Forwarding from 127.0.0.1:10000 -> 10000
Forwarding from [::1]:10000 -> 10000
```

Open a browser and navigate to:

```
http://localhost:10000/api/v1/tornjak/serverinfo
```

You should see a JSON response with server information, confirming the Tornjak backend is accessible.

### Step 3b: Access the Tornjak Frontend (Optional)

To view the Tornjak UI, run the frontend locally using Docker. Ensure the backend port-forward is active (run `kubectl port-forward` in the background if needed):

```bash
kubectl -n spire port-forward spire-server-0 10000:10000 &
```

In a new terminal, start the Tornjak frontend container:

```bash
docker run -p 3000:3000 -e REACT_APP_API_SERVER_URI='http://localhost:10000' ghcr.io/spiffe/tornjak-frontend:2.1.0
```

Example output:
```
> tornjak-frontend@0.1.0 start
> react-scripts start

Compiled successfully!
You can now view tornjak-frontend in the browser.
  Local: http://localhost:3000
```

**Note**: It may take a few minutes for the frontend to compile and start.

Open a browser and navigate to:

```
http://localhost:3000
```

You should see the Tornjak UI, allowing you to visualize and manage SPIFFE identities.

**Troubleshooting Tip**: If the UI fails to load, ensure the backend is accessible at `http://localhost:10000` and check Docker logs for errors.

## Cleanup

Remove the deployed resources:

1. Delete the client workload:
```bash
kubectl delete deployment client
```

2. Delete the SPIRE namespace and its contents:
```bash
kubectl delete namespace spire
```

3. Delete cluster-wide roles:
```bash
kubectl delete clusterrole spire-server-trust-role spire-agent-cluster-role
kubectl delete clusterrolebinding spire-server-trust-role-binding spire-agent-cluster-role-binding
```

4. Stop Minikube:
```bash
minikube stop
```

## Troubleshooting

### Minikube Fails to Start

**Symptom**: `minikube start` fails with a Docker-related error, e.g., "Cannot connect to the Docker daemon."

**Solution**:
1. Verify Docker is running:
   - On macOS/Windows, open Docker Desktop or run:
     ```bash
     open -a Docker
     ```
2. Check Docker context:
   ```bash
   docker context ls
   docker context use default
   ```
3. Reset Minikube if needed:
   ```bash
   minikube delete
   minikube start --driver=docker
   ```
4. Ensure sufficient resources (e.g., 4GB RAM, 2 CPUs) are allocated to Docker.

### Tornjak Backend Unreachable

**Symptom**: `http://localhost:10000/api/v1/tornjak/serverinfo` returns a connection error.

**Solution**:
1. Verify the port-forward is active:
   ```bash
   kubectl -n spire port-forward spire-server-0 10000:10000
   ```
2. Check pod status:
   ```bash
   kubectl -n spire get pods
   ```
3. View pod logs for errors:
   ```bash
   kubectl -n spire logs spire-server-0 -c tornjak-backend
   ```

### Frontend Fails to Load

**Symptom**: The Tornjak UI at `http://localhost:3000` is blank or shows errors.

**Solution**:
1. Ensure the backend is accessible at `http://localhost:10000`.
2. Check Docker container logs:
   ```bash
   docker ps
   docker logs <container-id>
   ```
3. Restart the frontend container:
   ```bash
   docker stop <container-id>
   docker run -p 3000:3000 -e REACT_APP_API_SERVER_URI='http://localhost:10000' ghcr.io/spiffe/tornjak-frontend:2.1.0
   ```

For additional help, consult the [SPIRE documentation](https://spiffe.io/docs/latest/) or [Tornjak repository](https://github.com/spiffe/tornjak).

**Production Note**: This tutorial is for demonstration purposes. For production, implement RBAC, network policies, and secure storage for sensitive data.