# k3s + Cilium + NGINX Ingress + Private Registry + GitHub Actions CI/CD

A complete, start-to-finish guide for a single VPS.

**What you end up with**

- A single-node **k3s** cluster using **Cilium** for networking (replacing flannel and kube-proxy)
- **NGINX Ingress Controller** (F5) as a DaemonSet on the node's ports 80/443
- **cert-manager** issuing free Let's Encrypt HTTPS certificates
- A private **Docker registry** inside the cluster
- **GitHub Actions** that build your app, push it to the registry, and deploy it:
  - push to `dev` deploys to **staging**
  - push to `main` deploys to **production**
- Each environment has its own **namespace**, its own **restricted ServiceAccount**, its own **Linux deploy user**, and its own **SSH key**, so the staging key can never touch production

```
                       git push dev / main
                              |
                       GitHub Actions
                 build image --> push --> https://<REGISTRY_HOST>
                              |
                  ssh as deploy-<env> user (restricted key)
                              |
        +---------------------+----------------------+
        |   namespace houston-staging                |   namespace houston-production
        |   ServiceAccount: deployer (this ns only)  |   ServiceAccount: deployer (this ns only)
        |   app --> https://<STAGING_DOMAIN>         |   app --> https://<PRODUCTION_DOMAIN>
        +--------------------------------------------+
                  both served by one NGINX ingress, one registry, one cert-manager
```

**Contents**

- Part A: Cluster, ingress, certificates, registry (Steps 0-13)
- Part B: Environments and CI/CD (Steps 14-23)
- Troubleshooting and things to remember

---

# Part A: Cluster, Ingress, Certificates, Registry

## Step 0: Set your variables

All commands read from environment variables, so you only edit this block once.

```bash
export SETUP_NODEIP=<YOUR_SERVER_IP>                    # public IP of the VPS
export SETUP_CLUSTERTOKEN=$(openssl rand -hex 32)       # shared cluster secret, save it
export LE_EMAIL=<YOUR_REAL_EMAIL>                       # must be a real address, NOT example.com
export REGISTRY_HOST=registry.${SETUP_NODEIP}.nip.io    # or your own domain, e.g. registry.example.org
export REGISTRY_USER=<REGISTRY_USERNAME>
export REGISTRY_PASS='<A_STRONG_PASSWORD>'              # keep the single quotes
```

**Variables are lost when your SSH session drops.** Save them so you can reload them at any time:

```bash
cat > ~/cluster.env <<EOF
export SETUP_NODEIP=$SETUP_NODEIP
export SETUP_CLUSTERTOKEN=$SETUP_CLUSTERTOKEN
export LE_EMAIL=$LE_EMAIL
export REGISTRY_HOST=$REGISTRY_HOST
export REGISTRY_USER=$REGISTRY_USER
export REGISTRY_PASS='$REGISTRY_PASS'
EOF
chmod 600 ~/cluster.env

# in any new session:
source ~/cluster.env
```

### About `nip.io`

`nip.io` is a public wildcard DNS service. A name like `registry.1.2.3.4.nip.io` resolves to `1.2.3.4`, and Let's Encrypt will issue a real certificate for it. That means you can host the registry without using one of your own domains.

Trade-offs: nip.io certificates share Let's Encrypt rate limits with every other nip.io user, you depend on a third-party DNS service, and the IP is baked into every image name. If you hit rate limits, `sslip.io` works the same way. For a long-lived setup, set `REGISTRY_HOST` to a real domain and point an A record at the server.

---

## Step 1: Install k3s

```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="--node-ip $SETUP_NODEIP \
  --disable=local-storage,metrics-server,servicelb,traefik \
  --flannel-backend=none \
  --disable-network-policy \
  --disable-cloud-controller \
  --disable-kube-proxy" \
  K3S_TOKEN=$SETUP_CLUSTERTOKEN \
  K3S_KUBECONFIG_MODE=600 sh -s -
```

### What each flag does

| Flag                         | Meaning                                     | Consequence                                                                       |
| ---------------------------- | ------------------------------------------- | --------------------------------------------------------------------------------- |
| `--node-ip`                  | The IP the node advertises to the cluster   | Pods and the API server use this address                                          |
| `--disable=traefik`          | Skips the bundled ingress controller        | Frees ports 80/443 for NGINX                                                      |
| `--disable=servicelb`        | Skips k3s's built-in LoadBalancer emulation | `LoadBalancer` services stay `<pending>`, so we use `hostPort` instead            |
| `--disable=local-storage`    | Skips the default local-path storage class  | You must install storage yourself if you need PVCs (the registry uses `hostPath`) |
| `--disable=metrics-server`   | Skips the metrics API                       | `kubectl top` won't work until you add it                                         |
| `--flannel-backend=none`     | Turns off k3s's built-in pod network        | **Node stays NotReady until you install a CNI**                                   |
| `--disable-network-policy`   | Skips k3s's NetworkPolicy controller        | Cilium enforces network policies instead                                          |
| `--disable-cloud-controller` | Skips k3s's mini cloud controller           | Fine on a plain VPS                                                               |
| `--disable-kube-proxy`       | Skips kube-proxy                            | **Cilium must replace it**, or Services won't work                                |
| `K3S_KUBECONFIG_MODE=600`    | Only root can read the admin kubeconfig     | **Important for this guide**, see below                                           |

> **Why `600` and not `644`:** `/etc/rancher/k3s/k3s.yaml` holds the cluster **admin** credentials. With mode `644`, any local user, including the restricted deploy users created later, could read it and bypass their limited permissions. If you already installed with `644`, fix it with:
>
> ```bash
> echo 'write-kubeconfig-mode: "0600"' | sudo tee -a /etc/rancher/k3s/config.yaml
> sudo chmod 600 /etc/rancher/k3s/k3s.yaml
> sudo systemctl restart k3s
> ```

---

## Step 2: Set up kubectl access (as root)

```bash
mkdir -p $HOME/.kube
sudo cp /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
chmod 600 $HOME/.kube/config
```

