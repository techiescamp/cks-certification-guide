# CKS Practice Tasks

> This page has 25 exam-style scenarios that together cover all 6 domains. Try each task on your own before you open its solution, and time yourself so you get used to working under exam pressure.

**How to use:** Read each task carefully and write down every exact value it gives you, such as host names, namespaces, users, ports, file paths and versions, because the exam checks those values precisely. Then solve the task and compare your work with the solution. If you get a task wrong, rebuild it from scratch the next day so the fix sticks.

Every task names the host where the work must happen. Connect with `ssh <host>`, become root with `sudo -i` when you need it, and run `exit` to return to `base` when you finish. Before you edit any file on a node, copy it to `/root/` first, so you can roll back if something breaks.

**Readiness check:** Use this table to judge how ready you are after a full timed run of all 25 tasks.

| Tasks solved without the solution | Where you stand |
|:---------------------------------:|-----------------|
| 21 to 25 | Ready. Book the exam and use the simulator for final timing practice |
| 16 to 20 | Close. Redo the tasks you missed and reread the matching study notes |
| 10 to 15 | Not yet. Work through the study notes for your weakest domains, then retry |
| Under 10 | Start with the study notes and come back to these tasks afterwards |

---

## Domain 1: Cluster Setup

### Task 1: Fix CIS benchmark findings on the control plane

A CIS scan of control plane node `cp-1` reported three failures. The API server allows anonymous requests, etcd accepts clients without a certificate, and the kubelet on `cp-1` allows anonymous access and uses the `AlwaysAllow` authorization mode. Fix all three so they follow the CIS recommendations.

<details><summary>Solution</summary>

Connect to the node and back up both static pod manifests to `/root/`. Never leave a backup inside `/etc/kubernetes/manifests/`, because the kubelet runs every file in that folder as a static pod.

```bash
ssh cp-1
sudo -i
cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.bak
cp /etc/kubernetes/manifests/etcd.yaml /root/etcd.yaml.bak
cp /var/lib/kubelet/config.yaml /root/kubelet-config.yaml.bak
```

In `/etc/kubernetes/manifests/kube-apiserver.yaml`, add this flag to the `command` list, or change it if it is already there with `true`:

```yaml
- --anonymous-auth=false
```

In `/etc/kubernetes/manifests/etcd.yaml`, make sure the `command` list has this flag:

```yaml
- --client-cert-auth=true
```

In `/var/lib/kubelet/config.yaml`, set these two fields, then restart the kubelet so it reads the file again:

```yaml
authentication:
  anonymous:
    enabled: false
authorization:
  mode: Webhook
```

```bash
systemctl restart kubelet
```

Wait for both static pods to come back, then check that an anonymous request to the API server is now rejected with `401 Unauthorized`:

```bash
watch crictl ps
curl -sk https://localhost:6443/api
kube-bench run --targets master,etcd,node | grep -iE "anonymous-auth|client-cert-auth|authorization-mode"
```

The kubelet's health probes call `/livez` and `/readyz` without credentials. So after you turn anonymous auth off, those probes can get a `401` too. If the API server pod then restarts every few minutes, check the probe errors with `kubectl -n kube-system describe pod kube-apiserver-cp-1`. Then read the Anonymous authenticator section of the authentication docs.
</details>

### Task 2: Lock down the prod namespace

In namespace `prod`, block all ingress and egress traffic by default. Then allow only pods labelled `app=frontend` in namespace `web` to reach pods labelled `app=backend` in `prod` on TCP 8080. Pods in `prod` must still be able to resolve DNS names. Use one NetworkPolicy per purpose.

<details><summary>Solution</summary>

The first policy selects every pod in `prod` and lists both policy types with no rules, which denies everything:

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

The second policy lets the frontend reach the backend. Notice that `namespaceSelector` and `podSelector` sit in the **same** list item, so a source must match both. If you put a `-` in front of `podSelector`, the rule would allow any pod in `web` OR any `app=frontend` pod in `prod`, which is much wider.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
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

The third policy allows DNS, because the default deny also blocks name lookups:

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

Then test one path that must work and one that must fail:

```bash
kubectl -n web exec deploy/frontend -- curl -s -m 3 backend.prod:8080
kubectl -n default run probe --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- -T 3 backend.prod:8080
```
</details>

### Task 3: Missing TLS secret and an HTTPS Ingress

