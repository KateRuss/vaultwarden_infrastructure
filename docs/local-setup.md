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

## Stage 2: Ingress Configuration for local deployment

For proper operation of vaultwarden, an HTTPS reverse proxy must be configured. As stated in the [official documentation](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-HTTPS), enabling HTTPS is essentially required since the Bitwarden web vault uses Web Crypto APIs that most browsers only make available in HTTPS contexts.

The [proxy examples page](https://github.com/dani-garcia/vaultwarden/wiki/Proxy-examples) in the official documentation provides configuration examples for various reverse proxies. For this project, **HAproxy Kubernetes Ingress** (by [@devinslick](https://github.com/devinslick)) is used.

### Step 2.1: Prepare the Ingress Manifest

Add the Ingress manifest to your k8s manifests folder. The base configuration is taken from the official docs and slightly modified:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vaultwarden
  namespace: default
  annotations:
    haproxy.org/forwarded-for: "true"
    haproxy.org/compression-algo: "gzip"
    haproxy.org/compression-type: "text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript"
    haproxy.org/http2-enabled: "true"
spec:
  ingressClassName: haproxy
  tls:
  - hosts:
    - vaultwarden.example.tld
    secretName: vaultwarden-tls # since the service is deployed locally,
                                # we will use a self-signed SSL certificate
  rules:
  - host: vaultwarden.example.tld
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: vwapp-service  # must match the name of the app service!
            port:
              number: 80
```

Also update `vwapp_svc.yaml` — change the service type from `NodePort` to `ClusterIP`, since external access will now be handled by the Ingress controller. Push the updated configuration to the repository.

---

### Step 2.2: Install HAproxy Ingress Controller

Switch to the machine with the cluster and install HAproxy:

```bash
kubectl apply -f https://raw.githubusercontent.com/haproxytech/kubernetes-ingress/master/deploy/haproxy-ingress.yaml
```

Wait for the pod to reach `Running` status:

```bash
kubectl get pods -n haproxy-controller
```

---

### Step 2.3: Create IngressClass for HAproxy

Check if IngressClass was created automatically:

```bash
kubectl get ingressclass
```

You should see `haproxy` in the list. If `haproxy` is **not** in the list, HAproxy did not create its IngressClass automatically — create it manually:

```bash
nano /tmp/ingressclass.yaml
```

Insert the following:

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: haproxy
spec:
  controller: haproxy.org/ingress-controller
```

Save and apply:

```bash
kubectl apply -f /tmp/ingressclass.yaml
```

Verify the result:

```bash
kubectl get ingressclass
```

Expected output:

```
NAME      CONTROLLER                       PARAMETERS   AGE
haproxy   haproxy.org/ingress-controller   <none>       8s
traefik   traefik.io/ingress-controller    <none>       2d13h
```
Now haproxy IngressClass created

---

### Step 2.4: Generate a Self-Signed SSL Certificate

Since the service is deployed locally without a real domain, we use a self-signed certificate. The CN value must match the domain specified in the Ingress manifest:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/tls.key \
  -out /tmp/tls.crt \
  -subj "/CN=vaultwarden.example.tld"
```

This creates two files — `tls.key` and `tls.crt`.

Store them as a Kubernetes TLS Secret. **Important:** the secret name must match `spec.tls.secretName` in the Ingress manifest:

```bash
kubectl create secret tls vaultwarden-tls \
  --cert=/tmp/tls.crt \
  --key=/tmp/tls.key
```

Verify the secret was created correctly:

```bash
kubectl get secret vaultwarden-tls
```

Expected output:

```
NAME              TYPE                DATA   AGE
vaultwarden-tls   kubernetes.io/tls   2      39h
```

---

### Step 2.5: Add `--default-ssl-certificate` to HAproxy

Before applying the Ingress manifest, make sure HAproxy knows where to find the certificate. Check the current controller arguments:

```bash
kubectl get deployment haproxy-kubernetes-ingress -n haproxy-controller \
  -o jsonpath='{.spec.template.spec.containers[0].args}' && echo
```

If `--default-ssl-certificate=default/vaultwarden-tls` is **not** in the output, add it:

```bash
kubectl patch deployment haproxy-kubernetes-ingress -n haproxy-controller --type json \
  -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--default-ssl-certificate=default/vaultwarden-tls"}]'
```

Wait for the controller to restart and pick up the new configuration:

```bash
kubectl rollout status deployment/haproxy-kubernetes-ingress -n haproxy-controller
```

---

### Step 2.6: Apply the Ingress Manifest

```bash
kubectl apply -f vwapp_ingress.yaml
kubectl get ingress
```

Now check that the HAproxy service is listening on port 443:

```bash
kubectl get svc -n haproxy-controller
```

> ⚠️ **Important:** The response may only contain port 80:
> ```
> NAME                         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
> haproxy-kubernetes-ingress   NodePort   10.43.254.214   <none>        80:30219/TCP     14m
> ```
> In this case, add port 443 manually:
> ```bash
> kubectl patch svc haproxy-kubernetes-ingress -n haproxy-controller --type json \
>   -p '[{"op":"add","path":"/spec/ports/-","value":{"name":"https","port":443,"targetPort":443,"nodePort":30443,"protocol":"TCP"}}]'
> ```
> Verify:
> ```bash
> kubectl get svc -n haproxy-controller
> ```
> Expected output:
> ```
> NAME                         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                                     AGE
> haproxy-kubernetes-ingress   NodePort   10.43.254.214   <none>        80:30219/TCP,443:30501/TCP,1024:30298/TCP   14m
> ```

---

### Step 3.7: Configure Local Access

Edit the hosts file on your local machine:
- Linux/Mac: `/etc/hosts`
- Windows: `C:\Windows\System32\drivers\etc\hosts` (open as Administrator)

Add the following line — the domain must match what you specified in the Ingress manifest. **Don't forget to uncomment the line if it's commented out!**

```
127.0.0.1    vaultwarden.example.tld
```

Then open a separate terminal and create an SSH tunnel:

```bash
ssh -L 443:localhost:30501 -N user@<remote-machine-ip>
```

Open your browser and navigate to:

```
https://vaultwarden.example.tld
```

If everything is configured correctly, you will see the Vaultwarden login page. The browser will warn about the self-signed certificate — this is expected. Click "Advanced" → "Proceed" to continue.

You can now create an account, log in, and verify that all pages and features load without errors. 🎉

---

### Troubleshooting

<details>
<summary>ERR_SSL_PROTOCOL_ERROR</summary>

**Step 1:** Check that the TLS secret exists in the correct namespace:

```bash
kubectl get secret vaultwarden-tls -n default
```

If the secret is missing, recreate it and try again.

**Step 2:** If the secret exists, check HAproxy logs:

```bash
kubectl logs -n haproxy-controller deployment/haproxy-kubernetes-ingress | grep -i "tls\|cert\|secret"
```

If you see an error like:

```
reload required : Runtime update of cert file '/etc/haproxy/certs/frontend/default_vaultwarden-tls.pem' failed :
Can't edit the crt-list: crt-list '/etc/haproxy/certs/frontend' does not exist!
```

This means HAproxy cannot find the certificate directory. Check whether `--default-ssl-certificate` is in the controller arguments:

```bash
kubectl get deployment haproxy-kubernetes-ingress -n haproxy-controller \
  -o jsonpath='{.spec.template.spec.containers[0].args}' && echo
```

If it's missing, add it:

```bash
kubectl patch deployment haproxy-kubernetes-ingress -n haproxy-controller --type json \
  -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--default-ssl-certificate=default/vaultwarden-tls"}]'
```

Wait for the controller to restart, then try accessing the application again.

</details>