At this point `kubectl get nodes` shows the node as **NotReady**. That's expected: with no CNI, Kubernetes marks the node unready and adds a `node.kubernetes.io/not-ready:NoSchedule` taint, so nothing can be scheduled, not even CoreDNS.

---

## Step 3: Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

Helm is the package manager for Kubernetes. A chart bundles all the manifests for an application, and you configure it with `--set` flags or a values file.

---

## Step 4: Install Cilium

Every pod needs an IP address and a way to reach other pods, Services, and the internet. Kubernetes delegates this to a **CNI plugin**, which is Cilium here. Cilium also replaces **kube-proxy** (translating Service IPs to pod IPs) and enforces **NetworkPolicies** using eBPF.

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update

helm install cilium cilium/cilium \
  --namespace kube-system \
  --set k8sServiceHost=$SETUP_NODEIP \
  --set k8sServicePort=6443 \
  --set kubeProxyReplacement=true \
  --set operator.replicas=1
```

| Setting                             | Why                                                                                                                            |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `k8sServiceHost` / `k8sServicePort` | Where Cilium finds the API server. Without kube-proxy there are no Service IPs yet, so it needs the real node IP and port 6443 |
| `kubeProxyReplacement=true`         | Cilium does kube-proxy's job, which is required because kube-proxy is disabled                                                 |
| `operator.replicas=1`               | The default is 2, which suits multi-node clusters, not a single node                                                           |

Verify:

```bash
kubectl get pods -n kube-system -w     # cilium, cilium-envoy, cilium-operator, coredns: Running
kubectl get nodes                      # becomes Ready within a minute or two
```

---

## Step 5: Install the NGINX ingress controller (DaemonSet)

> The community `kubernetes/ingress-nginx` project was retired in March 2026 and no longer receives security patches. This guide uses the actively maintained **F5 NGINX Ingress Controller**, which uses the `nginx.org/` annotation prefix.

```bash
helm install nginx-ingress oci://ghcr.io/nginx/charts/nginx-ingress \
  --namespace nginx-ingress --create-namespace \
  --set controller.kind=daemonset \
  --set controller.hostPort.enable=true \
  --set controller.service.type=ClusterIP
```

| Setting                             | Why                                                                                                  |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `controller.kind=daemonset`         | One controller pod per node                                                                          |
| `controller.hostPort.enable=true`   | Binds the pod directly to the node's ports 80/443. With no servicelb, this is how traffic reaches it |
| `controller.service.type=ClusterIP` | No LoadBalancer is needed, so there's no external IP to wait for                                     |

Verify:

```bash
kubectl get ds,pods,svc -n nginx-ingress -o wide
```

The DaemonSet should show `DESIRED 1 / READY 1`. The service's `EXTERNAL-IP` says `<none>`, which is correct: your server's IP is the entry point.

---

## Step 6: Open the firewall and test

```bash
ufw allow 22/tcp     # do this BEFORE enabling ufw, so you don't lock yourself out
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 6443/tcp   # only if you need remote kubectl or additional nodes (not needed for CI/CD here)
```

Then, **from your own computer**:

```bash
curl -i http://<YOUR_SERVER_IP>
```

A `404 Not Found` from nginx means everything works. If it times out, also open ports 80/443 in your hosting provider's firewall panel (Hostinger: hPanel, VPS, Firewall). `ufw` only controls the on-server firewall. Port 80 must be reachable from the internet because Let's Encrypt validates through it.

---

## Step 7: Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true

kubectl get pods -n cert-manager     # cert-manager, cainjector, webhook: all Running
```

Create the Let's Encrypt issuer. **Use a real email**: Let's Encrypt rejects placeholder domains like `example.com`, and the issuer would stay `READY False`.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ${LE_EMAIL}
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
EOF

kubectl get clusterissuer letsencrypt      # READY must be True
```

If it shows `False`: `kubectl describe clusterissuer letsencrypt | tail -15`.

---

## Step 8: Create the registry login

```bash
sudo apt install -y apache2-utils

htpasswd -Bbn "$REGISTRY_USER" "$REGISTRY_PASS" > htpasswd

kubectl create namespace registry
kubectl -n registry create secret generic registry-auth --from-file=htpasswd
rm htpasswd

# verify: should print "<user>:$2y$..."
kubectl get secret registry-auth -n registry -o jsonpath='{.data.htpasswd}' | base64 -d
```

---

## Step 9: Deploy the registry (Deployment + Service)

`local-storage` is disabled, so a normal PVC won't work. This uses a `hostPath` folder on the server, which is fine for a single node. Images are stored in `/var/lib/registry-data`.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry
  namespace: registry
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels: {app: registry}
  template:
    metadata:
      labels: {app: registry}
    spec:
      containers:
        - name: registry
          image: registry:2
          ports:
            - containerPort: 5000
          env:
            - {name: REGISTRY_AUTH, value: htpasswd}
            - {name: REGISTRY_AUTH_HTPASSWD_REALM, value: Registry}
            - {name: REGISTRY_AUTH_HTPASSWD_PATH, value: /auth/htpasswd}
            - {name: REGISTRY_STORAGE_DELETE_ENABLED, value: "true"}
          volumeMounts:
            - {name: data, mountPath: /var/lib/registry}
            - {name: auth, mountPath: /auth, readOnly: true}
      volumes:
        - name: data
          hostPath:
            path: /var/lib/registry-data
            type: DirectoryOrCreate
        - name: auth
          secret:
            secretName: registry-auth
---
apiVersion: v1
kind: Service
metadata:
  name: registry
  namespace: registry
spec:
  selector: {app: registry}
  ports:
    - port: 5000
      targetPort: 5000
EOF

kubectl get pods,svc -n registry -w     # pod must be 1/1 Running
```

**Do not skip this step.** Without the Deployment and Service, the Ingress in the next step has nothing to forward to and returns `502 Bad Gateway`.

---

## Step 10: Create the registry Ingress with HTTPS

Two annotations matter on this controller. You'll use them again for every app Ingress:

