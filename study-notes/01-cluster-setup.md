# Domain 1: Cluster Setup (15%)

> **Exam Weight: 15%**: This domain covers restricting traffic with NetworkPolicies, reviewing component settings against the CIS Benchmark, serving Ingress traffic over TLS, protecting node metadata and kubelet endpoints, and verifying Kubernetes binaries before you use them.

A cluster is only as safe as its first setup. Maybe the API server accepts anonymous requests, etcd trusts any client, or every pod can reach every other pod. Then later hardening has a weak base. So this domain is about the basics. Who can talk to what? How is the control plane configured? Are the binaries on your nodes the ones the Kubernetes project released? Most tasks here are short. But they touch static pod manifests and node files, so a small typo can take the control plane down. Work carefully and always verify.

## Index

1. [Restrict Cluster Access with Network Policies](#1-restrict-cluster-access-with-network-policies)
2. [Review Components with the CIS Benchmark](#2-review-components-with-the-cis-benchmark)
3. [Set Up Ingress with TLS](#3-set-up-ingress-with-tls)
4. [Protect Node Metadata and Endpoints](#4-protect-node-metadata-and-endpoints)
5. [Verify Platform Binaries Before Deploying](#5-verify-platform-binaries-before-deploying)
6. [Hands-on Checklist](#6-hands-on-checklist)
7. [References](#7-references)

---

## 1. Restrict Cluster Access with Network Policies

By default, every pod can talk to every other pod in the cluster. A NetworkPolicy changes that for the pods it selects. Once a policy selects a pod for a direction (Ingress or Egress), the pod is **isolated** in that direction. Then only the traffic that some policy allows can pass. Policies can only allow traffic, so there is no "deny" rule. They are also additive, which means the result is the union of all policies that select the pod.

NetworkPolicies are enforced by the CNI plugin. If the CNI does not support them, the objects are accepted but nothing is blocked.

### Default deny for a namespace

Start with a policy that selects every pod in the namespace (`podSelector: {}`) and lists both directions. It has no rules, so all ingress and egress traffic for pods in `prod` is blocked:

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
```

A default deny on egress also blocks DNS, because a DNS query is egress traffic to the CoreDNS pods. So add a second policy that allows DNS for every pod in the namespace. On kubeadm clusters the CoreDNS pods carry the label `k8s-app=kube-dns`:

```yaml
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

### Allow only frontend to backend

Now open exactly one path. The backend pods (`app=backend`) run in `prod` and listen on TCP 8080. The frontend pods (`app=frontend`) run in the `web` namespace. The policy below selects the backend pods and allows ingress only from frontend pods in `web`:

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

The control plane sets the label `kubernetes.io/metadata.name` on every namespace, and the label cannot be changed. So it is the safest way to select a namespace by its name.

If the `web` namespace also has a default deny, the frontend pods also need an egress rule. Each side of a connection must allow it, so this is a second policy in the second namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend
  namespace: web
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: prod
      podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 8080
```

Reply packets of an allowed connection are always allowed. So the backend does not need an egress rule to answer the frontend.

### AND vs OR in selectors

How you group the selectors changes the meaning of the rule:

| YAML shape | Meaning |
|------------|---------|
| `namespaceSelector` and `podSelector` in the **same** list item (one `-`) | AND: pods with that label **inside** the selected namespaces |
| `namespaceSelector` and `podSelector` as **separate** list items (two `-`) | OR: any pod in the selected namespaces, **or** pods with that label in the policy's own namespace |
| `podSelector` alone | Pods in the policy's own namespace only |

**Verify:**

First check how Kubernetes understood the policies. Then test from a pod with the allowed labels and from a pod without them:

```bash
kubectl -n prod get networkpolicy
kubectl -n prod describe networkpolicy backend-allow-frontend

kubectl -n web run test-allowed --rm -it --image=busybox:1.36 --labels=app=frontend --restart=Never -- wget -qO- -T 2 http://backend.prod:8080
kubectl -n web run test-denied --rm -it --image=busybox:1.36 --labels=app=other --restart=Never -- wget -qO- -T 2 http://backend.prod:8080
```

The first request should return a response. The second one should time out.

**Common mistakes:**

- Adding a dash before `podSelector`, which turns an AND into an OR and opens the backend to far more pods.
- Forgetting DNS after a default deny on egress. Then every name lookup times out, and it looks like the policy is broken.
- Testing with a pod that does not have the exact labels the policy expects.
- Writing `policyTypes: [Ingress]` and expecting egress to be blocked too.

---

## 2. Review Components with the CIS Benchmark

The CIS Kubernetes Benchmark is a list of recommended settings for the API server, controller manager, scheduler, etcd, the kubelet and general policies. **kube-bench** from Aqua Security runs these checks on a node and prints `PASS`, `FAIL`, `WARN` or `INFO` for each one. For each failed check it also prints a remediation, which tells you which file and flag to change.

kube-bench detects the Kubernetes version and picks a matching benchmark. If there is no exact match for a new version, it falls back to the closest older one. It needs root to read the control plane files.

### Run kube-bench

Run it on the node you want to check. A control plane node has the `master` and `etcd` targets, and every node has the `node` target:

```bash
kube-bench run --targets master
kube-bench run --targets etcd
kube-bench run --targets node
```

To check only a few items, pass their IDs with `--check`. To keep the output short, filter for failures:

```bash
kube-bench run --targets master --check 1.2.1,1.2.14
kube-bench run --targets master | grep FAIL
```

If you only have the release tarball (not a package), point kube-bench at the extracted config folder:

```bash
./kube-bench --config-dir "$(pwd)/cfg" --config "$(pwd)/cfg/config.yaml"
```

These are the checks you will most often fix by hand. The IDs are the same in the recent benchmark versions that kube-bench ships, but always read the ID from your own output:

| Check | Component | Expected setting | File on a kubeadm node |
|-------|-----------|------------------|------------------------|
| 1.1.1 | API server | Manifest permissions 600 or stricter | `/etc/kubernetes/manifests/kube-apiserver.yaml` |
| 1.2.1 | API server | `--anonymous-auth=false` | `/etc/kubernetes/manifests/kube-apiserver.yaml` |
| 1.2.7 / 1.2.8 | API server | `--authorization-mode` includes `Node` and `RBAC` | `/etc/kubernetes/manifests/kube-apiserver.yaml` |
| 1.2.14 | API server | `NodeRestriction` in `--enable-admission-plugins` | `/etc/kubernetes/manifests/kube-apiserver.yaml` |
| 1.2.15 | API server | `--profiling=false` | `/etc/kubernetes/manifests/kube-apiserver.yaml` |
| 2.2 | etcd | `--client-cert-auth=true` | `/etc/kubernetes/manifests/etcd.yaml` |
| 2.5 | etcd | `--peer-client-cert-auth=true` | `/etc/kubernetes/manifests/etcd.yaml` |
| 4.2.1 | kubelet | Anonymous auth disabled | `/var/lib/kubelet/config.yaml` |

### Fix the API server and etcd

The API server and etcd run as **static pods**. The kubelet watches `/etc/kubernetes/manifests/` and recreates a pod each time its file changes. So you fix them by editing the manifest, not with `kubectl edit`.

Take a backup first. Keep it outside the manifests folder, because the kubelet tries to run every file in that folder:

```bash
cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.bak
cp /etc/kubernetes/manifests/etcd.yaml /root/etcd.yaml.bak
```

Then add or change the flags in the `command` list. For the API server, it looks like this:

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --anonymous-auth=false
    - --profiling=false
    - --authorization-mode=Node,RBAC
    - --enable-admission-plugins=NodeRestriction
```

For etcd, make sure client certificates are required:

```yaml
spec:
  containers:
  - command:
    - etcd
    - --client-cert-auth=true
    - --peer-client-cert-auth=true
```

Edit only the flags the task names. Leave the other flags as they are.

> [!NOTE]
> Some clusters configure anonymous access through an `AuthenticationConfiguration` file (passed with `--authentication-config`) instead of the flag. If that file has an `anonymous` section, you cannot also set `--anonymous-auth`. In that case set `anonymous.enabled` in the file. The file can also keep anonymous access only for health endpoints such as `/livez` and `/readyz`.

> [!WARNING]
> kubeadm gives the API server pod liveness, readiness and startup probes. The kubelet sends them to `/livez` and `/readyz` without credentials. With `--anonymous-auth=false`, those probes can get `401`, and then the kubelet may restart the API server again and again. So after the change, watch the `RESTARTS` column of the kube-apiserver pod. The `AuthenticationConfiguration` file avoids this problem, because it can keep anonymous access only for the health endpoints.

**Verify:**

Wait for the pods to come back, then run the same checks again:

```bash
watch crictl ps
kubectl -n kube-system get pods
kube-bench run --targets master --check 1.2.1,1.2.15
kube-bench run --targets etcd --check 2.2
```

An unauthenticated request to the API server should now get `401 Unauthorized`:

```bash
curl -k https://localhost:6443/version
```

The kubelet check (4.2.1) is covered in section 4, because it is part of protecting node endpoints.

**Common mistakes:**

- Saving the backup inside `/etc/kubernetes/manifests/`. The kubelet then tries to run the backup file as well.
- Editing the manifest with a typo and walking away. If the pod does not come back within a minute or two, read the logs (Domain 2 shows how).
- Adding a flag a second time instead of changing the existing line. Check with `grep` that each flag appears only once.
- Forgetting that kube-bench runs per node. A worker node does not show API server checks.

---

## 3. Set Up Ingress with TLS

An Ingress can terminate TLS for a host. The certificate and key live in a Secret of type `kubernetes.io/tls` with the keys `tls.crt` and `tls.key`. The Ingress controller reads that Secret and serves HTTPS on port 443. By default, traffic from the controller to the Service is plain HTTP.

### Create the certificate and the TLS Secret

For practice, create a self-signed certificate. Set the host name in both the CN and the subjectAltName, because clients check the SAN:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=secure.example.com" \
  -addext "subjectAltName = DNS:secure.example.com"
```

Then store the pair in a Secret. The Secret must be in the same namespace as the Ingress:

```bash
kubectl -n shop create secret tls secure-tls --cert=tls.crt --key=tls.key
```

### Fix a workload that needs a missing TLS Secret

Sometimes a Deployment mounts a TLS Secret that does not exist yet. Its pods then stay in `ContainerCreating`, and the pod events say the secret was not found. First read the exact name and namespace from the pod spec, then create a Secret with that name:

```bash
kubectl -n shop describe pod -l app=web | grep -A5 Events
kubectl -n shop get deploy web -o jsonpath='{.spec.template.spec.volumes}'
kubectl -n shop create secret tls web-tls --cert=tls.crt --key=tls.key
```

The kubelet keeps retrying the mount, so the pods start on their own once the Secret exists.

### Create the Ingress

This Ingress serves `secure.example.com` over TLS with the `nginx` IngressClass and sends two paths to two Services. The host in `tls.hosts` must match the host in `rules`:

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
      - path: /frontend
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
      - path: /backend
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 80
```

Some guides add `nginx.ingress.kubernetes.io/rewrite-target: /`. Only use that annotation when the backend expects requests at `/` instead of at `/frontend`. It is not needed for TLS.

With ingress-nginx, an Ingress that has a `tls` section already redirects HTTP to HTTPS with a `308`. You can turn that off per Ingress with the annotation `nginx.ingress.kubernetes.io/ssl-redirect: "false"`.

**Verify:**

Check that the Ingress has an address and the right Secret. Then send a request with `--resolve`, which maps the host name to the controller's IP without changing DNS:

```bash
kubectl -n shop get ingress secure-ingress
kubectl -n shop describe ingress secure-ingress
kubectl -n ingress-nginx get svc

curl -k --resolve secure.example.com:443:<INGRESS_IP> https://secure.example.com/frontend
curl -kv --resolve secure.example.com:443:<INGRESS_IP> https://secure.example.com/frontend 2>&1 | grep -i "subject\|issuer"
```

If the controller is exposed through a NodePort, use the node IP and the HTTPS NodePort instead of 443. If the subject is not `CN=secure.example.com`, the controller is serving its default self-signed certificate. That means it could not use your Secret, or the host did not match.

**Common mistakes:**

- Creating the Secret in a different namespace than the Ingress.
- A typo in `secretName`, or a Secret that is not of type `kubernetes.io/tls`.
- Leaving out `tls.hosts`, or using a host that does not match `rules.host`.
- Testing with the IP in the URL. Then the TLS SNI does not match the host, and the controller serves its default certificate.

---

## 4. Protect Node Metadata and Endpoints

### Block the cloud metadata endpoint

On most clouds, every VM can reach a metadata service at `169.254.169.254`. It can return instance details and sometimes cloud credentials. Pods can reach it too, unless you block it. A NetworkPolicy with an `ipBlock` and an `except` list allows all egress except to that address:

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

Because policies are additive, this only works if no other policy for the same pods allows egress to `169.254.169.254`. For example, an "allow all egress" policy would open it again.

If a few pods do need the metadata service, give them a label and a separate policy that allows `169.254.169.254/32`. Then exclude them from the deny policy with a `podSelector` that uses `matchExpressions` and the `NotIn` operator.

**Verify:**

From a pod in the namespace, the request should time out:

```bash
kubectl -n prod run meta-test --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- -T 2 http://169.254.169.254/
```

### Know the node ports

Each node exposes a few ports. Only the components that need them should reach them, so a firewall or security group in front of the nodes should allow nothing else:

| Port | Component | Who needs it |
|------|-----------|--------------|
| 6443 | kube-apiserver | Everyone who uses the cluster |
| 2379-2380 | etcd | kube-apiserver and the other etcd members |
| 10250 | kubelet API | The node itself and the control plane |
| 10257 | kube-controller-manager | The node itself |
| 10259 | kube-scheduler | The node itself |
| 10256 | kube-proxy health | The node itself and load balancers |
| 30000-32767 | NodePort Services | Clients of those Services |

To see what really listens on a node, run:

```bash
ss -tlpn
```

### Harden the kubelet endpoint

The kubelet API on port 10250 can list pods, read logs and run commands in containers. So it must not accept anonymous requests. It must also ask the API server whether a caller is allowed. kubeadm already sets this up. Still, a task may give you a node where someone changed it.

On a kubeadm node, the kubelet reads `/var/lib/kubelet/config.yaml`. These are the settings to check:

```yaml
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

| Setting | Why |
|---------|-----|
| `authentication.anonymous.enabled: false` | Unauthenticated requests get `401 Unauthorized` |
| `authentication.webhook.enabled: true` | The kubelet checks bearer tokens with the API server (TokenReview) |
| `authorization.mode: Webhook` | The kubelet asks the API server (SubjectAccessReview) whether the caller is allowed. `AlwaysAllow` would allow every request |
| `readOnlyPort: 0` | Turns off the old unauthenticated read-only port (10255). `0` is the default |

In the config file, `anonymous.enabled` defaults to `false` and `mode` defaults to `Webhook`. The command-line flags are different, because `--anonymous-auth` defaults to `true` and `--authorization-mode` defaults to `AlwaysAllow`. A flag on the kubelet command line also overrides the config file. So check the systemd drop-in and the flag files for flags such as `--anonymous-auth=true` or `--authorization-mode=AlwaysAllow`:

```bash
ps -ef | grep kubelet
cat /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
cat /var/lib/kubelet/kubeadm-flags.env
cat /etc/default/kubelet
```

After you change anything, restart the kubelet:

```bash
systemctl daemon-reload
systemctl restart kubelet
```

**Verify:**

An anonymous request to the kubelet should now be rejected. Then confirm the CIS check passes:

```bash
curl -sk https://localhost:10250/pods
kube-bench run --targets node --check 4.2.1
```

The `curl` should print `Unauthorized`.

**Common mistakes:**

- Editing `/var/lib/kubelet/config.yaml` and forgetting to restart the kubelet.
- Fixing the config file while a command-line flag still sets the old value.
- Granting `nodes/proxy` to users. Even `get` on `nodes/proxy` lets a user run commands in any container on the node, because some kubelet endpoints use WebSockets over HTTP `GET`.
- Assuming the metadata policy works when another policy for the same pods allows all egress.

---

## 5. Verify Platform Binaries Before Deploying

A tampered `kubelet` or `kube-apiserver` binary gives an attacker full control of a node. So before you install or trust a binary, compare its hash with the hash the Kubernetes project publishes. Each binary on `dl.k8s.io` has a matching `.sha256` and `.sha512` file, and that file contains only the hash.

First find out which version you are checking:

```bash
kubectl version --client
kubelet --version
kubeadm version -o short
```

### Check a downloaded binary

Download the binary and its checksum file for the same version, OS and architecture. Then let `sha512sum` compare them:

```bash
VERSION=v1.37.2
curl -LO "https://dl.k8s.io/release/${VERSION}/bin/linux/amd64/kubelet"
curl -LO "https://dl.k8s.io/release/${VERSION}/bin/linux/amd64/kubelet.sha512"
echo "$(cat kubelet.sha512)  kubelet" | sha512sum --check
```

A good binary prints `kubelet: OK`. A bad one prints `kubelet: FAILED` and a warning. The command then exits with a non-zero code. Note the **two spaces** between the hash and the file name, because `sha512sum --check` expects that format. The same steps work with `.sha256` and `sha256sum`.

### Check a binary that is already on the node

Sometimes you are given binaries in a folder and asked which one was changed. Hash each file and compare it with the official hash for the same version, such as the `kubelet.sha512` file you downloaded above:

```bash
sha512sum /usr/bin/kubelet
cat kubelet.sha512
```

The server tarball also has a published hash, so you can check all control plane binaries at once. After you extract it, the binaries are in `kubernetes/server/bin/`:

```bash
curl -LO "https://dl.k8s.io/${VERSION}/kubernetes-server-linux-amd64.tar.gz"
curl -LO "https://dl.k8s.io/${VERSION}/kubernetes-server-linux-amd64.tar.gz.sha512"
echo "$(cat kubernetes-server-linux-amd64.tar.gz.sha512)  kubernetes-server-linux-amd64.tar.gz" | sha512sum --check
tar -xzf kubernetes-server-linux-amd64.tar.gz
sha512sum kubernetes/server/bin/kube-apiserver
```

If a binary does not match, do not run it. Delete it, or do what the task asks (for example, remove the binaries that fail the check).

### Check signatures with cosign

The release binaries are also signed with cosign keyless signing. A signature proves who built the file, not only that nobody changed it. Download the binary with its `.sig` and `.cert` files, then verify:

```bash
URL=https://dl.k8s.io/release/${VERSION}/bin/linux/amd64
curl -sSfL "${URL}/kubectl" -o kubectl
curl -sSfL "${URL}/kubectl.sig" -o kubectl.sig
curl -sSfL "${URL}/kubectl.cert" -o kubectl.cert

cosign verify-blob kubectl \
  --signature kubectl.sig \
  --certificate kubectl.cert \
  --certificate-identity krel-staging@k8s-releng-prod.iam.gserviceaccount.com \
  --certificate-oidc-issuer https://accounts.google.com
```

**Common mistakes:**

- Comparing against the hash of a different version or architecture. The hash only matches the exact same build.
- Using one space instead of two in the `echo` line, so `--check` cannot parse it.
- Mixing up the tools: a `.sha512` file must be checked with `sha512sum`, not `sha256sum`.
- Using `kubectl --version`. That flag does not exist. Use `kubectl version --client`.

---

## 6. Hands-on Checklist

- [ ] Can you write a default deny policy for a namespace and then allow DNS so name lookups still work?
- [ ] Can you allow only `app=frontend` pods from one namespace to reach `app=backend` pods in another namespace on one port?
- [ ] Can you explain the difference between a `namespaceSelector` and `podSelector` in one list item and in two list items?
- [ ] Can you run kube-bench for the master, etcd and node targets and filter the output for failures?
- [ ] Can you set `--anonymous-auth=false` and `--profiling=false` on the API server and confirm it comes back up?
- [ ] Can you set `--client-cert-auth=true` on etcd and confirm the check passes?
- [ ] Can you create a TLS Secret from a certificate and key, including one that a Deployment is waiting for?
- [ ] Can you create an Ingress that serves a host over TLS and test it with `curl --resolve`?
- [ ] Can you block pods from reaching `169.254.169.254` and prove the request times out?
- [ ] Can you disable anonymous access on the kubelet, set its authorization mode to `Webhook`, and prove that an anonymous `curl` to port 10250 fails?
- [ ] Can you check a Kubernetes binary against its `.sha512` or `.sha256` file from `dl.k8s.io`?
- [ ] Can you find which binary in a folder does not match the official release?

---

## 7. References

- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [kube-bench: running and flags](https://github.com/aquasecurity/kube-bench/blob/main/docs/flags-and-commands.md)
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
- [Static Pods](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
- [Authenticating: anonymous requests](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#anonymous-requests)
- [Ingress: TLS](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls)
- [ingress-nginx TLS/HTTPS](https://kubernetes.github.io/ingress-nginx/user-guide/tls/)
- [Ports and Protocols](https://kubernetes.io/docs/reference/networking/ports-and-protocols/)
- [Kubelet authentication/authorization](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/)
- [Install kubectl on Linux (checksum validation)](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
- [Verify Signed Kubernetes Artifacts](https://kubernetes.io/docs/tasks/administer-cluster/verify-signed-artifacts/)