Deployment `shop-web` in namespace `shop` is stuck, because its pods mount a Secret named `shop-tls` that does not exist. The certificate and key are at `/opt/certs/shop.crt` and `/opt/certs/shop.key` on `cp-1`. Create the Secret so the pods start. Then create Ingress `shop-ingress` with ingress class `nginx` that serves `https://shop.example.local` with that Secret and sends traffic to Service `shop-web` on port 80.

<details><summary>Solution</summary>

First confirm the cause. The pod events show that the Secret is not found:

```bash
ssh cp-1
kubectl -n shop get pods
kubectl -n shop describe pod -l app=shop-web | grep -A5 Events
```

Create a Secret of type `kubernetes.io/tls` from the two files:

```bash
kubectl -n shop create secret tls shop-tls --cert=/opt/certs/shop.crt --key=/opt/certs/shop.key
kubectl -n shop get pods -w
```

Then create the Ingress. The `tls` block names the host and the Secret, and the host in `rules` must match it:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-ingress
  namespace: shop
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - shop.example.local
    secretName: shop-tls
  rules:
  - host: shop.example.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: shop-web
            port:
              number: 80
```

Finally, find the address of the ingress controller and check that it serves your certificate:

```bash
kubectl -n shop get ingress shop-ingress
curl -kv --resolve shop.example.local:443:<ingress-ip> https://shop.example.local/ 2>&1 | grep -E "subject|HTTP/"
```
</details>

### Task 4: Verify platform binaries

On `cp-1`, the folder `/opt/release/` contains three binaries: `kubelet`, `kube-apiserver` and `kubectl`. The official SHA256 checksums for this release are in `/opt/release/checksums.txt`, one line per file. Delete every binary whose checksum does not match, and leave the rest in place.

<details><summary>Solution</summary>

Calculate the hash of each file and compare it with the official list:

```bash
ssh cp-1
cd /opt/release
sha256sum kubelet kube-apiserver kubectl
cat checksums.txt
```

If `checksums.txt` uses the `<hash>  <file>` format, `sha256sum` can do the comparison for you, and it prints `OK` or `FAILED` for each file:

```bash
sha256sum --check checksums.txt
```

Delete each file that shows `FAILED`, then run the check again:

```bash
rm /opt/release/<failed-binary>
```
</details>

---

## Domain 2: Cluster Hardening

### Task 5: Upgrade a worker node

The control plane node `cp-1` runs Kubernetes `v1.37.2`, but worker node `node-1` still runs `v1.36`. Upgrade kubeadm and the kubelet on `node-1` to exactly the same version as the control plane. The node must be schedulable again when you finish.

<details><summary>Solution</summary>

First check the versions from the control plane:

```bash
ssh cp-1
kubectl get nodes
exit
```

On `node-1`, switch the package repository to the v1.37 minor release, then upgrade kubeadm and run the node upgrade:

```bash
ssh node-1
sudo -i
sed -i 's#/v1.36/#/v1.37/#' /etc/apt/sources.list.d/kubernetes.list
apt-mark unhold kubeadm kubectl
apt-get update
apt-get install -y kubeadm='1.37.2-*' kubectl='1.37.2-*'
apt-mark hold kubeadm kubectl
kubeadm upgrade node
exit
exit
```

Drain the node from the control plane, because the base machine has no `kubectl`:

```bash
ssh cp-1
kubectl drain node-1 --ignore-daemonsets
exit
```

Back on `node-1`, upgrade the kubelet and restart it:

```bash
ssh node-1
sudo -i
apt-mark unhold kubelet
apt-get update
apt-get install -y kubelet='1.37.2-*'
apt-mark hold kubelet
systemctl daemon-reload
systemctl restart kubelet
exit
exit
```

Finally, uncordon the node and confirm both nodes show `v1.37.2`:

```bash
ssh cp-1
kubectl uncordon node-1
kubectl get nodes
```
</details>

### Task 6: Short-lived service account tokens

In namespace `payments`, the ServiceAccount `pay-api` is used by Deployment `pay-api`. Stop Kubernetes from mounting the token automatically for this ServiceAccount. Then give the Deployment a projected volume named `api-token` that holds a ServiceAccount token valid for 20 minutes, mounted at `/var/run/secrets/tokens` in the container `app` with the file name `token`.

<details><summary>Solution</summary>

Turn off automounting on the ServiceAccount:

```bash
kubectl -n payments patch serviceaccount pay-api -p '{"automountServiceAccountToken": false}'
```

Then edit the Deployment with `kubectl -n payments edit deploy pay-api`. The relevant part of the pod template looks like this. 20 minutes is 1200 seconds, and the kubelet refreshes the token before it expires:

```yaml
spec:
  serviceAccountName: pay-api
  containers:
  - name: app
    image: registry.example.com/pay-api:2.4.1
    volumeMounts:
    - name: api-token
      mountPath: /var/run/secrets/tokens
      readOnly: true
  volumes:
  - name: api-token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 1200