| Annotation                                          | Why it's needed                                                                                                                                                                                                                                                                                                                 |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `acme.cert-manager.io/http01-edit-in-place: "true"` | The F5 NGINX controller serves only **one Ingress per hostname**. Without this, cert-manager creates a second, temporary Ingress for the Let's Encrypt challenge and the controller ignores it, so the certificate stays `READY False` forever. This makes cert-manager add the challenge path to your existing Ingress instead |
| `ingress.kubernetes.io/ssl-redirect: "false"`       | The controller redirects HTTP to HTTPS by default. The challenge is fetched over HTTP, and HTTPS has no certificate yet (`tls: unrecognized name`), so the redirect must be off until the certificate exists. You turn it back on in Step 12                                                                                    |

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: registry
  namespace: registry
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
    acme.cert-manager.io/http01-edit-in-place: "true"
    ingress.kubernetes.io/ssl-redirect: "false"
    nginx.org/client-max-body-size: "0"
    nginx.org/proxy-read-timeout: "600s"
    nginx.org/proxy-send-timeout: "600s"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [${REGISTRY_HOST}]
      secretName: registry-tls
  rules:
    - host: ${REGISTRY_HOST}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: registry
                port:
                  number: 5000
EOF
```

`client-max-body-size: "0"` and the timeouts exist because image layers are large. Without them, pushes fail with `413 Request Entity Too Large` or time out.

You don't create a `Certificate` resource yourself: cert-manager builds it from the `cert-manager.io/cluster-issuer` annotation and the `tls:` block.

---

## Step 11: Wait for the certificate

```bash
kubectl get certificate -n registry -w     # READY becomes True in 1-2 minutes
```

If it stays `False` after a few minutes, see the troubleshooting section.

---

## Step 12: Turn the HTTPS redirect back on

```bash
kubectl annotate ingress registry -n registry --overwrite \
  ingress.kubernetes.io/ssl-redirect="true"
```

Keep the `edit-in-place` annotation: renewals (about every 60 days) use the same mechanism.

---

## Step 13: Test the registry

**From the server or your computer:**

```bash
# 401 Unauthorized is the SUCCESS result: HTTPS works and the registry wants a login
curl -i https://${REGISTRY_HOST}/v2/

# with credentials, expect {"repositories":[]}
curl -u "$REGISTRY_USER:$REGISTRY_PASS" https://${REGISTRY_HOST}/v2/_catalog
```

**Push a test image from your computer** (your laptop doesn't have the server's variables, so substitute your values):

```bash
docker login <REGISTRY_HOST>

docker pull busybox:1.36
docker tag busybox:1.36 <REGISTRY_HOST>/test/busybox:1.0
docker push <REGISTRY_HOST>/test/busybox:1.0

curl -u <USER>:'<PASSWORD>' https://<REGISTRY_HOST>/v2/_catalog
curl -u <USER>:'<PASSWORD>' https://<REGISTRY_HOST>/v2/test/busybox/tags/list
```

**Pull it into k3s (on the server):**

```bash
kubectl create namespace testpull
kubectl -n testpull create secret docker-registry regcred \
  --docker-server=${REGISTRY_HOST} \
  --docker-username="$REGISTRY_USER" \
  --docker-password="$REGISTRY_PASS"

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pulltest
  namespace: testpull
spec:
  imagePullSecrets:
    - name: regcred
  restartPolicy: Never
  containers:
    - name: busybox
      image: ${REGISTRY_HOST}/test/busybox:1.0
      command: ["sh", "-c", "echo pulled from private registry && sleep 30"]
EOF

kubectl get pod -n testpull -w
kubectl logs -n testpull pulltest       # "pulled from private registry"

kubectl delete namespace testpull       # clean up
```

When this works, the infrastructure is done.

---

# Part B: Environments and CI/CD

You'll run Steps 15, 16 and 20 **once per environment** (staging, then production). Everything that is environment-specific is driven by two variables, `APP_ENV` and `APP_HOST`, plus the names derived from them:

| Variable                | Staging                                         | Production                                 |
| ----------------------- | ----------------------------------------------- | ------------------------------------------ |
| `APP_ENV`               | `staging`                                       | `production`                               |
| `APP_HOST`              | `<STAGING_DOMAIN>` (e.g. `staging.example.com`) | `<PRODUCTION_DOMAIN>` (e.g. `example.com`) |
| `NS` (derived)          | `houston-staging`                               | `houston-production`                       |
| `DEPLOY_USER` (derived) | `deploy-staging`                                | `deploy-production`                        |
| Git branch              | `dev`                                           | `main`                                     |

## Step 14: DNS records

Create an A record for each app hostname **before** the first deploy, because Let's Encrypt validates through it:

```
<STAGING_DOMAIN>      A   <YOUR_SERVER_IP>
<PRODUCTION_DOMAIN>   A   <YOUR_SERVER_IP>
```

Check that it resolves: `getent hosts <STAGING_DOMAIN>` should print your server IP.

---

## Step 15: Per-environment server setup (run once per environment)

This creates the namespace, the registry pull secret, a ServiceAccount that can only act inside that one namespace, and a Linux user whose kubeconfig uses that ServiceAccount.

### 15a. Choose the environment

```bash
source ~/cluster.env

# ---- staging ----
export APP_ENV=staging
export APP_HOST=<STAGING_DOMAIN>

# ---- production (run the whole step again with these instead) ----
# export APP_ENV=production
# export APP_HOST=<PRODUCTION_DOMAIN>

export NS=houston-${APP_ENV}
export DEPLOY_USER=deploy-${APP_ENV}

echo "env=$APP_ENV  host=$APP_HOST  ns=$NS  user=$DEPLOY_USER"
```

Check the `echo` line before continuing.

### 15b. Namespace and registry pull secret

These commands are safe to re-run.

```bash
kubectl create namespace $NS --dry-run=client -o yaml | kubectl apply -f -

kubectl -n $NS create secret docker-registry regcred \
  --docker-server=$REGISTRY_HOST \
  --docker-username="$REGISTRY_USER" \
  --docker-password="$REGISTRY_PASS" \
  --dry-run=client -o yaml | kubectl apply -f -
