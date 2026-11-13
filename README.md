# Certified Kubernetes Security Specialist (CKS) Learning Path (2026)

![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.37-326CE5?logo=kubernetes&logoColor=white)
![Exam](https://img.shields.io/badge/CKS-Exam%20Prep-green)
![Passing Score](https://img.shields.io/badge/Passing%20Score-67%25-blue)
![Duration](https://img.shields.io/badge/Duration-2%20Hours-orange)
![License](https://img.shields.io/badge/License-All%20Rights%20Reserved-red)

This is a structured, hands-on learning path for the **Certified Kubernetes Security Specialist (CKS)** exam from the Linux Foundation and CNCF. It follows the official curriculum one domain at a time. For each topic, you get the concept, working YAML, the commands to verify it, and the mistakes you should be able to spot and fix.

---
## Hit the Star! :star:

If you are planning to use this repo for reference, please hit the star. Thanks!

---

<a id="exam-coupon"></a>

## CKS Exam Coupon (40% Off Exclusive Discount)

To save a flat 40% on CKS exam registration, use the following coupon code.
> [!IMPORTANT]
> **Coupon:** Use code **40KUBE26CT** at [kube.promo/cks](https://kube.promo/cks)

Use code **40KUBE26CT** to save up to 47% on the following bundles:

- CKA + CKAD: [kube.promo/cka-ckad](https://kube.promo/cka-ckad)
- CKA + CKS Bundle: [kube.promo/bundle](https://kube.promo/bundle)
- CKA + CKAD + CKS Exam bundle: [kube.promo/k8s-bundle](https://kube.promo/k8s-bundle)
- KCNA + KCSA + CKA + CKAD + CKS: [kube.promo/kubestronaut](https://kube.promo/kubestronaut)
---


## About the Exam

| Item | Detail |
|------|--------|
| **Format** | Online and proctored, with performance-based tasks in a Linux remote desktop |
| **Duration** | 2 hours |
| **Tasks** | 15 to 20 performance-based tasks |
| **Passing Score** | 67% |
| **Kubernetes Version** | v1.37 |
| **Prerequisite** | You must have passed the CKA before you attempt the CKS |
| **Results** | Sent to you by email within 24 hours |
| **Retake** | One retake is included |
| **Eligibility** | You have 12 months from purchase to take the exam |
| **Validity** | The certification is valid for 2 years |
| **Simulator** | Killer.sh, with 2 attempts and 36 hours of access for each one |

### Official Weightage per Domain

<details>
<summary><strong>Cluster Setup (15%)</strong></summary>

<br>

- Use Network security policies to restrict cluster level access
- Use CIS benchmark to review the security configuration of Kubernetes components (etcd, kubelet, kubedns, kubeapi)
- Properly set up Ingress objects with TLS
- Protect node metadata and endpoints
- Verify platform binaries before deploying

</details>

<details>
<summary><strong>Cluster Hardening (15%)</strong></summary>

<br>

- Use Role Based Access Controls to minimize exposure
- Exercise caution in using service accounts e.g. disable defaults, minimize permissions on newly created ones
- Restrict access to Kubernetes API
- Upgrade Kubernetes to avoid vulnerabilities

</details>

<details>
<summary><strong>System Hardening (10%)</strong></summary>

<br>

- Minimize host OS footprint (reduce attack surface)
- Using least-privilege identity and access management
- Minimize external access to the network
- Appropriately use kernel hardening tools such as AppArmor, seccomp

</details>

<details>
<summary><strong>Minimize Microservice Vulnerabilities (20%)</strong></summary>

<br>

- Use appropriate pod security standards
- Manage kubernetes secrets
- Understand and implement isolation techniques (multi-tenancy, sandboxed containers, etc.)
- Implement Pod-to-Pod encryption (Cilium, Istio)

</details>

<details>
<summary><strong>Supply Chain Security (20%)</strong></summary>

<br>

- Minimize base image footprint
- Understand your supply chain (e.g. SBOM, CI/CD, artifact repositories)
- Secure your supply chain (permitted registries, sign and validate artifacts, etc.)
- Perform static analysis of user workloads and container images (e.g. Kubesec, KubeLinter)

</details>

<details>
<summary><strong>Monitoring, Logging and Runtime Security (20%)</strong></summary>

<br>

- Perform behavioral analytics to detect malicious activities
- Detect threats within physical infrastructure, apps, networks, data, users and workloads
- Investigate and identify phases of attack and bad actors within the environment
- Ensure immutability of containers at runtime
- Use Kubernetes audit logs to monitor access

</details>

---

## The Learning Path

> **Source:** [CNCF CKS Curriculum](https://github.com/cncf/curriculum)

The weeks follow the curriculum order, because each domain builds on the one before it. You lock down the cluster first, then the nodes, then the workloads, and finally you learn to catch what still gets through.

| Week | Domain | Weight | Study Notes | What you should be able to do at the end |
|:----:|--------|:------:|:-----------:|-------------------------------------------|
| 1 | Cluster Setup | **15%** | [Notes](./study-notes/01-cluster-setup.md) | Write default-deny and allow NetworkPolicies, fix kube-bench findings on the API server, etcd and kubelet, serve an Ingress over TLS, block the metadata endpoint, and check binaries against their release hashes |
| 2 | Cluster Hardening | **15%** | [Notes](./study-notes/02-cluster-hardening.md) | Write least-privilege RBAC, turn off service account token automounting, mount short-lived projected tokens, close anonymous API access, recover a crashed API server, and upgrade nodes with kubeadm |
| 3 | System Hardening | **10%** | [Notes](./study-notes/03-system-hardening.md) | Remove unneeded services and packages, take users out of the `docker` group, close open ports such as a TCP Docker socket, and confine Pods with AppArmor and seccomp |
| 4 | Minimize Microservice Vulnerabilities | **20%** | [Notes](./study-notes/04-minimize-microservice-vulnerabilities.md) | Write secure `securityContext` settings, label namespaces for Pod Security Admission, encrypt Secrets at rest, run Pods in gVisor, and encrypt Pod traffic with Cilium and Istio |
| 5 | Supply Chain Security | **20%** | [Notes](./study-notes/05-supply-chain-security.md) | Fix an over-privileged Dockerfile, generate SPDX SBOMs with `bom` and Trivy, allow only trusted registries with ImagePolicyWebhook, and scan images and manifests with Trivy, Kubesec and KubeLinter |
| 6 | Monitoring, Logging and Runtime Security | **20%** | [Notes](./study-notes/06-monitoring-logging-runtime-security.md) | Write and override Falco rules, trace an alert back to a Pod and its Deployment, make containers immutable, and set up audit logging on the API server |
| 7 | Review | - | - | Solve the practice tasks under time pressure, and only then use your Killer.sh sessions |

> [!TIP]
> Three domains carry **20%** each: Microservice Vulnerabilities, Supply Chain Security and Runtime Security. Together they make up 60% of the exam, so plan most of your study time around them. Also, many tasks in every domain edit the API server's static pod manifest. So practise breaking it and bringing it back until you can do it without notes.

---

## 📚 CKS Exam Detailed Study Guide

### [Cluster Setup](#1-cluster-setup-15) `15%`

| # | Topic |
|---|-------|
| 1 | 🛡️ [Restricting cluster access with network policies](#restricting-cluster-access-with-network-policies) |
| 2 | 📋 [Reviewing components with the CIS benchmark](#reviewing-components-with-the-cis-benchmark) |
| 3 | 🔒 [Setting up Ingress with TLS](#setting-up-ingress-with-tls) |
| 4 | 🖥️ [Protecting node metadata and endpoints](#protecting-node-metadata-and-endpoints) |
| 5 | ✅ [Verifying platform binaries](#verifying-platform-binaries) |

---

### [Cluster Hardening](#2-cluster-hardening-15) `15%`

| # | Topic |
|---|-------|
| 1 | 👥 [Minimizing exposure with RBAC](#minimizing-exposure-with-rbac) |
| 2 | 🪪 [Using service accounts with care](#using-service-accounts-with-care) |
| 3 | 🚧 [Restricting access to the Kubernetes API](#restricting-access-to-the-kubernetes-api) |
| 4 | ⬆️ [Upgrading Kubernetes to avoid vulnerabilities](#upgrading-kubernetes-to-avoid-vulnerabilities) |

---

### [System Hardening](#3-system-hardening-10) `10%`

| # | Topic |
|---|-------|
| 1 | 🧹 [Minimizing the host OS footprint](#minimizing-the-host-os-footprint) |
| 2 | 🔑 [Using least-privilege identity and access management](#using-least-privilege-identity-and-access-management) |
| 3 | 🚪 [Minimizing external access to the network](#minimizing-external-access-to-the-network) |
| 4 | 🧱 [Using kernel hardening tools: AppArmor and seccomp](#using-kernel-hardening-tools-apparmor-and-seccomp) |

---

### [Minimize Microservice Vulnerabilities](#4-minimize-microservice-vulnerabilities-20) `20%`

| # | Topic |
|---|-------|
| 1 | 📏 [Using appropriate Pod Security Standards](#using-appropriate-pod-security-standards) |
| 2 | 🗝️ [Managing Kubernetes secrets](#managing-kubernetes-secrets) |
| 3 | 📦 [Implementing isolation techniques](#implementing-isolation-techniques) |
| 4 | 🔐 [Implementing Pod-to-Pod encryption](#implementing-pod-to-pod-encryption) |

---

### [Supply Chain Security](#5-supply-chain-security-20) `20%`

| # | Topic |
|---|-------|
| 1 | 🪶 [Minimizing the base image footprint](#minimizing-the-base-image-footprint) |
| 2 | 🧾 [Understanding your supply chain with SBOMs](#understanding-your-supply-chain-with-sboms) |
| 3 | 🏷️ [Securing your supply chain](#securing-your-supply-chain) |
| 4 | 🔍 [Performing static analysis of workloads and images](#performing-static-analysis-of-workloads-and-images) |

---

### [Monitoring, Logging and Runtime Security](#6-monitoring-logging-and-runtime-security-20) `20%`

| # | Topic |
|---|-------|
| 1 | 🦅 [Performing behavioral analytics with Falco](#performing-behavioral-analytics-with-falco) |
| 2 | 🚨 [Detecting threats and tracing them to a workload](#detecting-threats-and-tracing-them-to-a-workload) |
| 3 | 🕵️ [Investigating phases of attack](#investigating-phases-of-attack) |
| 4 | 🧊 [Ensuring immutability of containers at runtime](#ensuring-immutability-of-containers-at-runtime) |
| 5 | 📜 [Using audit logs to monitor access](#using-audit-logs-to-monitor-access) |

---

## 1. Cluster Setup (15%)

This section walks through each subtopic of the Cluster Setup domain.

### Restricting cluster access with network policies
> Default Deny : By default, every pod can talk to every other pod. A NetworkPolicy that selects all pods (`podSelector: {}`) and lists both directions with no rules blocks all traffic in the namespace. A deny on egress also blocks DNS, so add a second policy that allows port 53 to `kube-dns`.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

> Allow One Path : Policies only add allowed traffic, and the result is the union of all policies that select a pod. So after the default deny, you open exactly the path you need. When `namespaceSelector` and `podSelector` sit in the **same** `from` item, the source must match both. If you split them into two items, either one is enough, which opens far more.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: web
      podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

> Test Both Sides : Always test with one pod that has the allowed labels and one that does not. The first request should get a response, and the second one should time out. If the source namespace also has a default deny, the frontend pods need their own egress policy too.

```bash
# Allowed source
kubectl -n web run test-allowed --rm -it --image=busybox:1.36 --labels=app=frontend --restart=Never -- wget -qO- -T 2 http://backend.prod:8080

# Denied source
kubectl -n web run test-denied --rm -it --image=busybox:1.36 --labels=app=other --restart=Never -- wget -qO- -T 2 http://backend.prod:8080
```

### Reviewing components with the CIS benchmark
> kube-bench : kube-bench runs the CIS Kubernetes Benchmark checks on a node. It prints `PASS`, `FAIL`, `WARN` or `INFO` for each check, and a remediation that names the file and flag to change. It runs per node, so a worker node does not show API server checks.

```bash
# Run the checks for each target on the right node
kube-bench run --targets master
kube-bench run --targets etcd
kube-bench run --targets node

# Re-check only a few items
kube-bench run --targets master --check 1.2.1,1.2.15
```

> Fix Static Pods : The API server and etcd run as static pods. The kubelet recreates them each time their manifest in `/etc/kubernetes/manifests/` changes. So you fix them by editing that file, not with `kubectl edit`. Back up to `/root/` first, because the kubelet runs every file in the manifests folder.

```bash
cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.bak
cp /etc/kubernetes/manifests/etcd.yaml /root/etcd.yaml.bak
```

> Common CIS Fixes : Two fixes come up often. The API server should reject anonymous requests, and etcd should only accept clients with a valid certificate. Change only the flags the task names, and check with `grep` that each flag appears once.

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --anonymous-auth=false
    - --profiling=false
    - --authorization-mode=Node,RBAC
    - --enable-admission-plugins=NodeRestriction
```

```yaml
# /etc/kubernetes/manifests/etcd.yaml
spec:
  containers:
  - command:
    - etcd
    - --client-cert-auth=true
    - --peer-client-cert-auth=true
```

```bash
# Wait for the pods, then confirm anonymous requests get 401 Unauthorized
watch crictl ps
curl -k https://localhost:6443/version
```

> [!WARNING]
> kubeadm gives the API server liveness, readiness and startup probes. The kubelet sends them to `/livez` and `/readyz` without credentials. So with `--anonymous-auth=false`, the probes can get `401`, and the kubelet may restart the API server again and again. After the change, watch the `RESTARTS` column of the kube-apiserver pod.

### Setting up Ingress with TLS
> TLS Secret : An Ingress reads its certificate and key from a Secret of type `kubernetes.io/tls`. The Secret must be in the same namespace as the Ingress. The same command also fixes a Deployment whose pods are stuck in `ContainerCreating` because a TLS Secret they mount was never created.

```bash
# Self-signed certificate for practice
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=secure.example.com" \
  -addext "subjectAltName = DNS:secure.example.com"

kubectl -n shop create secret tls secure-tls --cert=tls.crt --key=tls.key
```

> Ingress Spec : The host in `tls.hosts` must match the host in `rules`. With ingress-nginx, an Ingress that has a `tls` section already redirects HTTP to HTTPS.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  namespace: shop
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure.example.com
    secretName: secure-tls
  rules:
  - host: secure.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

> Test with --resolve : Test with the host name, not the IP. Otherwise the TLS SNI does not match, and the controller serves its default certificate instead of yours.

```bash
curl -kv --resolve secure.example.com:443:<INGRESS_IP> https://secure.example.com/ 2>&1 | grep -i "subject\|issuer"
```

### Protecting node metadata and endpoints
> Metadata Endpoint : On most clouds, pods can reach the metadata service at `169.254.169.254`, which can return cloud credentials. An egress policy with an `ipBlock` and an `except` list allows all egress except that address. It only works if no other policy for the same pods allows it again.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-metadata
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32
```

> Kubelet Endpoint : The kubelet API on port 10250 can run commands in any container on the node. So it must reject anonymous requests and ask the API server whether a caller is allowed. On a kubeadm node, these settings live in `/var/lib/kubelet/config.yaml`, and you restart the kubelet after a change. In the config file, `anonymous.enabled` defaults to `false` and `mode` defaults to `Webhook`. But the command-line flags default to `--anonymous-auth=true` and `--authorization-mode=AlwaysAllow`, and a flag overrides the file. So also check the kubelet's systemd drop-in and flag files.

```yaml
# /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
readOnlyPort: 0
```

```bash
systemctl daemon-reload
systemctl restart kubelet

# An anonymous request should now print Unauthorized
curl -sk https://localhost:10250/pods
```

### Verifying platform binaries
> Checksums : A tampered `kubelet` or `kube-apiserver` gives an attacker full control of a node. Every binary on `dl.k8s.io` has a matching `.sha256` and `.sha512` file. Compare against the hash for the exact same version, OS and architecture. Note the **two spaces** between the hash and the file name.

```bash
VERSION=v1.37.2
curl -LO "https://dl.k8s.io/release/${VERSION}/bin/linux/amd64/kubelet"
curl -LO "https://dl.k8s.io/release/${VERSION}/bin/linux/amd64/kubelet.sha512"
echo "$(cat kubelet.sha512)  kubelet" | sha512sum --check

# Hash a binary that is already on the node and compare it by eye
sha512sum /usr/bin/kubelet
cat kubelet.sha512
```

---

## 2. Cluster Hardening (15%)

This section walks through each subtopic of the Cluster Hardening domain.

### Minimizing exposure with RBAC
> Least Privilege : A Role holds rules for one namespace, and a RoleBinding gives them to a user, group or service account. RBAC only adds permissions, so a subject's rights are the sum of all its bindings. The imperative commands are the fastest way to create them in the exam.

```bash
kubectl -n dev create role pod-reader --verb=get,list,watch --resource=pods,pods/log
kubectl -n dev create rolebinding pod-reader-jane --role=pod-reader --user=jane

# Test as the user without switching context
kubectl auth can-i list pods -n dev --as jane
kubectl auth can-i delete pods -n dev --as jane
```

> Risky Rights : Some rights are much stronger than they look. `*` grants everything, `list` on `secrets` returns the Secret data, `create` on pods lets a pod mount any Secret in the namespace, and `escalate`, `bind` and `impersonate` let a subject go beyond its own rights. Find bindings to strong roles and remove what is not needed.

```bash
kubectl get clusterrolebindings -o wide | grep -E "cluster-admin|system:anonymous|system:unauthenticated"
kubectl -n dev get rolebindings -o wide
kubectl -n dev edit role app-role
```

### Using service accounts with care
> Disable Automounting : By default, every pod gets a token for its service account. Most apps never call the API, so that token is just something to steal. Turn it off on the `default` ServiceAccount, or on a single pod. If both set the field, the pod wins.

```bash
kubectl -n prod patch serviceaccount default -p '{"automountServiceAccountToken": false}'
```

> Dedicated Account : When an app does need the API, give it its own ServiceAccount with only the rights it uses. Do not add rights to `default`, because every other pod in the namespace would get them too. A service account's user name for `--as` is `system:serviceaccount:<namespace>:<name>`.

```bash
kubectl -n prod create serviceaccount app-sa
kubectl -n prod create role app-pod-reader --verb=get,list --resource=pods
kubectl -n prod create rolebinding app-sa-pod-reader --role=app-pod-reader --serviceaccount=prod:app-sa
kubectl auth can-i list pods -n prod --as system:serviceaccount:prod:app-sa
```

> Projected Token : A projected volume puts a token with a set lifetime at a path you choose. Turn off the automatic token, then add a `serviceAccountToken` source. `expirationSeconds` must be at least 600, and 1200 gives a 20 minute token. The file ends up at `<mountPath>/<path>`, and the kubelet refreshes it before it expires.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: token-client
  namespace: prod
spec:
  serviceAccountName: app-sa
  automountServiceAccountToken: false
  containers:
  - name: app
    image: nginx:1.27
    volumeMounts:
    - name: sa-token
      mountPath: /var/run/secrets/tokens
      readOnly: true
  volumes:
  - name: sa-token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 1200
```

### Restricting access to the Kubernetes API
> API Server Flags : Every request passes authentication, authorization and then admission. So check three flags in `/etc/kubernetes/manifests/kube-apiserver.yaml`. Anonymous access should be off, the authorizers should be `Node,RBAC` and never `AlwaysAllow`, and `NodeRestriction` should be in the admission plugins. Some clusters use an `--authorization-config` file instead of `--authorization-mode`. You cannot combine the two, because the API server then exits at startup. So check which one the manifest uses first.

```bash
grep -E "anonymous-auth|authorization-mode|authorization-config|admission-plugins" /etc/kubernetes/manifests/kube-apiserver.yaml
```

> NodeRestriction : This plugin limits what each kubelet can change. A kubelet can only modify its own Node object and the pods bound to its node. It also cannot set labels with the `node-restriction.kubernetes.io/` or `node-role.kubernetes.io/` prefix. It works together with the `Node` authorizer.

> Admission Plugins : To enable a plugin, add it to the existing list. Do not create a second `--enable-admission-plugins` line. Plugins such as `ImagePolicyWebhook` also need a config file, which must be mounted into the pod.

```yaml
- --enable-admission-plugins=NodeRestriction,AlwaysPullImages
```

> Crashed API Server : If `kubectl` still says `connection refused` a minute or two after an edit, the new manifest has a problem. You cannot debug it with `kubectl`, so use the container runtime and the kubelet logs on the control plane node. If there is no container at all, the YAML itself is usually broken.

```bash
# Is the container restarting? Read its logs
crictl ps -a | grep kube-apiserver
crictl logs <container-id>

# Logs stay on disk even after the container is gone
tail -n 30 /var/log/pods/kube-system_kube-apiserver-*/kube-apiserver/*.log

# YAML parse errors show up in the kubelet log
journalctl -u kubelet --since "10 min ago" | grep -i apiserver

# Roll back if needed
cp /root/kube-apiserver.yaml.bak /etc/kubernetes/manifests/kube-apiserver.yaml
```

### Upgrading Kubernetes to avoid vulnerabilities
> Package Repository : With kubeadm you upgrade one minor version at a time, control plane first, then each worker. The packages come from `pkgs.k8s.io`, which has one repository per minor version. So first point every node at the v1.37 repository.

```bash
sudo sed -i 's|/v1.36/|/v1.37/|' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
apt-cache madison kubeadm
```

> Control Plane : On the first control plane node, upgrade `kubeadm`, then run `kubeadm upgrade apply`. After that, drain the node, upgrade the kubelet and kubectl, restart the kubelet and uncordon it. Use the patch release that `apt-cache madison` shows.

```bash
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.37.2-*' && \
sudo apt-mark hold kubeadm

sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.37.2
```

> Worker Node : A worker runs `kubeadm upgrade node`, not `upgrade apply`. You drain and uncordon it from the control plane. In the exam you cannot ssh from one node to another, so `exit` to the base host between nodes.

```bash
# On the worker
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.37.2-*' && \
sudo apt-mark hold kubeadm
sudo kubeadm upgrade node

# On the control plane
kubectl drain node01 --ignore-daemonsets

# On the worker
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet='1.37.2-*' kubectl='1.37.2-*' && \
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# On the control plane
kubectl uncordon node01
kubectl get nodes
```

---

## 3. System Hardening (10%)

This section walks through each subtopic of the System Hardening domain.

### Minimizing the host OS footprint
> Services and Packages : A node only needs the container runtime, the kubelet and a few system services. Anything else is attack surface. Stop and disable services you do not need, so they do not come back after a reboot. Then remove the package if the task asks for it.

```bash
systemctl list-units --type=service --state=running
systemctl disable --now vsftpd
apt purge -y vsftpd

# Find which package owns a binary
dpkg -S /usr/sbin/vsftpd
```

### Using least-privilege identity and access management
> The docker Group : Membership of the `docker` group is the same as root on the node, because the user can start a privileged container that mounts `/`. `gpasswd -d` removes one user from one group and leaves their other groups alone. Group changes apply at the next login.

```bash
getent group docker
gpasswd -d alice docker
id alice
```

> Docker Socket : The socket `/var/run/docker.sock` is owned by `root:docker` by default. `chgrp` changes the group at once, but a Docker restart can recreate the socket. On systemd hosts the `docker.socket` unit sets the group, so override it there for a lasting change. Then stop Docker and its socket and start them again, so the socket is created with the new group.

```bash
chgrp root /var/run/docker.sock
ls -l /var/run/docker.sock

# Make it permanent
systemctl edit docker.socket
```

```ini
[Socket]
SocketGroup=root
```

```bash
systemctl daemon-reload
systemctl stop docker.service docker.socket
systemctl start docker.socket docker.service
```

### Minimizing external access to the network
> Open Ports : Every listening port is a way in. Find the port and the process behind it, then stop the program properly. A killed process can be restarted by systemd, so disable its unit instead.

```bash
ss -tulpn
lsof -i :8080
ls -l /proc/<PID>/exe
systemctl status <PID>
```

> Docker on TCP : `dockerd` can listen on a TCP port such as `tcp://0.0.0.0:2375`, which gives root to anyone who can reach it. The listener is set either with `-H tcp://...` in the unit's `ExecStart` or in the `hosts` key of `/etc/docker/daemon.json`. Remove the TCP entry, keep `-H fd://`, then reload and restart.

```bash
ss -tlpn | grep dockerd
systemctl cat docker.service | grep ExecStart
cat /etc/docker/daemon.json

systemctl daemon-reload
systemctl restart docker
ss -tlpn | grep 2375
```

### Using kernel hardening tools: AppArmor and seccomp
> AppArmor Profile : A profile lists which files, capabilities and network access a program may use. Kubernetes does not load profiles for you. So load the profile with `apparmor_parser` on every node where the pod can run, or the pod stays `Pending`.

```text
#include <tunables/global>

profile k8s-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  file,

  # Deny all file writes.
  deny /** w,
}
```

```bash
apparmor_parser -q /etc/apparmor.d/k8s-deny-write
aa-status | grep k8s-deny-write
```

> Apply AppArmor : Since v1.30, you set the profile with `securityContext.appArmorProfile`, not with the old annotation. For `Localhost`, `localhostProfile` is the profile name declared inside the file, not the file path.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-apparmor
spec:
  securityContext:
    appArmorProfile:
      type: Localhost
      localhostProfile: k8s-deny-write
  containers:
  - name: hello
    image: busybox:1.36
    command: ["sh", "-c", "echo 'Hello AppArmor!' && sleep 1h"]
```

```bash
# Should print k8s-deny-write (enforce)
kubectl exec hello-apparmor -- cat /proc/1/attr/current
```

> seccomp : seccomp filters the system calls a process can make. `RuntimeDefault` uses the container runtime's profile, and the `restricted` Pod Security Standard requires it (or `Localhost`). A `Localhost` profile lives under `/var/lib/kubelet/seccomp/`, and `localhostProfile` is a path relative to that folder.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: audit-pod
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/audit.json
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 1h"]
    securityContext:
      allowPrivilegeEscalation: false
```

```bash
# Seccomp: 2 means a filter is active
kubectl exec audit-pod -- grep Seccomp /proc/1/status
```

---

## 4. Minimize Microservice Vulnerabilities (20%)

This section walks through each subtopic of the Minimize Microservice Vulnerabilities domain.

### Using appropriate Pod Security Standards
> securityContext : The `securityContext` decides which user a container runs as and which kernel privileges it gets. `runAsUser`, `runAsGroup` and `runAsNonRoot` can be set for the whole pod. `allowPrivilegeEscalation`, `readOnlyRootFilesystem` and `capabilities` only exist at the container level. Mount an `emptyDir` where the app still needs to write.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    runAsNonRoot: true
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 1h"]
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      privileged: false
      capabilities:
        drop: ["ALL"]
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

> PSA Labels : Pod Security Admission enforces the `privileged`, `baseline` and `restricted` levels per namespace through labels. `enforce` rejects pods, while `warn` and `audit` only report. Test with a server-side dry run first, because it lists the running pods that would break the rule.

```bash
kubectl label --dry-run=server --overwrite ns team-a \
  pod-security.kubernetes.io/enforce=restricted

kubectl label --overwrite ns team-a \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted
```

> Missing Pods : PSA checks pods, not Deployments. So a Deployment whose template breaks the rules is created without error but never gets pods. The reason is in the **ReplicaSet** events. Each line of the `FailedCreate` message names one field to fix.

```bash
kubectl -n team-a describe rs -l app=web
kubectl -n team-a get events --field-selector reason=FailedCreate
```

```yaml
# Pod template fields that restricted needs
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: web
    image: busybox:1.36
    command: ["sh", "-c", "sleep 1h"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
```

### Managing Kubernetes secrets
> Secrets Basics : Secret values are only base64-encoded, not encrypted. So limit `get`, `list` and `watch` on `secrets` with RBAC. A mounted volume is usually safer than environment variables, because env values can leak into logs.

```bash
kubectl -n app create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password='S3cure!pass'

kubectl -n app get secret db-creds -o jsonpath='{.data.password}' | base64 -d
```

> Encryption at Rest : By default, the API server stores Secrets in etcd as plain text. An `EncryptionConfiguration` makes it encrypt them first. The first provider encrypts new writes, so `identity: {}` goes last, which keeps old Secrets readable. The key must be 32 random bytes, base64-encoded.

```bash
head -c 32 /dev/urandom | base64
```

```yaml
# /etc/kubernetes/enc/enc.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <BASE64_ENCODED_32_BYTE_KEY>
      - identity: {}
```

> Wire It In : The API server needs the flag, a `volumeMount` and a `hostPath` volume, or it cannot see the file. After it restarts, rewrite all Secrets so the old ones are encrypted too. Then read one from etcd. An encrypted value starts with `k8s:enc:aescbc:v1:key1:`.

```yaml
# Parts of /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --encryption-provider-config=/etc/kubernetes/enc/enc.yaml
    volumeMounts:
    - name: enc
      mountPath: /etc/kubernetes/enc
      readOnly: true
  volumes:
  - name: enc
    hostPath:
      path: /etc/kubernetes/enc
      type: DirectoryOrCreate
```

```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -

ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/my-secret | hexdump -C
```

### Implementing isolation techniques
> Multi-tenancy : Kubernetes has no tenant object, so you build isolation from layers. The usual pattern is one namespace per tenant, with RBAC, a `ResourceQuota`, PSA labels and a default-deny NetworkPolicy on each one. A namespace on its own only groups objects.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
```

> Sandboxed Containers : Normal containers share the host kernel. gVisor adds a user-space kernel between the container and the host, and its runtime is called `runsc`. A RuntimeClass maps a name to that `handler`, and the pod picks it with `runtimeClassName`. The handler must already be set up in the containerd config on the node.

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
---
apiVersion: v1
kind: Pod
metadata:
  name: sandboxed
spec:
  runtimeClassName: gvisor
  containers:
  - name: nginx
    image: nginx:1.27
```

```bash
# dmesg shows gVisor's own kernel messages inside the sandbox
kubectl exec sandboxed -- dmesg
```

> CiliumNetworkPolicy : When the cluster runs Cilium, a CiliumNetworkPolicy can also **deny** traffic with `ingressDeny` and `egressDeny`. Deny rules always win over allow rules. But any policy turns on default deny for the selected pods, so pair a deny with a broad allow. Ports are strings in a CNP.

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: db-block-ports
  namespace: prod
spec:
  endpointSelector:
    matchLabels:
      app: db
  ingress:
  - fromEntities:
    - all
  ingressDeny:
  - fromEntities:
    - all
    toPorts:
    - ports:
      - port: "22"
        protocol: TCP
  egress:
  - toEntities:
    - all
  egressDeny:
  - toEntities:
    - all
    toPorts:
    - ports:
      - port: "25"
        protocol: TCP
```

### Implementing Pod-to-Pod encryption
> Cilium WireGuard : Cilium can encrypt all pod traffic between nodes with WireGuard. That traffic then leaves through `cilium_wg0` on UDP port 51871. Traffic between two pods on the same node never leaves the host, so it is not encrypted. Restart the agents after the change.

```bash
cilium upgrade --reuse-values \
  --set encryption.enabled=true \
  --set encryption.type=wireguard
kubectl -n kube-system rollout restart ds/cilium

kubectl -n kube-system exec ds/cilium -- cilium-dbg status | grep Encryption
```

> Istio Sidecars : In sidecar mode, Istio adds an `istio-proxy` container to each pod, and the proxies handle mTLS. Injection only happens when a pod is created. So label the namespace, then restart the workloads. Injected pods show `2/2` ready containers.

```bash
kubectl label namespace payments istio-injection=enabled --overwrite
kubectl -n payments rollout restart deployment
kubectl -n payments get pods
```

> STRICT mTLS : By default, Istio runs in `PERMISSIVE` mode and accepts both mTLS and plain text. A PeerAuthentication with mode `STRICT` in a namespace accepts only mTLS. So clients without a sidecar lose access.

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: payments
spec:
  mtls:
    mode: STRICT
```

---

## 5. Supply Chain Security (20%)

This section walks through each subtopic of the Supply Chain Security domain.

### Minimizing the base image footprint
> Small Images : Every package in an image is something an attacker can use and a scanner can flag. So start from a small, pinned base, and use a multi-stage build to keep compilers out of the final image. Run as a non-root user, and never put secrets in `ENV` or `ARG`.

```dockerfile
FROM golang:1.25 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app .

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

> Fix a Dockerfile : When a task gives you an over-privileged Dockerfile, look for `latest`, `USER root` or no `USER` line, extra tools such as `sudo` and `curl`, secrets in `ENV`, and `ADD` with a URL. Create the user before you switch to it, and put `USER` near the end.

```dockerfile
FROM python:3.13-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
RUN useradd --uid 10001 --no-create-home appuser
USER 10001
CMD ["python3", "app.py"]
```

### Understanding your supply chain with SBOMs
> SBOM : A Software Bill of Materials lists every package inside an image, with its version. SPDX is the format to know, and it comes as tag-value (plain text) or JSON. `bom generate` writes tag-value by default, and `--format json` switches to JSON. Trivy uses `spdx` and `spdx-json`.

```bash
bom generate --image registry.k8s.io/kube-apiserver:v1.37.0 --output /opt/sbom/kube-apiserver.spdx
trivy image --format spdx --output /opt/sbom/nginx.spdx nginx:1.27-alpine
trivy image --format spdx-json --output /opt/sbom/nginx.spdx.json nginx:1.27-alpine
```

> Find a Package : To find which container in a multi-container Deployment has a given package version, list each container's image, then check each image. After you save the SBOM, remove that container from the Deployment template, not from a running pod.

```bash
kubectl -n apps get deploy web -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"\t"}{.image}{"\n"}{end}'
trivy image --format spdx redis:7.4-alpine | grep -A3 "PackageName: libcrypto3"

# Save the SBOM, then delete that container's block and apply
bom generate --image redis:7.4-alpine --output /opt/sbom/web.spdx
kubectl -n apps get deploy web -o yaml > web.yaml
vim web.yaml
kubectl apply -f web.yaml
```

### Securing your supply chain
> ImagePolicyWebhook : This admission plugin sends each pod's images to an external webhook that allows or denies them. It needs an AdmissionConfiguration, a kubeconfig for the webhook, two API server flags and a mount for the folder. Set `defaultAllow: false`, so images are denied when the webhook is down.

```yaml
# /etc/kubernetes/policywebhook/admission_config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ImagePolicyWebhook
  configuration:
    imagePolicy:
      kubeConfigFile: /etc/kubernetes/policywebhook/kubeconf
      allowTTL: 100
      denyTTL: 50
      retryBackoff: 500
      defaultAllow: false
```

```yaml
# Flags in /etc/kubernetes/manifests/kube-apiserver.yaml
- --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
- --admission-control-config-file=/etc/kubernetes/policywebhook/admission_config.yaml
```

> Fix a Broken Setup : When the setup does not work, check each part in order. Is the plugin enabled, and is the config file path mounted? Does `kubeConfigFile` exist? Do the cluster, user and context names in the kubeconfig match, and is `server` an `https` URL? The API server logs usually name the broken part.

```yaml
# /etc/kubernetes/policywebhook/kubeconf
apiVersion: v1
kind: Config
clusters:
- name: image-checker
  cluster:
    certificate-authority: /etc/kubernetes/policywebhook/external-cert.pem
    server: https://image-checker.example.com:8443/image_policy
users:
- name: api-server
  user:
    client-certificate: /etc/kubernetes/policywebhook/apiserver-client-cert.pem
    client-key: /etc/kubernetes/policywebhook/apiserver-client-key.pem
contexts:
- name: image-checker
  context:
    cluster: image-checker
    user: api-server
current-context: image-checker
```

> ValidatingAdmissionPolicy : A built-in alternative that needs no webhook. It has been stable since v1.30. You write the rule as a CEL expression, and it does nothing until a binding uses it. Keep the trailing `/` in the prefix, so a look-alike registry does not match.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: allowed-registries
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["pods"]
  validations:
  - expression: >-
      object.spec.containers.all(c, c.image.startsWith('registry.example.com/')) &&
      (!has(object.spec.initContainers) ||
      object.spec.initContainers.all(c, c.image.startsWith('registry.example.com/')))
    message: "Images must come from registry.example.com"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: allowed-registries-prod
spec:
  policyName: allowed-registries
  validationActions: [Deny]
  matchResources:
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: prod
```

> Sign and Verify : Cosign signs an image and stores the signature in the registry. Sign by digest, so the signature refers to exact content and not to a tag that can move. Anyone with the public key can verify it.

```bash
cosign generate-key-pair
cosign sign --key cosign.key registry.example.com/team-a/app@sha256:<digest>
cosign verify --key cosign.pub registry.example.com/team-a/app@sha256:<digest>
```

### Performing static analysis of workloads and images
> Trivy Scans : Trivy compares the packages in an image with vulnerability databases. Scan the exact image reference from the pod spec, and use upper-case severity names. Trivy can also check manifests and Dockerfiles for misconfigurations.

```bash
# List the images that run in a namespace
kubectl -n apps get pods -o jsonpath='{range .items[*].spec.containers[*]}{.image}{"\n"}{end}' | sort -u

trivy image --severity HIGH,CRITICAL nginx:1.27-alpine
trivy config Dockerfile
```

> Kubesec and KubeLinter : Static analysis reads a manifest without running it. Kubesec returns JSON with a `score` and a `scoring` object. Inside `scoring`, the `critical` list shows settings that lower the score, and the `advise` list shows settings that would raise it. KubeLinter reports failed checks such as `privileged-container`, `run-as-non-root` and `latest-tag`.

```bash
kubesec scan deploy.yaml
kube-linter lint deploy.yaml
```

> Fix a Manifest : Look for `hostPID`, `hostNetwork`, `privileged: true`, `runAsUser: 0`, added capabilities, `hostPath` volumes and `latest` tags. Remove the host access and lock down the security context. Then run both scanners again, and the score should go up.

```yaml
# Fixed pod template of an over-privileged Deployment
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 101
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginxinc/nginx-unprivileged:1.27
    ports:
    - containerPort: 8080
    securityContext:
      privileged: false
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

---

## 6. Monitoring, Logging and Runtime Security (20%)

This section walks through each subtopic of the Monitoring, Logging and Runtime Security domain.

### Performing behavioral analytics with Falco
> Falco Basics : Falco runs on each node and reads system calls from the kernel. When a rule's condition matches, it writes an alert with the rule's output text. Its fields, such as `fd.name`, `container.id` and `k8s.pod.name`, tie each event to a file, a container and a pod.

> Rules Files : Falco loads `/etc/falco/falco_rules.yaml` first, then `/etc/falco/falco_rules.local.yaml`, then the files in `/etc/falco/rules.d/`. Never edit the default file. Put your rules in the local file, so they load after the defaults.

```yaml
# /etc/falco/falco_rules.local.yaml
- list: sensitive_dev_files
  items: [/dev/mem, /dev/kmem, /dev/port]

- rule: Container opened sensitive device file
  desc: A process in a container opened a sensitive device file under /dev
  condition: >
    evt.type in (open, openat, openat2)
    and container
    and fd.name in (sensitive_dev_files)
  output: >
    Sensitive device opened (file=%fd.name proc=%proc.name cmd=%proc.cmdline
    user=%user.name container_id=%container.id container_name=%container.name
    image=%container.image.repository pod=%k8s.pod.name ns=%k8s.ns.name)
  priority: WARNING
  tags: [container, filesystem]
```

> Override a Rule : To change a default rule, add an entry with the same name to the local file. List only the keys you change, and say in an `override` block whether each one is `append` or `replace`. The older `append: true` key is deprecated.

```yaml
- rule: Terminal shell in container
  output: Shell in container (time=%evt.time user=%user.name container_id=%container.id container_name=%container.name pod=%k8s.pod.name ns=%k8s.ns.name cmd=%proc.cmdline)
  priority: WARNING
  override:
    output: replace
    priority: replace
```

```bash
# Check that all rules load, then restart
falco --dry-run
systemctl restart falco
```

### Detecting threats and tracing them to a workload
> Read the Alerts : The packages install one systemd unit per driver, such as `falco-modern-bpf.service`, `falco-kmod.service` or `falco-custom.service`. Enabling one of them also creates the alias `falco.service`. The alerts go to the journal under the real unit name, so find that name first. Falco only sees its own node, so read the alerts on the node where the pod runs.

```bash
systemctl list-units | grep falco
journalctl -u falco-modern-bpf.service --since "5 min ago" | grep "Sensitive device"
```

> Container to Deployment : An alert gives you a container ID. `crictl` maps it to a pod, and the owner references lead from the pod to its ReplicaSet and then its Deployment. Scale the Deployment to 0 to stop it. Deleting only the pod does not help, because the ReplicaSet creates a new one.

```bash
crictl ps | grep 4f1c2a9e7b3d
crictl inspect 4f1c2a9e7b3d | grep -E 'io.kubernetes.pod.(name|namespace)'

kubectl -n team-b get pod mem-reader-6d5f8b7c9-x2kqp -o jsonpath='{.metadata.ownerReferences[0].name}{"\n"}'
kubectl -n team-b get rs mem-reader-6d5f8b7c9 -o jsonpath='{.metadata.ownerReferences[0].name}{"\n"}'
kubectl -n team-b scale deployment mem-reader --replicas=0
```

> File Output : If a task wants alerts in a file, turn on `file_output` in `/etc/falco/falco.yaml` with an absolute path, then restart Falco. Set `json_output: true` if the task wants JSON.

```yaml
file_output:
  enabled: true
  keep_alive: false
  filename: /var/log/falco/events.txt
```

### Investigating phases of attack
> Attack Phases : An attack moves through phases, such as initial access, privilege escalation, credential access, persistence and exfiltration. Each phase leaves traces in a different place. Falco shows what processes did, and the audit log shows what changed in the API.

> Node Tools : Use `crictl` for containers, `ps` and `/proc` for processes, `ss` for ports and `journalctl` for services. The cgroup path of a process contains its container ID, which links a host PID back to `crictl ps`. Write down the node, container, pod and owner before you act.

```bash
crictl ps -a
crictl inspect <container-id>
ps -ef --forest
ls -l /proc/<pid>/exe
cat /proc/<pid>/cgroup
ss -tnp
journalctl -u kubelet --since "30 min ago"
```

### Ensuring immutability of containers at runtime
> Read-only Root : An immutable container cannot change its own files, so an attacker cannot drop tools or edit configs. Set `readOnlyRootFilesystem: true`, and mount `emptyDir` volumes only where the app really writes. Also leave out `privileged`, because a privileged container can undo the protection.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: immutable-nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    securityContext:
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
    volumeMounts:
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
  volumes:
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
```

```bash
# The first write fails with Read-only file system, the second works
kubectl exec immutable-nginx -- touch /etc/test
kubectl exec immutable-nginx -- touch /var/cache/nginx/test
```

### Using audit logs to monitor access
> Audit Policy : The audit log records who did what to which resource, and when. Rules are checked from top to bottom, and the first match decides the level. So put specific rules first and a catch-all last. Log Secrets at `Metadata`, because `Request` would write their values into the log.

```yaml
# /etc/kubernetes/audit/policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - "RequestReceived"
rules:
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: ""
      resources: ["endpoints", "services"]
  - level: Metadata
    resources:
    - group: ""
      resources: ["secrets", "configmaps"]
  - level: RequestResponse
    namespaces: ["prod"]
    verbs: ["create", "update", "patch", "delete"]
    resources:
    - group: ""
      resources: ["pods"]
    - group: "apps"
      resources: ["deployments"]
  - level: Metadata
```

> API Server Flags : `--audit-log-path` turns on the log file, and the `maxage`, `maxbackup` and `maxsize` flags control rotation. The API server must also see the policy file (read-only) and the log folder (writable), so add both as `hostPath` volumes with mounts.

```yaml
# Flags in /etc/kubernetes/manifests/kube-apiserver.yaml
- --audit-policy-file=/etc/kubernetes/audit/policy.yaml
- --audit-log-path=/var/log/kubernetes/audit/audit.log
- --audit-log-maxage=30
- --audit-log-maxbackup=10
- --audit-log-maxsize=100
```

```yaml
# Container volumeMounts and pod volumes
volumeMounts:
- name: audit-policy
  mountPath: /etc/kubernetes/audit/policy.yaml
  readOnly: true
- name: audit-log
  mountPath: /var/log/kubernetes/audit/
  readOnly: false
volumes:
- name: audit-policy
  hostPath:
    path: /etc/kubernetes/audit/policy.yaml
    type: File
- name: audit-log
  hostPath:
    path: /var/log/kubernetes/audit/
    type: DirectoryOrCreate
```

```bash
# Who touched Secrets?
jq -c 'select(.objectRef.resource=="secrets") | {user: .user.username, verb, ns: .objectRef.namespace, name: .objectRef.name, level}' /var/log/kubernetes/audit/audit.log | tail
```

---

## 📖 Exam References

<details>
<summary><strong>🎯 Exam Tips &amp; Strategy</strong></summary>

<br>

### Quick Checks (save time)

```bash
kubectl auth can-i --list -n <ns> --as system:serviceaccount:<ns>:<sa>
kubectl get netpol,cnp -A
kubectl get ns --show-labels
kubectl describe rs -n <ns> -l app=<app>
crictl ps -a | grep kube-apiserver
journalctl -u kubelet --since "10 min ago" | grep -i apiserver
```

### Dry-run → Edit → Apply (for longer YAML)

```bash
kubectl create deployment web --image=nginx:1.27 --dry-run=client -o yaml > web.yaml
vim web.yaml
kubectl apply -f web.yaml
```

### Common Traps

| Trap | Prevention |
|------|-----------|
| The API server does not come back after a manifest edit | Check `crictl ps -a`, `crictl logs`, `/var/log/pods/` and `journalctl -u kubelet`, then compare with your backup |
| A backup left in `/etc/kubernetes/manifests/` | Always back up to `/root/` |
| A new config file is not visible to the API server | Add a `hostPath` volume and a `volumeMount` for every file it reads |
| PSA rejects pods, but the Deployment shows no error | Read the ReplicaSet events with `kubectl describe rs` |
| A default-deny policy breaks DNS | Also allow UDP and TCP 53 to `kube-dns` |
| Custom Falco rules are ignored | Put them in `falco_rules.local.yaml` or `rules.d/`, then restart Falco |
| Working on the wrong host | Check the hostname in your prompt before every change |

### Last-Minute Checklist

- [ ] Write a default-deny NetworkPolicy and an AND-style `namespaceSelector` + `podSelector` allow rule
- [ ] Fix `--anonymous-auth`, etcd `--client-cert-auth` and the kubelet config from memory
- [ ] Mount a projected service account token with a set `expirationSeconds`
- [ ] Wire a config file into the API server with the flag, mount and volume
- [ ] Fix a Deployment to meet the `restricted` Pod Security Standard
- [ ] Generate an SPDX SBOM with `bom` and with Trivy
- [ ] Override a Falco rule's output and map an alert to its Deployment
- [ ] Write an audit policy with the rules in the right order

> 📄 Full guide: [EXAM_GUIDE.md](./EXAM_GUIDE.md)

</details>

---

<details>
<summary><strong>📅 Exam Day Guide</strong></summary>

<br>

### Quick Stats

| Item | Detail |
|------|--------|
| Time | 2 hours for 15 to 20 tasks |
| Environment | A Linux remote desktop with a terminal, Firefox and VSCodium |
| Task hosts | Each task names a host. Run `ssh <host>`, use `sudo -i` for root, then `exit` back to `base` |
| Tools on the hosts | `kubectl` with the `k` alias and completion, `yq`, `curl`, `wget` and `man` |
| Timer alerts | At 30, 15 and 5 minutes left. Pause Exam does not stop the timer |

### Keyboard Shortcuts

| Action | Shortcut |
|--------|----------|
| `Ctrl+W` in the remote desktop | `Ctrl+Alt+W` (**NOT** `Ctrl+W`, which closes the browser tab) |
| Copy (terminal) | `Ctrl+Shift+C` |
| Paste (terminal) | `Ctrl+Shift+V` |
| Find the mouse cursor | `Ctrl+Alt+K` |
| Insert mode in `vim` | Press `i`, because the `INSERT` key is disabled |

### Time Management

| Domain | Weight | Suggested Time |
|--------|:------:|:--------------:|
| Cluster Setup | 15% | ~18 min |
| Cluster Hardening | 15% | ~18 min |
| System Hardening | 10% | ~12 min |
| Minimize Microservice Vulnerabilities | 20% | ~24 min |
| Supply Chain Security | 20% | ~24 min |
| Monitoring, Logging and Runtime Security | 20% | ~24 min |

**Note:** Work in three passes. In the first pass (about 60 min), do the quick tasks, such as NetworkPolicies, RBAC, PSA labels and securityContext fixes. In the second pass (about 45 min), do the tasks that edit static pods or node services, such as audit logging, ImagePolicyWebhook, upgrades and Falco. Use the last 15 minutes to check every task again, and make sure the API server is running.

</details>

---

<details>
<summary><strong>❓ CKS FAQs</strong></summary>

<br>

**Q: What is the CKS certification?**

A: CKS stands for Certified Kubernetes Security Specialist. It is a hands-on exam from the CNCF and the Linux Foundation. It tests how well you can secure a Kubernetes cluster, its nodes, its workloads and its supply chain, and how you detect threats at runtime.

**Q: Do I need the CKA before the CKS?**

A: Yes. You must have taken and passed the CKA before you attempt the CKS.

**Q: How much does the CKS exam cost?**

A: The price can change, so check the official certification page below for the current amount. To save on registration, use the discount code in the [exam coupon section](#exam-coupon) at the top of this page.

**Q: What is the passing score for the CKS?**

A: You need 67% to pass, and the results are emailed to you within 24 hours.

**Q: How many questions are on the CKS exam?**

A: The exam has 15 to 20 performance-based tasks. There is no multiple choice. You solve every task in a live environment through a terminal.

**Q: Which Kubernetes version does the CKS use?**

A: The exam environment runs Kubernetes v1.37.

**Q: Is the CKS exam open book? Which docs can I use?**

A: Partly. You may use these docs during the exam:

- `kubernetes.io/docs` and `kubernetes.io/blog`
- The Falco docs at `falco.org/docs`
- The bom CLI reference at `kubernetes-sigs.github.io/bom/cli-reference`
- The etcd docs at `etcd.io/docs`
- The ingress-nginx user guide at `kubernetes.github.io/ingress-nginx`
- The Cilium docs at `docs.cilium.io`
- The Istio docs at `istio.io/latest/docs`
- The links in each task's Quick Reference box

You may also read the docs installed on the hosts under `/usr/share` and the `man` pages.

**Q: Which tools should I know for the CKS?**

A: Practise Falco, Trivy, kube-bench, bom, Kubesec, KubeLinter, AppArmor, seccomp, Cilium, Istio, `etcdctl` and `crictl`. Several of them have no docs you can open in the exam, so learn their common flags well.

**Q: Is the CKS harder than the CKA?**

A: Most people find it more demanding. It assumes all the CKA skills, then adds many security tools and node-level work in the same 2 hours. So build speed on the CKA basics first, then practise each security tool until you do not need to look it up.

**Q: Which domains carry the most weight?**

A: Minimize Microservice Vulnerabilities, Supply Chain Security, and Monitoring, Logging and Runtime Security are 20% each. Together they make up 60% of the exam.

**Q: Is there a free retake?**

A: Yes, one retake is included. You must take the exam within 12 months of purchase.

**Q: How long is the CKS valid, and how do I renew it?**

A: It is valid for 2 years. To renew it, you pass the CKS exam again before it expires. Under the CARE program, if you earn or recertify the CKS on or after June 18, 2026, your CKA expiration date is extended to match it. See the [CKA, CKAD and CKS FAQ](https://docs.linuxfoundation.org/tc-docs/certification/faq-cka-ckad-cks) for the details.

**Q: Is there a practice exam simulator?**

A: Yes. Your registration includes 2 Killer.sh attempts, and each one gives you 36 hours of access from the moment you activate it. The simulator is not included with the CKS-SINGLE offer, so check what your purchase includes.

**Q: How long does it take to prepare for the CKS?**

A: If you passed the CKA recently, plan for about 6 weeks of steady hands-on practice and a review week. The learning path above follows that plan.

</details>

---

<details>
<summary><strong>📝 Practice Questions</strong></summary>

<br>

There are 25 exam-style tasks that cover all 6 CKS domains. Try each task on your own before you open the solution.

### How to Use

1. Read the task and note every exact value, such as hosts, namespaces, users, ports and file paths.
2. Solve the task on your own cluster.
3. Open the solution and compare it with your work.
4. If you got it wrong, redo the task from scratch the next day.

After a full timed run of all 25 tasks, use this table to judge how ready you are:

| Tasks solved without the solution | Where you stand |
|:---------------------------------:|-----------------|
| 21 to 25 | Ready. Book the exam and use the simulator for final timing practice |
| 16 to 20 | Close. Redo the tasks you missed and reread the matching study notes |
| 10 to 15 | Not yet. Work through the study notes for your weakest domains, then retry |
| Under 10 | Start with the study notes and come back to these tasks afterwards |

> 📄 Full list: [PRACTICE_QUESTIONS.md](./PRACTICE_QUESTIONS.md)

</details>

---

<details>
<summary><strong>⚡ Command Cheatsheet</strong></summary>

<br>

The cheatsheet gives one-line reminders for every tool in the exam, grouped by topic:

- kubectl security checks
- Control plane flags and static pods
- CIS Benchmark with kube-bench, and platform binaries
- kubeadm upgrade
- NetworkPolicy, Cilium and Istio
- RBAC and service accounts
- Pod Security Admission
- Secrets and encryption at rest
- AppArmor, seccomp and host hardening
- Image scanning and SBOMs (`trivy`, `bom`), static analysis (`kubesec`, `kube-linter`) and image signing (`cosign`)
- Falco
- Audit logs

> 📄 Full sheet: [CHEATSHEET.md](./CHEATSHEET.md)

</details>

---

<details>
<summary><strong>🔗 Official Resources</strong></summary>

<br>

| Resource | Link |
|----------|------|
| CKS Certification Page | [training.linuxfoundation.org](https://training.linuxfoundation.org/certification/certified-kubernetes-security-specialist/) |
| Candidate Handbook | [docs.linuxfoundation.org](https://docs.linuxfoundation.org/tc-docs/certification/lf-handbook2) |
| Important Instructions: CKS | [docs.linuxfoundation.org](https://docs.linuxfoundation.org/tc-docs/certification/important-instructions-cks) |
| Resources Allowed | [docs.linuxfoundation.org](https://docs.linuxfoundation.org/tc-docs/certification/certification-resources-allowed) |
| Killer.sh Simulator | [killer.sh](https://killer.sh) |

</details>

---

<details>
<summary><strong>🤝 Contributing</strong></summary>

<br>

Contributions are welcome. You can help in these ways:

- **Corrections:** Open an issue that describes the error, with a link to the official source.
- **New content:** Open a pull request, and keep it within the CKS exam scope.
- **Typos and formatting:** Send a pull request directly, no issue needed.

> 📄 Full guide: [CONTRIBUTING.md](./CONTRIBUTING.md)

</details>

---

> ⭐ If this guide helped you, please star the repo!
