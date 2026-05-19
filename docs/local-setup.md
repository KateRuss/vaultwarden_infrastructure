# Vaultwarden Infrastructure
This document details local deployment of the cluster. For skipping additional explanation run step 1 and then go to step 3

## Prerequisites

To work with this project you will need:

- **WSL2** with Ubuntu distribution — [Install WSL](https://learn.microsoft.com/en-us/windows/wsl/install)
- **Docker Engine** — [Install Docker](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)
- **kubectl + k3s** (deployed on a separate physical machine with Debian, but a VM works just as well)
  - [Install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management)
  - [k3s Quick Start](https://docs.k3s.io/quick-start)
- **git** (on all machines/VMs where project work is done)

---

## Stage 1: Local Cluster Deployment in k8s

The goal of this stage is to deploy the application locally in a k8s cluster.

### Step 1: Cluster Preparation

Connect to the remote machine via SSH where the cluster will be deployed. Make sure `kubectl` and `k3s` are already installed, then run the following commands:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
kubectl get nodes
```

If successful, you should see output similar to this:

```
NAME               STATUS   ROLES           AGE   VERSION
debian-katerina    Ready    control-plane   24h   v1.35.4+k3s1
```

> ⚠️ **If the command returns an error:**
> k3s stores its config at `/etc/rancher/k3s/k3s.yaml`, while kubectl looks in `~/.kube/config` by default. Double-check the paths and configuration.

---

### Step 2: Manifests Planning

To understand which components are needed, we look at the main vaultwarden repository.

From the default Dockerfile we can see that:
- the application listens on port **80**
- it has a dedicated volume at **/data**

```dockerfile
VOLUME /data
EXPOSE 80
```

Additionally, we need a database. The wiki contains instructions for connecting an external database. For this project **MariaDB** is used:
[Using the MariaDB (MySQL) Backend](https://github.com/dani-garcia/vaultwarden/wiki/Using-the-MariaDB-%28MySQL%29-Backend)

The docker-compose example in the wiki provides all the information needed to configure the database connection. Based on this, the following manifests are planned:

| Deployments | Services | Additional |
|-------------|----------|------------|
| `db_deploy` — MariaDB deployment | `db_service` — DB service, type: ClusterIP | `db_pvc` — PersistentVolumeClaim for DB storage |
| `vwapp_deploy` — Vaultwarden deployment | `vwapp_service` — App service, type: NodePort | `vwapp_pvc` — PersistentVolumeClaim for app data |

---

### Step 3: Running

Connect to the cluster machine via SSH, clone the repository, and navigate to the project folder:

```bash
git clone https://github.com/KateRuss/vaultwarden_infrastructure.git
cd vaultwarden_infrastructure/k8s
```

Apply all manifests:

```bash
kubectl create -f .
```

Wait for the cluster components to start, then verify:

```bash
kubectl get pods
kubectl get pvc
kubectl get services
kubectl get deployments
```

Make sure all pods have status **Running** and all PVCs have status **Bound**.

Open a separate terminal and create an SSH tunnel to forward the remote port to your local machine:

```bash
ssh -L 8080:localhost:30080 user@<remote-machine-ip>
```

Then open your browser and go to:

```
http://localhost:8080
```

If everything is working correctly, you will see the Vaultwarden login page. 🎉