```

Secrets are per-namespace, which is why each environment needs its own `regcred`.

### 15c Optional step: Persistent storage (local-path-provisioner)

> **Where to put this:** at the end of Part A, after Step 13. Do it before deploying anything that needs a PersistentVolumeClaim (a database, uploads, and so on).

k3s normally bundles a local storage provisioner, but this guide disables it with `--disable=local-storage` in Step 1. Without a storage class, any PVC stays `Pending` forever. This step installs the same provisioner (Rancher's `local-path-provisioner`) yourself, so you control its version.

#### 1. Find the latest release and store it

```bash
LP_VERSION=$(curl -s https://api.github.com/repos/rancher/local-path-provisioner/releases/latest \
  | grep -m1 '"tag_name"' | cut -d'"' -f4)
echo "$LP_VERSION"
```

Other ways to check: the releases page at https://github.com/rancher/local-path-provisioner/releases (look for **Latest**), or `gh release view --repo rancher/local-path-provisioner --json tagName -q .tagName`. Ignore tags ending in `-rc1`, which are release candidates.

Write down the version it prints, so you can reproduce the install later. Don't use anything older than `v0.0.36`, which contains a security fix for a helper-pod template injection issue.

#### 2. Install it

```bash
# confirm the manifest exists for that tag (expect HTTP/2 200)
curl -sI "https://raw.githubusercontent.com/rancher/local-path-provisioner/${LP_VERSION}/deploy/local-path-storage.yaml" | head -1

kubectl apply -f "https://raw.githubusercontent.com/rancher/local-path-provisioner/${LP_VERSION}/deploy/local-path-storage.yaml"

kubectl get storageclass local-path
kubectl get pods -n local-path-storage     # should reach 1/1 Running
```

#### 3. Make it the default storage class (recommended)

The upstream manifest does **not** mark it as default, so a PVC without an explicit `storageClassName: local-path` would stay `Pending`. To make it the default:

```bash
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

kubectl get storageclass     # local-path should show "(default)"
```

#### 4. Test it

The storage class binds volumes only when a pod uses them, so a PVC alone shows `Pending` until a pod claims it. This is normal. Test with a PVC and a pod together:

```bash
kubectl create namespace storage-test

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
  namespace: storage-test
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: local-path
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: storage-test
spec:
  restartPolicy: Never
  containers:
    - name: busybox
      image: busybox:1.36
      command: ["sh", "-c", "echo hello > /data/test.txt && cat /data/test.txt && sleep 20"]
      volumeMounts:
        - {name: data, mountPath: /data}
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: test-pvc
EOF

kubectl get pvc,pod -n storage-test     # PVC becomes Bound, pod runs
kubectl logs -n storage-test test-pod   # prints: hello

kubectl delete namespace storage-test   # clean up
```

### Things to know

- **Data lives on the server's disk**, under `/opt/local-path-provisioner` by default. It is tied to this one node, so include that path in your backups.
- **Deleting a PVC deletes its data** (the default reclaim policy is `Delete`). Take a backup before removing a database's PVC.
- **Size limits are not enforced.** The `1Gi` in a PVC is only a request, so one app can fill the whole disk. Keep an eye on disk usage with `df -h /opt`.
- **Single node only.** A volume is bound to the node that created it. That's fine here, but it would matter if you add more nodes later.
- **Upgrading:** re-run step 2 with a newer `LP_VERSION`. Existing volumes are kept.

### Troubleshooting

| Symptom                                            | Cause                                                                    | Fix                                                                        |
| -------------------------------------------------- | ------------------------------------------------------------------------ | -------------------------------------------------------------------------- |
| PVC stays `Pending` with no pod using it           | Expected: volumes bind when a pod claims them                            | Create the pod, then check again                                           |
| PVC stays `Pending` even with a pod                | No storage class set and `local-path` isn't the default                  | Add `storageClassName: local-path`, or do step 3                           |
| `local-path-provisioner` pod in `CrashLoopBackOff` | Check `kubectl logs -n local-path-storage deploy/local-path-provisioner` | The log names the cause, often a failed image pull or a networking problem |
| `curl` for the release tag prints nothing          | GitHub API rate limit or no network                                      | Wait and retry, or set `LP_VERSION` by hand from the releases page         |

### 15d. ServiceAccount, Role, and RoleBinding (restricted to this namespace)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deployer
  namespace: $NS
---
apiVersion: v1
kind: Secret
metadata:
  name: deployer-token
  namespace: $NS
  annotations:
    kubernetes.io/service-account.name: deployer
type: kubernetes.io/service-account-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: $NS
rules:
  - apiGroups: ["", "apps", "networking.k8s.io"]
    resources: [deployments, replicasets, pods, pods/log, services, ingresses, events, persistentvolumeclaims]
    verbs: [get, list, watch, create, update, patch]
  - apiGroups: ["batch"]
    resources: [cronjobs, jobs]
    verbs: [get, list, watch, create, update, patch]
  - apiGroups: ["cert-manager.io"]
    resources: [certificates]
    verbs: [get, list, watch]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployer
  namespace: $NS
subjects:
  - kind: ServiceAccount
    name: deployer
    namespace: $NS
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
EOF
```

A `Role` (as opposed to a `ClusterRole`) only grants rights inside its own namespace, so the staging account has no access to production, and neither can read Secrets. It also cannot delete anything. If you want it to be able to delete resources, add `delete` to the verbs.

### 15d. Linux deploy user with a kubeconfig for that ServiceAccount