```

After the rollout, the token must exist at the new path. The default path must not exist, so the last command must fail:

```bash
kubectl -n payments rollout status deploy pay-api
kubectl -n payments exec deploy/pay-api -c app -- ls /var/run/secrets/tokens
kubectl -n payments exec deploy/pay-api -c app -- ls /var/run/secrets/kubernetes.io/serviceaccount
```
</details>

### Task 7: Trim an over-permissive Role

In namespace `dev`, the ServiceAccount `ci-bot` is bound to Role `ci-role`, which allows every verb on every resource. Change `ci-role` so that `ci-bot` can only `get`, `list` and `watch` Pods and Deployments in `dev`. Do not change the RoleBinding.

<details><summary>Solution</summary>

Look at what the ServiceAccount can do right now:

```bash
kubectl -n dev describe role ci-role
kubectl -n dev auth can-i --list --as system:serviceaccount:dev:ci-bot
```

Generate the smaller Role and apply it over the old one. The name stays the same, so the existing RoleBinding keeps working:

```bash
kubectl -n dev create role ci-role --verb=get,list,watch --resource=pods,deployments.apps --dry-run=client -o yaml | kubectl apply -f -
```

Then check one allowed action and one denied action:

```bash
kubectl -n dev auth can-i list deployments --as system:serviceaccount:dev:ci-bot
kubectl -n dev auth can-i delete pods --as system:serviceaccount:dev:ci-bot
```
</details>

### Task 8: Bring a crashed API server back

Someone edited the API server manifest on `cp-1`, and now `kubectl` cannot reach the cluster. Find the problem and fix it. When the API server is running again, make sure the `NodeRestriction` and `AlwaysPullImages` admission plugins are both enabled.

<details><summary>Solution</summary>

`kubectl` cannot help while the API server is down, so use the container runtime and the logs on the node:

```bash
ssh cp-1
sudo -i
crictl ps -a | grep kube-apiserver
crictl logs <container-id>
ls /var/log/pods/ | grep kube-apiserver
tail -n 30 /var/log/pods/kube-system_kube-apiserver-cp-1_*/kube-apiserver/*.log
```

If `crictl` shows no container at all, the manifest probably has a YAML error, so the kubelet cannot even create the pod. In that case the kubelet log tells you why:

```bash
journalctl -u kubelet --since "10 min ago" | grep -i apiserver
```

Back up the manifest, then fix the broken line, for example a misspelled flag, a wrong file path or bad indentation. Then set the admission plugins:

```bash
cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.bak
vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

```yaml
- --enable-admission-plugins=NodeRestriction,AlwaysPullImages
```

Wait for the API server to restart, then confirm the flag is live:

```bash
watch crictl ps
ps aux | grep kube-apiserver | grep -o "enable-admission-plugins=[^ ]*"
kubectl get nodes
```
</details>

---

## Domain 3: System Hardening

### Task 9: Harden the Docker daemon

Host `build-1` runs Docker. User `tom` must no longer be able to use Docker, `/var/run/docker.sock` must be owned by group `root`, and the Docker daemon must stop listening on TCP port 2375. Docker must still be running when you finish.

<details><summary>Solution</summary>

Remove `tom` from the `docker` group:

```bash
ssh build-1
sudo -i
gpasswd -d tom docker
id tom
```

Find where the TCP listener is configured. It is usually a `-H tcp://...` flag in the systemd unit or a `hosts` entry in `daemon.json`:

```bash
ss -lntp | grep 2375
systemctl cat docker | grep ExecStart
cat /etc/docker/daemon.json
```

Back up the file that contains the TCP address, then remove only the `tcp://0.0.0.0:2375` part and keep the local socket. For example, the unit line becomes:

```text
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

Reload systemd and restart Docker:

```bash
systemctl daemon-reload
systemctl restart docker
ss -lntp | grep 2375
```

Change the socket group **after** the restart, because a restart can create the socket again:

```bash
chgrp root /var/run/docker.sock
ls -l /var/run/docker.sock
```

On many installs, the socket is created by the `docker.socket` unit, and its `SocketGroup=` setting decides the group. So if the task asks for a change that survives a restart, set `SocketGroup=root` in that unit too, then run `systemctl daemon-reload` and restart `docker.socket`.
</details>

### Task 10: Apply an AppArmor profile

On `node-1`, the file `/opt/apparmor/k8s-deny-write` contains an AppArmor profile named `k8s-deny-write`. Load it in enforce mode. Then create Pod `writer` in namespace `default` with image `busybox:1.36` and the command `sleep 3600`, running on `node-1` and confined by that profile.

<details><summary>Solution</summary>

Load the profile on the node and check that it appears in the list:

```bash
ssh node-1
sudo -i
apparmor_parser -q /opt/apparmor/k8s-deny-write
aa-status | grep k8s-deny-write
exit
exit
```

From a host with `kubectl`, create the Pod. The `localhostProfile` value is the profile **name**, not the file path:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: writer
  namespace: default
spec:
  nodeName: node-1
  securityContext:
    appArmorProfile:
      type: Localhost
      localhostProfile: k8s-deny-write
  containers:
  - name: writer
    image: busybox:1.36
    command: ["sleep", "3600"]
```

Check that the profile is active. The first command must print `k8s-deny-write (enforce)`, and the `touch` command must fail:

```bash
kubectl exec writer -- cat /proc/1/attr/current
kubectl exec writer -- touch /tmp/test
```
</details>

### Task 11: Close an unexpected port

On `node-2`, something is listening on TCP port 4444, which is not part of the approved setup. Find the process, stop it so it does not start again after a reboot, and delete its binary.

<details><summary>Solution</summary>

Find the process that owns the port and the binary behind it:

```bash
ssh node-2
sudo -i
ss -lntp | grep 4444
lsof -i :4444
ls -l /proc/<pid>/exe
```

`systemctl status` accepts a PID, and it shows which unit started that process:

```bash
systemctl status <pid>
```

Stop and disable the unit, then delete the binary and confirm the port is closed:

```bash
systemctl disable --now <unit-name>
rm <path-to-binary>
ss -lntp | grep 4444
```

If the binary came from a package, remove the package with `apt remove <package>` instead of deleting the file, so the package manager stays consistent.
</details>

---

## Domain 4: Minimize Microservice Vulnerabilities

### Task 12: Set Pod Security Admission labels

Namespace `team-a` must reject pods that break the `baseline` Pod Security Standard. At the same time, users must see a warning, and the audit log must get an entry, when a pod breaks the `restricted` standard. Always use the latest version of each standard.

<details><summary>Solution</summary>

PSA is controlled by namespace labels, one label per mode, plus an optional version label for each mode:

```bash
kubectl label ns team-a --overwrite \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/audit-version=latest
```

Check the labels. Then test with a privileged pod, which must be rejected:

```bash
kubectl get ns team-a --show-labels
kubectl -n team-a run priv --image=nginx:1.27 --privileged
```
</details>

### Task 13: Pods blocked by Pod Security Admission

Namespace `secure` enforces the `restricted` standard. Deployment `api` in that namespace shows `0/2` ready, and no pods exist. Find out why, and fix the Deployment so its pods run as user `1000`, cannot escalate privileges, and have a read-only root filesystem. The app writes temporary files to `/tmp`.

<details><summary>Solution</summary>

When admission rejects a pod, the Deployment itself looks fine, but the ReplicaSet cannot create pods. So the error is in the ReplicaSet events:

```bash
kubectl -n secure get deploy,rs
kubectl -n secure describe rs -l app=api | grep -A10 Events
```

The event lists each rule the pod breaks, for example `allowPrivilegeEscalation != false`, `unrestricted capabilities`, `runAsNonRoot != true` and `seccompProfile`. Edit the Deployment with `kubectl -n secure edit deploy api` so the pod template has all of these settings:

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: api
    image: registry.example.com/api:3.1.0
    securityContext:
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

The `emptyDir` gives the app a writable `/tmp`, because the root filesystem is now read-only. Then check that the pods start and that the settings are applied. The `id` command must show UID `1000`, and the `touch` command must fail:

```bash
kubectl -n secure rollout status deploy api
kubectl -n secure exec deploy/api -- id
kubectl -n secure exec deploy/api -- touch /test
```
</details>

### Task 14: Encrypt Secrets at rest

On `cp-1`, configure the API server to encrypt Secrets in etcd with the `aescbc` provider and a key named `key1`. Save the configuration at `/etc/kubernetes/enc/enc.yaml`. Existing Secrets must be encrypted too.

<details><summary>Solution</summary>

Generate a random 32-byte key, encoded in base64:

```bash
ssh cp-1
sudo -i
mkdir -p /etc/kubernetes/enc
head -c 32 /dev/urandom | base64
```

Write `/etc/kubernetes/enc/enc.yaml`. The order of providers matters: the first one encrypts new data, and `identity` stays last so the API server can still read Secrets that are not encrypted yet.

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-key>
  - identity: {}
```

Back up the API server manifest, then add the flag, a volume and a volume mount. Without the mount, the API server cannot see the file and fails to start.

```bash
cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.bak
```

```yaml
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

After the API server restarts, rewrite all Secrets so they are stored encrypted:

```bash
kubectl get secrets -A -o json | kubectl replace -f -
```

To prove it works, read a Secret straight from etcd. The value must start with `k8s:enc:aescbc:v1:key1` instead of plain text:

```bash
kubectl create secret generic enc-test --from-literal=pass=demo123
etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/enc-test | hexdump -C | head
```
</details>

### Task 15: Run a workload in a gVisor sandbox

The container runtime on `node-1` already has a handler named `runsc` for gVisor. Create a RuntimeClass `gvisor` that uses this handler. Then create Pod `sandboxed` in namespace `isolated` with image `nginx:1.27` that runs with this RuntimeClass on `node-1`.

<details><summary>Solution</summary>

The RuntimeClass connects a name that pods use to a handler that the runtime knows:

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
  namespace: isolated
spec:
  runtimeClassName: gvisor
  nodeName: node-1
  containers:
  - name: nginx
    image: nginx:1.27
```

Inside a gVisor sandbox, the kernel messages come from gVisor and not from the host kernel, so `dmesg` shows that it works:

```bash
kubectl -n isolated get pod sandboxed
kubectl -n isolated exec sandboxed -- dmesg
```
</details>

### Task 16: Istio sidecars and strict mTLS

Istio is installed in sidecar mode. In namespace `orders`, every pod must get an Istio sidecar, including the pods that are already running. Then require mutual TLS for all traffic to workloads in `orders`.

<details><summary>Solution</summary>

Label the namespace for automatic injection. The label only affects new pods, so restart the existing Deployments:

```bash
kubectl label namespace orders istio-injection=enabled --overwrite
kubectl -n orders rollout restart deployment
kubectl -n orders get pods
```

Each pod must now show one extra container, for example `2/2`. Then create a namespace-wide PeerAuthentication. It has no selector, so it applies to every workload in the namespace:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: orders
spec:
  mtls:
    mode: STRICT
```

Check the policy, then confirm that a client without a sidecar, in a namespace without injection, can no longer connect:

```bash
kubectl -n orders get peerauthentication
kubectl -n default run plain --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- -T 3 http://<service>.orders:<port>
```
</details>

### Task 17: Cilium port deny and WireGuard encryption

The cluster uses Cilium. In namespace `internal`, pods labelled `app=scraper` must not open connections to TCP port 25 anywhere, but all their other traffic must keep working. After that, turn on WireGuard encryption for pod traffic between nodes and prove it is active.

<details><summary>Solution</summary>

A deny rule wins over any allow rule. However, once a policy selects a pod, Cilium puts that pod in default deny mode for the direction the policy covers, which here is egress. So add an egress rule that allows everything next to the deny rule. This keeps all other traffic working:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: scraper-no-smtp
  namespace: internal
spec:
  endpointSelector:
    matchLabels:
      app: scraper
  egressDeny:
  - toEntities:
    - all
    toPorts:
    - ports:
      - port: "25"
        protocol: TCP
  egress:
  - toEntities:
    - all
```

Test that port 25 is blocked and that other traffic still works:

```bash
kubectl -n internal exec deploy/scraper -- nc -zv -w 3 <smtp-host> 25
kubectl -n internal exec deploy/scraper -- nc -zv -w 3 <web-host> 443
```

Then turn on WireGuard, restart the agents, and check the encryption status:

```bash
cilium upgrade --reuse-values --set encryption.enabled=true --set encryption.type=wireguard
kubectl -n kube-system rollout restart ds/cilium
kubectl -n kube-system exec ds/cilium -- cilium-dbg status | grep Encryption
```

The output must show `Wireguard` with the `cilium_wg0` interface.
</details>

---

## Domain 5: Supply Chain Security

### Task 18: Find a package and generate an SBOM

Deployment `catalog` in namespace `inventory` has three containers: `web`, `cache` and `worker`. One of their images contains the package `libssl3` at version `3.0.15-1~deb12u1`. Generate an SPDX SBOM in tag-value format for that image at `/opt/sbom/catalog.spdx` on `cp-1`. Then remove that container from the Deployment, and leave the other two unchanged.

<details><summary>Solution</summary>

List the container names and their images:

```bash
ssh cp-1
kubectl -n inventory get deploy catalog -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"\t"}{.image}{"\n"}{end}'
```

Generate an SBOM for each image and search it for the package. The default format of `bom generate` is SPDX tag-value, so `grep` works on it directly:

```bash
bom generate --image <image> | grep -A3 "PackageName: libssl3"
```

When you find the right image, write its SBOM to the required path. Run only one of the two commands below. Both write SPDX tag-value, because `--format spdx` in trivy is the tag-value format:

```bash
bom generate --image <image> --output /opt/sbom/catalog.spdx
trivy image --format spdx --output /opt/sbom/catalog.spdx <image>
head -5 /opt/sbom/catalog.spdx
```

Finally, open the Deployment with `kubectl -n inventory edit deploy catalog`, delete the whole entry of that container, and check the result:

```bash
kubectl -n inventory get deploy catalog -o jsonpath='{.spec.template.spec.containers[*].name}'
kubectl -n inventory rollout status deploy catalog
```
</details>

### Task 19: Fix an over-privileged Dockerfile and manifest

On `cp-1`, the folder `/opt/app/` has a `Dockerfile` and a manifest `deploy.yaml`. Fix the Dockerfile so it uses the pinned base image `ubuntu:24.04`, does not install `sudo`, keeps no password in an `ENV` line, and runs as user `appuser` with UID `10001`. Fix the manifest so the container is not privileged, cannot escalate privileges, and runs as UID `10001`. Use static analysis to check both files.

<details><summary>Solution</summary>

Look at the original files first:

```bash
ssh cp-1
cat /opt/app/Dockerfile /opt/app/deploy.yaml
```

A fixed Dockerfile looks like this. The password is gone because it would be readable by anyone who pulls the image, so it should come from a Kubernetes Secret at runtime instead:

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/* \
    && useradd -u 10001 appuser
COPY app /app
USER appuser
CMD ["/app"]
```

In `deploy.yaml`, remove `privileged: true` and any `runAsUser: 0`, then set the container `securityContext` like this:

```yaml
securityContext:
  privileged: false
  allowPrivilegeEscalation: false
  runAsNonRoot: true
  runAsUser: 10001
```

Run the static analysis tools on the manifest. Each one must no longer report the privileged settings:

```bash
kubesec scan /opt/app/deploy.yaml
kube-linter lint /opt/app/deploy.yaml
```
</details>

### Task 20: Fix the ImagePolicyWebhook setup

On `cp-1`, a teammate started to set up the `ImagePolicyWebhook` admission plugin, but it is not active. The files are in `/etc/kubernetes/admission/`: `admission-config.yaml` and `imagepolicy.kubeconfig`. The webhook backend is `https://image-checker.internal:8443/check`. Finish the setup so that the API server uses the plugin, and so that pods are rejected when the backend cannot be reached.

<details><summary>Solution</summary>

Read both files and back up everything you will change:

```bash
ssh cp-1
sudo -i
cat /etc/kubernetes/admission/admission-config.yaml /etc/kubernetes/admission/imagepolicy.kubeconfig
cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.bak
cp -r /etc/kubernetes/admission /root/admission.bak
```

The AdmissionConfiguration must point to the kubeconfig, and `defaultAllow` must be `false`, so images are denied when the backend is down:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ImagePolicyWebhook
  configuration:
    imagePolicy:
      kubeConfigFile: /etc/kubernetes/admission/imagepolicy.kubeconfig
      allowTTL: 50
      denyTTL: 50
      retryBackoff: 500
      defaultAllow: false
```

In the kubeconfig, the cluster `server` must be the backend URL, and it must use `https`:

```yaml
apiVersion: v1
kind: Config
clusters:
- name: image-checker
  cluster:
    certificate-authority: /etc/kubernetes/admission/ca.crt
    server: https://image-checker.internal:8443/check
users:
- name: api-server
  user:
    client-certificate: /etc/kubernetes/admission/client.crt
    client-key: /etc/kubernetes/admission/client.key
contexts:
- name: default
  context:
    cluster: image-checker
    user: api-server
current-context: default
```

Then add the plugin and the config file flag to the API server. Keep any plugins that are already in the list, and add `ImagePolicyWebhook` to them. Also mount the folder so the API server can read the files:

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
    - --admission-control-config-file=/etc/kubernetes/admission/admission-config.yaml
    volumeMounts:
    - name: admission
      mountPath: /etc/kubernetes/admission
      readOnly: true
  volumes:
  - name: admission
    hostPath:
      path: /etc/kubernetes/admission
      type: DirectoryOrCreate
```

After the API server restarts, a new pod must be rejected while the backend is unreachable:

```bash
watch crictl ps
kubectl run test-img --image=nginx:1.27
```

If a task asks you to allow only certain registries without a webhook backend, a `ValidatingAdmissionPolicy` (`admissionregistration.k8s.io/v1`) with a CEL rule such as `object.spec.containers.all(c, c.image.startsWith('registry.internal.example/'))` and a matching `ValidatingAdmissionPolicyBinding` with `validationActions: [Deny]` does the same job inside the API server.
</details>

### Task 21: Remove pods with critical vulnerabilities

In namespace `apps`, some pods run images that have `CRITICAL` vulnerabilities. Use `trivy` on `cp-1` to scan every image used in the namespace, and delete each pod whose image has at least one `CRITICAL` finding.

<details><summary>Solution</summary>

List each pod with its images:

```bash
ssh cp-1
kubectl -n apps get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'
```

Scan each image for critical findings only. With `--exit-code 1`, trivy returns 1 when it finds something, which makes the result easy to see. trivy also returns 1 when the scan itself fails, so read the output as well:

```bash
trivy image --severity CRITICAL --quiet --exit-code 1 <image>; echo "exit=$?"
```

Delete the pods whose image returned `exit=1`, then check what is left:

```bash
kubectl -n apps delete pod <pod-name>
kubectl -n apps get pods
```
</details>

---

## Domain 6: Monitoring, Logging and Runtime Security

### Task 22: Find the pod that reads device memory

In namespace `ops`, three Deployments run on `node-1`: `collector`, `exporter` and `shipper`. One of them opens `/dev/mem`. Falco is installed on `node-1`. Use Falco to find which Deployment it is, and scale that Deployment to 0 replicas. Do not change the other two.

<details><summary>Solution</summary>

Add a small custom rule in its own file under `/etc/falco/rules.d/`, which Falco loads by default. Do not edit `/etc/falco/falco_rules.yaml`, because it holds the default rules, and custom rules belong in `falco_rules.local.yaml` or `rules.d`.

```bash
ssh node-1
sudo -i
vim /etc/falco/rules.d/devmem.yaml
```

The output includes the container ID and the pod name, so you can map the event to a workload:

```yaml
- rule: Device memory opened
  desc: A process inside a container opened /dev/mem
  condition: evt.type in (open, openat, openat2) and fd.name = /dev/mem and container.id != host
  output: "devmem opened (container_id=%container.id pod=%k8s.pod.name ns=%k8s.ns.name proc=%proc.name file=%fd.name)"
  priority: WARNING
```

Restart Falco and follow its log. The real unit name depends on the driver, so check it first:

```bash
systemctl list-units 'falco*'
systemctl restart falco
journalctl -u falco-modern-bpf.service -f | grep devmem
```

If the pod name is empty, map the container ID to a pod with `crictl`:

```bash
crictl ps --id <container-id>
crictl inspect <container-id> | grep io.kubernetes.pod.name
exit
exit
```

Find the Deployment that owns that pod and scale it to zero:

```bash
ssh cp-1
kubectl -n ops get pod <pod-name> -o jsonpath='{.metadata.ownerReferences[0].name}{"\n"}'
kubectl -n ops scale deploy <deployment-name> --replicas=0
kubectl -n ops get deploy
```

The owner of a pod is its ReplicaSet, and the ReplicaSet name is the Deployment name plus a hash. So remove the last part of the name to get the Deployment.
</details>

### Task 23: Change the output of a Falco rule

On `node-1`, change the output of the existing Falco rule `Terminal shell in container` so that each alert prints only `shell,%evt.time,%user.uid,%container.id,%k8s.pod.name`. Do not edit `/etc/falco/falco_rules.yaml`. Falco must also write its alerts to `/var/log/falco/alerts.log`.

<details><summary>Solution</summary>

Override the rule in `/etc/falco/falco_rules.local.yaml`, which Falco loads after the default rules file. The `override` block tells Falco to replace only the `output` field and keep the rest of the rule:

```yaml
- rule: Terminal shell in container
  output: "shell,%evt.time,%user.uid,%container.id,%k8s.pod.name"
  override:
    output: replace
```

Back up `/etc/falco/falco.yaml`, then turn on file output in it:

```yaml
file_output:
  enabled: true
  keep_alive: false
  filename: /var/log/falco/alerts.log
```

Create the folder, check that the rules load without errors, and restart Falco:

```bash
mkdir -p /var/log/falco
falco --dry-run
systemctl restart falco
```

Trigger the rule by opening a shell in any container, then read the file:

```bash
crictl exec -it <container-id> sh -c "exit"
tail /var/log/falco/alerts.log
```
</details>

### Task 24: Turn on audit logging

On `cp-1`, enable API server auditing. Store the policy at `/etc/kubernetes/audit/policy.yaml` and write the log to `/var/log/kubernetes/audit/audit.log`. Keep logs for 7 days, keep at most 3 old files, and rotate at 50 MB. The policy must log Secret and ConfigMap access at `Metadata` level, log changes to Deployments in namespace `prod` at `RequestResponse` level, and log nothing else.

<details><summary>Solution</summary>

Write the policy. Kubernetes uses the first rule that matches, so the catch-all `None` rule goes last:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
- RequestReceived
rules:
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]
- level: RequestResponse
  namespaces: ["prod"]
  verbs: ["create", "update", "patch", "delete"]
  resources:
  - group: "apps"
    resources: ["deployments"]
- level: None
```

Back up the manifest, then add the flags and mount both the policy file and the log folder:

```bash
ssh cp-1
sudo -i
cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.bak
```

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --audit-policy-file=/etc/kubernetes/audit/policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit/audit.log
    - --audit-log-maxage=7
    - --audit-log-maxbackup=3
    - --audit-log-maxsize=50
    volumeMounts:
    - name: audit-policy
      mountPath: /etc/kubernetes/audit/policy.yaml
      readOnly: true
    - name: audit-log
      mountPath: /var/log/kubernetes/audit/
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

When the API server is back, read the Secrets and check that an event shows up. The hosts do not list `jq` as an installed tool, so use `grep`:

```bash
kubectl get secrets -A > /dev/null
grep '"resource":"secrets"' /var/log/kubernetes/audit/audit.log | tail -n 1
```
</details>

### Task 25: Make a workload immutable

Deployment `ledger` in namespace `finance` must not be able to change files in its container at runtime. Set its root filesystem to read-only and stop it from gaining privileges. The app still needs a writable `/var/cache/ledger`.

<details><summary>Solution</summary>

Edit the Deployment with `kubectl -n finance edit deploy ledger`. A read-only root filesystem blocks all writes, so mount an `emptyDir` only where the app really needs to write:

```yaml
spec:
  containers:
  - name: ledger
    image: registry.example.com/ledger:1.8.0
    securityContext:
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      privileged: false
    volumeMounts:
    - name: cache
      mountPath: /var/cache/ledger
  volumes:
  - name: cache
    emptyDir: {}
```

Then check that writes fail on the root filesystem but work in the cache folder. So the first `touch` must fail and the second must work:

```bash
kubectl -n finance rollout status deploy ledger
kubectl -n finance exec deploy/ledger -- touch /bin/test
kubectl -n finance exec deploy/ledger -- touch /var/cache/ledger/test
```
</details>