```bash
# wait until the token is populated (usually instant)
until TOKEN=$(kubectl -n $NS get secret deployer-token -o jsonpath='{.data.token}' | base64 -d) && [ -n "$TOKEN" ]; do sleep 1; done
CA=$(kubectl -n $NS get secret deployer-token -o jsonpath='{.data.ca\.crt}')

id $DEPLOY_USER >/dev/null 2>&1 || sudo useradd -m -s /bin/bash $DEPLOY_USER
sudo mkdir -p /home/$DEPLOY_USER/.kube /home/$DEPLOY_USER/.ssh

sudo tee /home/$DEPLOY_USER/.kube/config >/dev/null <<EOF
apiVersion: v1
kind: Config
clusters:
- name: k3s
  cluster:
    server: https://127.0.0.1:6443
    certificate-authority-data: $CA
users:
- name: deployer
  user:
    token: $TOKEN
contexts:
- name: default
  context:
    cluster: k3s
    user: deployer
    namespace: $NS
current-context: default
EOF

sudo chown -R $DEPLOY_USER:$DEPLOY_USER /home/$DEPLOY_USER
sudo chmod 700 /home/$DEPLOY_USER/.ssh
sudo chmod 600 /home/$DEPLOY_USER/.kube/config
```

The API address is `127.0.0.1` because the deploy user connects from the server itself over SSH, so **port 6443 never has to be exposed to the internet** for CI/CD.

---

## Step 16: Create the SSH key for this environment

Use a separate key pair per environment, so a leaked staging key cannot reach production.

**On your laptop:**

```bash
ssh-keygen -t ed25519 -f gha_deploy_<ENV> -N "" -C "github-actions-<ENV>"
# creates gha_deploy_<ENV> (private) and gha_deploy_<ENV>.pub (public)
cat gha_deploy_<ENV>.pub
```

Replace `<ENV>` with `staging` or `production`.

**On the server**, install the public key. Paste the **entire** line printed above between the quotes. The `restrict` prefix disables port forwarding, agent forwarding, X11, and terminals for this key, so it can only run commands:

```bash
PUBKEY='<PASTE_THE_FULL_CONTENTS_OF_THE_.pub_FILE_HERE>'

echo "restrict $PUBKEY" | sudo tee /home/$DEPLOY_USER/.ssh/authorized_keys >/dev/null
sudo chown $DEPLOY_USER:$DEPLOY_USER /home/$DEPLOY_USER/.ssh/authorized_keys
sudo chmod 600 /home/$DEPLOY_USER/.ssh/authorized_keys
```

**Test from your laptop.** The single quotes matter, so `$HOME` is expanded on the server:

```bash
ssh -i gha_deploy_<ENV> deploy-<ENV>@<YOUR_SERVER_IP> \
  'KUBECONFIG=$HOME/.kube/config kubectl get pods'
```

Expected: `No resources found in houston-<ENV> namespace.`

Then confirm the restriction works. This must be **forbidden**:

```bash
ssh -i gha_deploy_<ENV> deploy-<ENV>@<YOUR_SERVER_IP> \
  'KUBECONFIG=$HOME/.kube/config kubectl get pods -n kube-system'
```

Expected: `Error from server (Forbidden)`. If it works instead, the user is reaching the admin kubeconfig. Re-check the `K3S_KUBECONFIG_MODE` note in Step 1.

Once the private key is stored in GitHub (Step 17), **delete it from your laptop**.

**Repeat Steps 15-16 for the other environment** (set `APP_ENV`/`APP_HOST` for production). Step 20 (app secrets) is also done once per environment, after the manifest is in place.

---

## Step 17: Configure GitHub secrets, variables, and Environments

Shared values live at the repo level. Anything environment-specific lives in a GitHub **Environment**, so the production key can only be used by jobs running against `production`.

**Settings → Secrets and variables → Actions** (repo level, shared):

| Type     | Name            | Value                  |
| -------- | --------------- | ---------------------- |
| Secret   | `REGISTRY_USER` | your registry username |
| Secret   | `REGISTRY_PASS` | your registry password |
| Variable | `REGISTRY_HOST` | `<REGISTRY_HOST>`      |
| Variable | `SSH_HOST`      | `<YOUR_SERVER_IP>`     |

**Settings → Environments**: create `staging` and `production`, each with:

| Type     | Name             | Staging                                             | Production                          |
| -------- | ---------------- | --------------------------------------------------- | ----------------------------------- |
| Secret   | `DEPLOY_SSH_KEY` | contents of `gha_deploy_staging`                    | contents of `gha_deploy_production` |
| Variable | `DEPLOY_USER`    | `deploy-staging`                                    | `deploy-production`                 |
| Variable | `APP_HOST`       | `<STAGING_DOMAIN>`                                  | `<PRODUCTION_DOMAIN>`               |
| Variable | `SSL_REDIRECT`   | `false` (set to `true` after the first certificate) | `false` (same)                      |
| Variable | `REPLICAS`       | `1`                                                 | `2`                                 |

Recommended protection rules on each environment:

- **Deployment branches:** `staging` allows only `dev`; `production` allows only `main`. This stops any other branch from using the production key.
- **Required reviewers** on `production`: each production deploy waits for your approval.

**Optional: do all of this with the GitHub CLI** (run inside the repo on your laptop, with `gh auth login` done):

```bash
gh secret set REGISTRY_USER --body "<REGISTRY_USERNAME>"
gh secret set REGISTRY_PASS --body '<REGISTRY_PASSWORD>'
gh variable set REGISTRY_HOST --body "<REGISTRY_HOST>"
gh variable set SSH_HOST --body "<YOUR_SERVER_IP>"

for E in staging production; do
  gh api -X PUT "repos/<OWNER>/<REPO>/environments/$E" >/dev/null
  gh secret set DEPLOY_SSH_KEY --env $E < gha_deploy_$E
  gh variable set DEPLOY_USER  --env $E --body "deploy-$E"
  gh variable set SSL_REDIRECT --env $E --body "false"
done

gh variable set APP_HOST --env staging    --body "<STAGING_DOMAIN>"
gh variable set APP_HOST --env production --body "<PRODUCTION_DOMAIN>"
gh variable set REPLICAS --env staging    --body "1"
gh variable set REPLICAS --env production --body "2"

shred -u gha_deploy_staging gha_deploy_production    # remove the private keys from your laptop
```

---

## Step 18: Add the Dockerfile

Your repo needs a `Dockerfile` at its root. This is an example for a Node app listening on port 3000; adapt it to your stack:

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
ENV PORT=3000
EXPOSE 3000
CMD ["node", "server.js"]
```

Set `APP_PORT` in the workflow (Step 21) to whatever port your app listens on.

---

## Step 19: Add the Kubernetes manifest to your repo

Create `k8s/app.yaml`. One file serves both environments: the workflow fills in the `${...}` placeholders. Every resource states its namespace explicitly (`namespace: ${NAMESPACE}`), so you can see where it will be deployed just by reading the manifest and the workflow, without checking the server. The value is set in one visible place, the `env:` block of the workflow in Step 21 (`houston-staging` for `dev`, `houston-production` for `main`).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: houston-web
  namespace: ${NAMESPACE}
spec:
  replicas: ${REPLICAS}
  selector:
    matchLabels:
      app: houston-web
  template:
    metadata:
      labels:
        app: houston-web
    spec:
      imagePullSecrets:
        - name: regcred
      containers:
        - name: web
          image: ${IMAGE_REF}
          envFrom:
            - secretRef:
                name: app-env
          ports:
            - containerPort: ${APP_PORT}
          readinessProbe:
            tcpSocket:
              port: ${APP_PORT}
            initialDelaySeconds: 5
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: houston-web
  namespace: ${NAMESPACE}
spec:
  selector:
    app: houston-web
  ports:
    - port: 80
      targetPort: ${APP_PORT}
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: houston-web
  namespace: ${NAMESPACE}
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
    acme.cert-manager.io/http01-edit-in-place: "true"
    ingress.kubernetes.io/ssl-redirect: "${SSL_REDIRECT}"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [${APP_HOST}]
      secretName: houston-web-tls
  rules:
    - host: ${APP_HOST}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: houston-web
                port:
                  number: 80
```

Notes:

- The TLS secret has the same name in both namespaces. That's fine, because secrets are namespace-scoped.
- `envFrom` loads every key of the `app-env` Secret as an environment variable. You create that Secret by hand in Step 20, **before the first deploy**: pods stay in `CreateContainerConfigError` until it exists. If your app needs no secrets, delete the `envFrom` block.
- If your app has a health endpoint, replace `tcpSocket` with `httpGet: {path: /health, port: ${APP_PORT}}`. A TCP probe only proves the port is open.
- For production, consider adding `resources:` requests and limits to the container.

---

## Step 20: Create the app secrets on the server (run once per environment)

Your app's runtime secrets (database URL, API keys, and so on) are created by hand as admin on the server. The values never pass through GitHub or the repository, and **the deploy user's Role does not change**. The deploy user only creates a Deployment that refers to the Secret by name, and the kubelet mounts it when the pod starts, the same way it already uses `regcred`.

### 20a. Create the secret

Set the namespace for the environment you're configuring (the same `NS` as in Step 15), then create the secret. Include **all** keys every time you run this, because it replaces the secret's contents:

```bash
source ~/cluster.env

export NS=houston-staging          # or houston-production

kubectl -n $NS create secret generic app-env \
  --from-literal=DATABASE_URL='<VALUE>' \
  --from-literal=API_KEY='<VALUE>' \
  --dry-run=client -o yaml | kubectl apply -f -
```

The `--dry-run ... | kubectl apply` pattern works whether or not the secret already exists. Use single quotes around values so characters like `$` and `!` aren't interpreted by the shell.

If you have many values, load them from a `KEY=VALUE` file instead (no quotes around the values), then delete the file:

```bash
kubectl -n $NS create secret generic app-env \
  --from-env-file=./<ENV>.env \
  --dry-run=client -o yaml | kubectl apply -f -

shred -u ./<ENV>.env
```

Repeat 20a with the other environment's `NS` and its own values. The two environments should never share credentials, so use a separate database and separate API keys for production.

### 20b. Verify (shows key names and sizes, not values)

```bash
kubectl -n $NS describe secret app-env
```

### 20c. Changing a value later

Environment variables from a Secret are read only when a pod starts, so editing the secret does not affect running pods. Re-run 20a with the updated values, then restart:

```bash
kubectl -n $NS rollout restart deployment/houston-web
```

A normal deploy from the workflow also restarts the pods (every deploy uses a new image tag), so new values are picked up then as well.

### 20d. Troubleshooting

If a pod shows `CreateContainerConfigError`, run `kubectl describe pod -n $NS <pod-name>`:

| Message                         | Fix                                                                                                         |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `secret "app-env" not found`    | Create it (20a) in **this** namespace                                                                       |
| `couldn't find key X in Secret` | The key names in the secret don't match what the manifest or app expects. Re-run 20a with the correct names |

> **Security note:** the deploy user can't read Secrets directly, but because it can create Deployments in its namespace, it could in principle run a pod that mounts any Secret in that namespace. Within one environment, treat the deploy key as having access to that environment's secrets. The separate Role per namespace still guarantees that the staging key can never reach production's secrets, which is the boundary that matters most.

---

## Step 21: Add the GitHub Actions workflow

Create `.github/workflows/deploy.yml`. One workflow handles both branches: `dev` deploys to `staging`, `main` deploys to `production`. Check the current major versions of the `actions/*` and `docker/*` actions when you add this.

```yaml
name: Build and deploy

on:
  push:
    branches: [dev, main]
  workflow_dispatch:

concurrency:
  group: deploy-${{ github.ref_name }}
  cancel-in-progress: false

env:
  APP_PORT: "3000" # the port your app listens on
  IMAGE_NAME: houston-web
  ENV_NAME: ${{ github.ref_name == 'main' && 'production' || 'staging' }}
  NAMESPACE: ${{ github.ref_name == 'main' && 'houston-production' || 'houston-staging' }}

jobs:
  build:
    runs-on: ubuntu-latest
    environment: ${{ github.ref_name == 'main' && 'production' || 'staging' }}
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ${{ vars.REGISTRY_HOST }}
          username: ${{ secrets.REGISTRY_USER }}
          password: ${{ secrets.REGISTRY_PASS }}

      - uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64
          push: true
          tags: |
            ${{ vars.REGISTRY_HOST }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ vars.REGISTRY_HOST }}/${{ env.IMAGE_NAME }}:${{ env.ENV_NAME }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: ${{ github.ref_name == 'main' && 'production' || 'staging' }}
    steps:
      - uses: actions/checkout@v4

      - name: Configure SSH
        env:
          SSH_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
        run: |
          install -m 700 -d ~/.ssh
          printf '%s\n' "$SSH_KEY" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan -H "${{ vars.SSH_HOST }}" >> ~/.ssh/known_hosts

      - name: Apply manifests
        env:
          IMAGE_REF: ${{ vars.REGISTRY_HOST }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          APP_HOST: ${{ vars.APP_HOST }}
          SSL_REDIRECT: ${{ vars.SSL_REDIRECT }}
          REPLICAS: ${{ vars.REPLICAS }}
        run: |
          envsubst '${IMAGE_REF} ${APP_HOST} ${SSL_REDIRECT} ${APP_PORT} ${REPLICAS} ${NAMESPACE}' < k8s/app.yaml \
            | ssh "${{ vars.DEPLOY_USER }}@${{ vars.SSH_HOST }}" \
                'KUBECONFIG=$HOME/.kube/config kubectl apply -f -'

      - name: Wait for rollout
        run: |
          ssh "${{ vars.DEPLOY_USER }}@${{ vars.SSH_HOST }}" \
            'KUBECONFIG=$HOME/.kube/config kubectl rollout status deployment/houston-web -n ${{ env.NAMESPACE }} --timeout=180s'

      - name: Show cluster state on failure
        if: failure()
        run: |
          ssh "${{ vars.DEPLOY_USER }}@${{ vars.SSH_HOST }}" \
            'export KUBECONFIG=$HOME/.kube/config; kubectl get pods,ingress,certificate -n ${{ env.NAMESPACE }}; kubectl get events -n ${{ env.NAMESPACE }} --sort-by=.lastTimestamp | tail -20'
```

How it works:

- Every image is tagged with the **commit SHA** (so a deploy is traceable and rollbacks are exact) and with a moving `staging` or `production` tag.
- The target namespace is stated in three places you can read without logging in to the server: the `NAMESPACE` line in the workflow's `env:` block, `namespace: ${NAMESPACE}` on every resource in `k8s/app.yaml`, and the `-n` flag on the rollout and diagnostic commands. It also acts as a safety check: the deploy user's Role only covers its own namespace, so if the two ever disagree (for example, a typo that points staging at `houston-production`), the apply fails with `Forbidden` instead of deploying to the wrong place.
- The `environment:` line selects which GitHub Environment's secrets and variables the job sees, so the `main` branch uses the production key and the `dev` branch uses the staging key.
- Only the SSH key for the matching environment is ever loaded into a job.
- `concurrency` prevents two deploys of the same branch from overlapping.

Commit both files (`k8s/app.yaml`, `.github/workflows/deploy.yml`) and the Dockerfile, then push to `dev`.

---

## Step 22: First deploy and enabling the HTTPS redirect

The first deploy of each environment needs the redirect **off** (`SSL_REDIRECT=false`), for the same reason as the registry.

Before you push, make sure Step 20 (the `app-env` secret) is done for that environment. Otherwise the pods stay in `CreateContainerConfigError`.

**1.** Push to `dev` and watch the run in the Actions tab. When it finishes, watch the certificate on the server:

```bash
kubectl get certificate -n houston-staging -w     # READY becomes True in 1-2 minutes
```

**2.** Once it's `True`, turn the redirect on by changing the variable (not just the cluster, or the next deploy would undo it), then re-run the workflow:

```bash
gh variable set SSL_REDIRECT --env staging --body true
gh workflow run deploy.yml --ref dev
```

(You can also change the variable in the GitHub UI and use "Re-run all jobs".)

**3.** Open `https://<STAGING_DOMAIN>`. `http://` should now redirect to `https://`.

**Production:** merge `dev` into `main`, approve the deployment if you added required reviewers, wait for `kubectl get certificate -n houston-production` to show `True`, then repeat with `--env production` and `--ref main`.

---

## Step 23: Day-to-day operations

Run these on the server as root.

```bash
# status
kubectl get pods,ingress,certificate -n houston-staging

# logs
kubectl logs -n houston-staging deploy/houston-web --tail=50

# which image is running
kubectl get deploy houston-web -n houston-staging -o jsonpath='{.spec.template.spec.containers[0].image}'

# rollback to the previous version
kubectl rollout undo deployment/houston-web -n houston-staging

# restart without changing the image
kubectl rollout restart deployment/houston-web -n houston-staging
```

**Adding another environment** (for example `qa`): repeat Steps 14-17 and 20 with a new `APP_ENV`/`APP_HOST`, and add a branch mapping in the workflow.

**Rotating a deploy key:** generate a new key pair (Step 16), replace the line in `/home/deploy-<ENV>/.ssh/authorized_keys`, update the `DEPLOY_SSH_KEY` environment secret, and delete the old private key.

**Rotating the ServiceAccount token:** delete the secret (`kubectl -n $NS delete secret deployer-token`), re-apply the manifest from Step 15c, and rebuild `/home/$DEPLOY_USER/.kube/config` as in Step 15d.

**App configuration and secrets:** don't commit them. They live in the `app-env` Secret created by hand in Step 20. To change a value, re-run Step 20a for that environment and restart with `kubectl rollout restart deployment/houston-web -n <NAMESPACE>`.

---

# Troubleshooting

| Symptom                                                                    | Cause                                                                                                            | Fix                                                                                                                                                                           |
| -------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Node `NotReady`, `cni plugin not initialized`                              | No CNI installed                                                                                                 | Install Cilium (Step 4)                                                                                                                                                       |
| DaemonSet `DESIRED 0`                                                      | Node tainted `not-ready`                                                                                         | Fix the node first; the taint clears once it's Ready                                                                                                                          |
| `LoadBalancer` service stuck `<pending>`                                   | `servicelb` is disabled                                                                                          | Use `hostPort` + `ClusterIP` as in Step 5                                                                                                                                     |
| Helm error: `IngressClass "nginx" exists`                                  | Leftover from an old ingress-nginx manifest install                                                              | `kubectl delete -f` the old manifest, then re-run Helm                                                                                                                        |
| ClusterIssuer `READY False`: `forbidden domain "example.com"`              | Placeholder email                                                                                                | Re-apply the issuer with a real `LE_EMAIL`                                                                                                                                    |
| Certificate `READY False`, Challenge `pending`: `tls: unrecognized name`   | HTTP-to-HTTPS redirect is on before a certificate exists                                                         | Set `ssl-redirect` to `"false"` and retry                                                                                                                                     |
| Challenge `pending`, two Ingresses on one host (`cm-acme-http-solver-...`) | Controller allows one Ingress per host                                                                           | Add `acme.cert-manager.io/http01-edit-in-place: "true"`                                                                                                                       |
| `502 Bad Gateway` from the registry URL                                    | Registry Deployment/Service missing or pod not ready                                                             | Complete Step 9; check `kubectl get pods,svc,endpoints -n registry`                                                                                                           |
| `413 Request Entity Too Large` on push                                     | Body size limit                                                                                                  | Keep `nginx.org/client-max-body-size: "0"` on the Ingress                                                                                                                     |
| Pod `CreateContainerConfigError`, `secret "app-env" not found`             | App secrets weren't created in this namespace                                                                    | Run Step 20 for the environment, then `kubectl rollout restart deployment/houston-web -n <NAMESPACE>`                                                                         |
| `ImagePullBackOff`, `401 unauthorized`                                     | Wrong or missing pull secret                                                                                     | Create `regcred` in the **same namespace** as the pod (Step 15b)                                                                                                              |
| `ImagePullBackOff`, `no match for platform`                                | Image built for ARM                                                                                              | Build with `platforms: linux/amd64` (already set in the workflow)                                                                                                             |
| Workflow: `Permission denied (publickey)`                                  | Wrong key, wrong user, or bad `authorized_keys`                                                                  | Check `DEPLOY_SSH_KEY` and `DEPLOY_USER` in the Environment; check file owner and mode of `authorized_keys`                                                                   |
| Workflow: `Forbidden` from kubectl                                         | Role is missing a resource or verb, or the `NAMESPACE` in the workflow doesn't match the deploy user's namespace | Check that `NAMESPACE` in the workflow equals the `NS` you used in Step 15 (`houston-<env>`); otherwise add the missing resource or verb to the Role in Step 15c and re-apply |
| Workflow: `namespaces "..." not found`                                     | The namespace in the workflow doesn't exist on the server                                                        | Create it with Step 15b, or fix the `NAMESPACE` value                                                                                                                         |
| Workflow: `The connection to the server localhost:8080 was refused`        | kubectl didn't find the kubeconfig                                                                               | Make sure the command sets `KUBECONFIG=$HOME/.kube/config` in single quotes                                                                                                   |
| Workflow: `Host key verification failed`                                   | `SSH_HOST` is wrong or the server was reinstalled                                                                | Check the `SSH_HOST` variable                                                                                                                                                 |
| Deploy user can read other namespaces                                      | Admin kubeconfig is world-readable                                                                               | Apply the `600` fix from Step 1                                                                                                                                               |
| `curl` works on the server but not from outside                            | Provider firewall                                                                                                | Open ports 80/443 in the hosting panel                                                                                                                                        |
| Re-deploy during certificate issuance breaks the challenge                 | Re-apply overwrites the path cert-manager added to the Ingress                                                   | Wait for the certificate to become `READY True` before deploying again                                                                                                        |

### Retrying a stuck certificate

Failed orders back off with increasing delays. After fixing the cause, clear everything and let the Ingress recreate it (use the right namespace):

```bash
kubectl delete challenge,order,certificaterequest --all -n <NAMESPACE>
kubectl delete certificate <TLS_SECRET_NAME> -n <NAMESPACE>
kubectl get certificate -n <NAMESPACE> -w
```

### Useful diagnostics

```bash
kubectl describe challenge -n <NAMESPACE> | grep -iE 'reason|state'     # why HTTP-01 is stuck
kubectl get ingress -n <NAMESPACE>                                      # should show ONE Ingress per host
kubectl describe clusterissuer letsencrypt | tail -15                   # issuer problems
kubectl logs -n registry deploy/registry --tail=30                      # registry pod
kubectl logs -n nginx-ingress daemonset/nginx-ingress-controller --tail=30   # 502 / upstream errors
```

---

# Things to remember

- **Don't remove or carelessly upgrade Cilium.** Without a CNI, the node goes NotReady again.
- **Back up** the cluster token, `/var/lib/rancher/k3s/server/token`, and `/var/lib/registry-data` (all your images live there).
- **Disk usage grows with every commit**, because each deploy pushes a new SHA-tagged image. Prune old tags occasionally by deleting them through the registry API and running garbage collection inside the registry pod.
- **Use immutable tags in manifests.** The workflow deploys by commit SHA, so a rollout always pulls exactly the image that was just built. The moving `staging` and `production` tags are for convenience only.
- **Single-node limits:** one server means no high availability. `replicas: 2` in production still gives you zero-downtime rolling updates, but not protection against the server going down.
- **Storage:** `local-storage` is disabled, so install a storage class (for example `local-path-provisioner`) before deploying anything that needs persistent volumes.
- **Optional F5 CRDs:** to use VirtualServer-style resources, apply `https://raw.githubusercontent.com/nginx/kubernetes-ingress/v5.7.1/deploy/crds.yaml`. Plain Ingress resources don't need them.
- **Secrets in shell history:** `REGISTRY_PASS` ends up in your shell history and `~/cluster.env`. That file is `chmod 600`, but consider clearing history on shared servers.
- **Delete private keys** (`gha_deploy_*`) from your laptop once they're stored in GitHub.
