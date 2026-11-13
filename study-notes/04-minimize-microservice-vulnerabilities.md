# Domain 4: Minimize Microservice Vulnerabilities (20%)

> **Exam Weight: 20%**: This domain covers locking down Pods with security contexts and Pod Security Standards, handling Secrets safely (including encryption at rest), isolating tenants and workloads, and encrypting traffic between Pods.

This domain is about the workloads themselves. The host can be well hardened. But a Pod that runs as root with every capability still gives an attacker a lot to work with. So most tasks here ask you to fix a manifest, label a namespace, create or protect a Secret, or turn on encryption. Many of them start with something that is broken, such as a Deployment with no Pods. So get used to reading events, because they usually tell you exactly what is wrong.

## Index

1. [Security Contexts](#1-security-contexts)
2. [Pod Security Standards and Pod Security Admission](#2-pod-security-standards-and-pod-security-admission)
3. [Managing Kubernetes Secrets](#3-managing-kubernetes-secrets)
4. [Encrypting Secrets at Rest](#4-encrypting-secrets-at-rest)
5. [Multi-tenancy Basics](#5-multi-tenancy-basics)
6. [Sandboxed Containers with RuntimeClass](#6-sandboxed-containers-with-runtimeclass)
7. [Isolating Workloads with CiliumNetworkPolicy](#7-isolating-workloads-with-ciliumnetworkpolicy)
8. [Pod-to-Pod Encryption with Cilium WireGuard](#8-pod-to-pod-encryption-with-cilium-wireguard)
9. [Pod-to-Pod mTLS with Istio](#9-pod-to-pod-mtls-with-istio)
10. [Hands-on Checklist](#10-hands-on-checklist)
11. [References](#11-references)

---

## 1. Security Contexts

A `securityContext` tells the kubelet and the container runtime which user a container runs as and which kernel privileges it gets. You can set it at the Pod level (`spec.securityContext`), where it applies to all containers, or at the container level (`spec.containers[].securityContext`). When both set the same field, the container value wins. Some fields only exist at one level, as the table shows:

| Field | Level | What it does |
|-------|-------|--------------|
| `runAsUser` | Pod or container | The UID the process runs as |
| `runAsGroup` | Pod or container | The primary GID of the process |
| `fsGroup` | Pod only | The group that owns mounted volumes |
| `runAsNonRoot` | Pod or container | The kubelet refuses to start the container if it would run as UID 0 |
| `allowPrivilegeEscalation` | Container only | Sets `no_new_privs`, so a process cannot gain more privileges (for example through setuid binaries) |
| `readOnlyRootFilesystem` | Container only | Mounts the container's root filesystem read-only |
| `capabilities` | Container only | Adds or drops Linux capabilities |
| `privileged` | Container only | Gives the container almost all host privileges. Avoid it |
| `seccompProfile`, `appArmorProfile` | Pod or container | Kernel confinement (see Domain 3) |

### A least-privilege Pod

This Pod runs as UID 1000 and GID 3000, cannot escalate privileges, has no capabilities and cannot write to its root filesystem. The app still needs a writable `/tmp`, so an `emptyDir` volume is mounted there:

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

### Spotting over-privileged settings

When a task asks you to fix an insecure manifest, look for these settings and change them:

| Insecure setting | Fix |
|------------------|-----|
| `privileged: true` | Remove it or set `privileged: false` |
| `allowPrivilegeEscalation: true` or not set | Set `allowPrivilegeEscalation: false` |
| `runAsUser: 0` or no user set | Set a non-zero `runAsUser` and `runAsNonRoot: true` |
| `capabilities.add: ["SYS_ADMIN"]` or `["ALL"]` | Remove the add, then `drop: ["ALL"]` |
| `hostPID`, `hostIPC` or `hostNetwork: true` | Remove them |
| `hostPath` volumes | Replace with `emptyDir`, a ConfigMap or a PVC |
| No `readOnlyRootFilesystem` | Set it to `true` and mount `emptyDir` where the app writes |

**Verify:**

Check the identity, the read-only root, the `no_new_privs` flag and the effective capabilities from inside the container:

```bash
kubectl exec secure-pod -- id
kubectl exec secure-pod -- touch /test
kubectl exec secure-pod -- touch /tmp/test
kubectl exec secure-pod -- grep -E "NoNewPrivs|CapEff" /proc/1/status
```

You should see `uid=1000 gid=3000 groups=2000,3000`, a `Read-only file system` error for `/test`, success for `/tmp/test`, `NoNewPrivs: 1` and `CapEff: 0000000000000000`.

**Common mistakes:**

- Putting `allowPrivilegeEscalation`, `readOnlyRootFilesystem` or `capabilities` in the Pod-level `securityContext`. They only exist at the container level, so `kubectl` rejects them there as unknown fields.
- Setting `runAsNonRoot: true` on an image that runs as root without also setting `runAsUser`. The container then fails with `CreateContainerConfigError`, and the events say the image will run as root.
- Combining `allowPrivilegeEscalation: false` with `privileged: true`. A privileged container can always escalate, so the API server rejects this combination. Adding the `SYS_ADMIN` capability also turns privilege escalation back on, so do not add it.
- Forgetting that `readOnlyRootFilesystem: true` often breaks apps that write to `/tmp` or a cache folder. Mount an `emptyDir` there.

---

## 2. Pod Security Standards and Pod Security Admission

The **Pod Security Standards** (PSS) define three levels:

| Level | Meaning |
|-------|---------|
| `privileged` | No restrictions at all |
| `baseline` | Blocks known privilege escalations: no `privileged`, no host namespaces, no `hostPath`, no `hostPort`, only a safe set of added capabilities, no `Unconfined` seccomp |
| `restricted` | Everything in baseline, plus: `runAsNonRoot: true`, no `runAsUser: 0`, `allowPrivilegeEscalation: false`, `capabilities.drop` includes `ALL` (only `NET_BIND_SERVICE` may be added), `seccompProfile` set to `RuntimeDefault` or `Localhost`, and only safe volume types |

**Pod Security Admission** (PSA) is the built-in admission controller that enforces these levels. You turn it on per namespace with labels. Each label sets a level for one of three modes:

| Mode | What happens on a violation |
|------|-----------------------------|
| `enforce` | The Pod is rejected |
| `audit` | The Pod is allowed, and an annotation is added to the audit log event |
| `warn` | The Pod is allowed, and the user sees a warning |

### Label a namespace

This command enforces `restricted` in the `team-a` namespace and also warns on violations. The optional `-version` label pins the rules to a Kubernetes minor version, and `latest` is the default:

```bash
kubectl label --overwrite ns team-a \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted
```

The same labels in a Namespace manifest look like this:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-a
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
```

Before you enforce a level on a namespace that already has Pods, test it with a server-side dry run. The output lists the existing Pods that would violate it:

```bash
kubectl label --dry-run=server --overwrite ns team-a \
  pod-security.kubernetes.io/enforce=restricted
```

### Troubleshoot a Deployment whose Pods are not created

PSA `enforce` only checks Pods, not Deployments. So when the Pod template breaks the rules, the Deployment is created without any error, but it never gets Pods. The ReplicaSet controller tries to create them and fails, which means the reason is in the **ReplicaSet** events, not the Deployment's:

```bash
kubectl -n team-a get deploy web
kubectl -n team-a get rs -l app=web
kubectl -n team-a describe rs -l app=web
kubectl -n team-a get events --field-selector reason=FailedCreate
```

The `FailedCreate` event looks similar to this (shortened):

```
Error creating: pods "web-7c9d8f6b5d-x2x9k" is forbidden: violates PodSecurity "restricted:latest":
allowPrivilegeEscalation != false (container "web" must set securityContext.allowPrivilegeEscalation=false),
unrestricted capabilities (container "web" must set securityContext.capabilities.drop=["ALL"]),
runAsNonRoot != true (pod or container "web" must set securityContext.runAsNonRoot=true),
seccompProfile (pod or container "web" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

Each line names one field to fix. Here is the fixed Deployment, with everything `restricted` needs:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: team-a
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
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

**Verify:**

After you apply the fix, the Deployment creates a new ReplicaSet, and its Pods should start:

```bash
kubectl -n team-a rollout status deploy web
kubectl -n team-a get pods -l app=web
kubectl get ns team-a --show-labels
```

**Common mistakes:**

- Looking at `kubectl describe deploy` and seeing nothing useful. The error is on the ReplicaSet.
- Misspelling a label, such as `pod-security.kubernetes.io/enforced`. PSA ignores labels it does not know, so nothing is enforced and there is no error.
- Fixing only the first line of the error. Fix every field it lists, then check the events again.
- Expecting a new `enforce` label to delete running Pods. It only affects new Pods, so existing ones keep running until they are recreated.

---

## 3. Managing Kubernetes Secrets

A Secret holds small pieces of sensitive data, such as passwords, tokens or TLS keys. The values are only **base64-encoded**, not encrypted. So anyone who can read the Secret through the API, or read etcd, can see the value. Protect Secrets with RBAC (limit `get`, `list` and `watch` on `secrets`), and encrypt them at rest (see section 4). Also remember that anyone who can create a Pod in a namespace can mount any Secret in that namespace.

### Create Secrets

Create a generic Secret from literal values, or a TLS Secret from a certificate and key:

```bash
kubectl -n app create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password='S3cure!pass'

kubectl -n app create secret tls web-tls --cert=tls.crt --key=tls.key
```

### Use a Secret as environment variables

This Pod reads one key into one variable with `secretKeyRef`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-secret
  namespace: app
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 1h"]
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: password
```

### Mount a Secret as a volume

A volume is usually safer than environment variables, because env values can leak into logs and crash dumps. Each key becomes a file in the mount path. When the Secret changes, the kubelet updates the files after a short delay, but environment variables never change:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-secret
  namespace: app
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 1h"]
    volumeMounts:
    - name: creds
      mountPath: /etc/creds
      readOnly: true
  volumes:
  - name: creds
    secret:
      secretName: db-creds
      defaultMode: 0400
```

### Fix a Deployment that references a missing TLS Secret

Secrets are required by default, so none of a Pod's containers start until every Secret it references exists. A typical case is a Deployment that mounts a TLS Secret that was never created. The Pods hang in `ContainerCreating`, and the Pod events show a `FailedMount` warning that says the secret was not found:

```bash
kubectl -n app get pods -l app=web
kubectl -n app describe pod -l app=web | grep -A5 Events
kubectl -n app get deploy web -o jsonpath='{.spec.template.spec.volumes}'
```

Find the exact Secret name in the Deployment, then create it with the certificate and key you are given. For practice, you can make a self-signed pair with `openssl`:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=web.example.com"

kubectl -n app create secret tls web-tls --cert=tls.crt --key=tls.key
```

**Verify:**

The kubelet keeps retrying the mount, so the stuck Pods start by themselves once the Secret exists. Then check that the data reaches the container:

```bash
kubectl -n app get pods -l app=web
kubectl -n app get secret web-tls
kubectl -n app exec env-secret -- printenv DB_PASSWORD
kubectl -n app exec volume-secret -- ls -l /etc/creds
kubectl -n app get secret db-creds -o jsonpath='{.data.password}' | base64 -d
```

**Common mistakes:**

- Creating the Secret in the wrong namespace. A Pod can only use Secrets from its own namespace.
- Creating a generic Secret instead of a TLS one. A `kubernetes.io/tls` Secret must have the keys `tls.crt` and `tls.key`.
- Using a different Secret name or key from the one the Deployment references. Copy the name from the manifest.
- Thinking base64 is encryption. It is only an encoding.

---

## 4. Encrypting Secrets at Rest

By default, the API server stores Secrets in etcd as plain text. Anyone with a copy of etcd, or of an etcd backup, can read them. Encryption at rest makes the API server encrypt the data before it writes it to etcd. You configure it with an `EncryptionConfiguration` file and the `--encryption-provider-config` flag.

The **first** provider in the list is used to encrypt new writes. All providers in the list are tried when reading. So the `identity` provider (no encryption) goes last, which lets the API server still read Secrets that were stored before you turned encryption on.

### Step 1: Generate a key

The key must be 32 random bytes, base64-encoded. Do not base64-encode a word or a password, because that does not give you a valid key:

```bash
head -c 32 /dev/urandom | base64
```

### Step 2: Write the EncryptionConfiguration

Create the file on the control plane node, for example at `/etc/kubernetes/enc/enc.yaml`:

```bash
mkdir -p /etc/kubernetes/enc
```

Put the generated key in the `secret` field:

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
              secret: <BASE64_ENCODED_32_BYTE_KEY>
      - identity: {}
```

Other local providers are `secretbox` and `aesgcm`, which take a key in the same way. Then make sure only root can read the file, because it holds the key:

```bash
chmod 600 /etc/kubernetes/enc/enc.yaml
```

### Step 3: Point the API server at it

Edit `/etc/kubernetes/manifests/kube-apiserver.yaml`. You need three changes: the flag, a `volumeMount` and a `hostPath` volume. Without the mount, the API server container cannot see the file on the host. Here is the relevant part:

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

The kubelet sees the changed manifest and restarts the API server. Wait for it to come back:

```bash
watch crictl ps --name kube-apiserver
kubectl get nodes
```

If it does not come back, the container logs usually show the problem, such as a bad path or an invalid key:

```bash
crictl ps -a --name kube-apiserver
crictl logs <CONTAINER_ID>
ls /var/log/pods/ | grep kube-apiserver
journalctl -u kubelet --since "10 min ago" | grep -i apiserver
```

### Step 4: Re-encrypt existing Secrets

Only Secrets that are written after the restart are encrypted. To encrypt the ones that already exist, rewrite all of them with the same data:

```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

**Verify:**

Read a Secret straight from etcd. An encrypted value starts with `k8s:enc:aescbc:v1:key1:` followed by unreadable bytes. A plain-text value shows the data in clear:

```bash
ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/my-secret | hexdump -C
```

If `etcdctl` is not installed on the node, run it inside the etcd static Pod, which mounts the same certificates. The pipe to `hexdump` still runs on your node:

```bash
kubectl -n kube-system exec etcd-<CONTROL_PLANE_NODE> -- etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/my-secret | hexdump -C
```

Finally, check that the API server can still decrypt it:

```bash
kubectl get secret my-secret -o yaml
```

**Common mistakes:**

- Making the key with `echo -n "mypassword" | base64`. The key must decode to 16, 24 or 32 bytes for AES, so use 32 random bytes.
- Putting `identity: {}` first. Then nothing is encrypted, because the first provider is the one used for writes.
- Adding the flag but forgetting the `volumeMount` and `volume`, or writing `readonly` / `volumemounts` instead of `readOnly` / `volumeMounts`. The API server then fails to start.
- Skipping step 4, so old Secrets stay in plain text in etcd.
- Forgetting that `etcdctl get` needs the full key path: `/registry/secrets/<namespace>/<name>`.

---

## 5. Multi-tenancy Basics

Multi-tenancy means several teams or customers share one cluster. Kubernetes has no single "tenant" object, so you build isolation from several layers. The usual pattern is **one namespace per tenant**, with these controls on each namespace:

| Layer | Control |
|-------|---------|
| API access | RBAC Roles and RoleBindings scoped to the tenant's namespace |
| Resource use | `ResourceQuota` and `LimitRange`, so one tenant cannot starve the others |
| Pod privileges | Pod Security Admission labels (section 2) |
| Network | A default-deny NetworkPolicy, then explicit allows (section 7) |
| Kernel | A sandboxed runtime such as gVisor for untrusted code (section 6) |
| Nodes | Dedicated nodes per tenant with taints, tolerations and node selectors |

This ResourceQuota caps the total CPU, memory and Pod count in a tenant's namespace:

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

**Verify:**

Check what the quota allows and what is used, and check that a tenant user cannot reach another tenant's namespace:

```bash
kubectl -n tenant-a describe resourcequota tenant-a-quota
kubectl auth can-i list pods -n tenant-b --as=alice
```

**Common mistakes:**

- Treating namespaces as a security boundary on their own. Without RBAC, NetworkPolicy and PSA, a namespace only groups objects.
- Adding a CPU or memory quota without a `LimitRange`. Pods that do not set requests and limits are then rejected.

---

## 6. Sandboxed Containers with RuntimeClass

Normal containers share the host kernel. So a kernel bug can let a process escape the container. A sandboxed runtime adds another layer between the container and the host kernel. gVisor does this with a user-space kernel that intercepts the container's syscalls, and its OCI runtime is called `runsc`.

A **RuntimeClass** connects a name you use in Pods to a runtime `handler` that the container runtime knows about. The handler must already be set up on the node. For containerd, that means a runtime section in `/etc/containerd/config.toml` whose name matches the handler:

```bash
grep -A2 runsc /etc/containerd/config.toml
```

In a containerd config with `version = 2`, the gVisor section looks like this:

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
  runtime_type = "io.containerd.runsc.v1"
```

containerd 2.x uses `version = 3`, and there the section is named `[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runsc]`. The `runtime_type` stays the same. After you change the file, restart containerd with `systemctl restart containerd`.

Create the RuntimeClass. It is a cluster-scoped object:

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
```

Then set `runtimeClassName` in the Pod spec, or in the Pod template of a Deployment:

```yaml
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

**Verify:**

Inside a gVisor sandbox, `dmesg` shows gVisor's own kernel messages instead of the host's:

```bash
kubectl get pod sandboxed -o jsonpath='{.spec.runtimeClassName}'
kubectl exec sandboxed -- dmesg
```

**Common mistakes:**

- Using the RuntimeClass name as the handler, or the other way round. The Pod uses the `metadata.name`, and the `handler` must match the runtime name in the containerd config.
- Using a RuntimeClass whose handler is not configured on the node. The Pod then fails, and its events explain why.
- Scheduling the Pod on a node without gVisor. Use the RuntimeClass `scheduling.nodeSelector` field, or a node selector on the Pod, when only some nodes have it.

---

## 7. Isolating Workloads with CiliumNetworkPolicy

When the cluster runs Cilium, you can use `CiliumNetworkPolicy` (CNP) as well as the Kubernetes NetworkPolicy. A CNP selects Pods with `endpointSelector` and supports explicit **deny** rules (`ingressDeny` and `egressDeny`), which the Kubernetes NetworkPolicy does not have. Deny rules always win over allow rules, whatever policy they come from.

Like a NetworkPolicy, a CNP puts a Pod into default-deny for a direction as soon as any policy selects it for that direction. After that, only traffic that some rule allows gets through.

### L3/L4 allow rules

This policy lets only `app=frontend` Pods reach the `app=backend` Pods, and only on TCP port 8080:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: backend-from-frontend
  namespace: prod
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
```

To allow traffic from another namespace, add the namespace label that Cilium attaches to every endpoint. This egress rule lets every Pod in `frontend` reach Pods labelled `app=api` in the `backend` namespace:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: frontend-to-backend
  namespace: frontend
spec:
  endpointSelector: {}
  egress:
  - toEndpoints:
    - matchLabels:
        k8s:io.kubernetes.pod.namespace: backend
        app: api
```

### Block specific ports with deny rules

Deny rules can carry `toPorts`, so you can block single ports. Because any policy turns on default-deny, pair the deny with a broad allow, or every other connection is blocked too. This policy lets all traffic into the `app=db` Pods, with two exceptions. TCP port 22 is blocked from everywhere, and TCP port 3306 is blocked from outside the cluster. The Pods also cannot connect out on TCP port 25:

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
  - fromEntities:
    - world
    toPorts:
    - ports:
      - port: "3306"
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

Deny rules work at L3 and L4 only. They cannot use L7 (HTTP) rules or `toFQDNs`.

**Verify:**

Check that the policy is accepted, then test an allowed and a blocked connection:

```bash
kubectl -n prod get cnp
kubectl -n prod describe cnp db-block-ports
kubectl -n prod exec deploy/frontend -- nc -zv -w 3 backend 8080
kubectl -n prod exec deploy/frontend -- nc -zv -w 3 db 22
kubectl -n kube-system exec ds/cilium -- cilium-dbg policy get
```

**Common mistakes:**

- Writing a deny rule without a matching allow rule, which blocks all other traffic for the selected Pods.
- Writing the port as a number. In a CNP, `port` is a string, so quote it: `"8080"`.
- Forgetting DNS. Once egress is restricted, Pods that resolve names also need an egress rule to `kube-dns` on port 53.
- Using `apiVersion: cilium.io/v1`. The current version is `cilium.io/v2`.

---

## 8. Pod-to-Pod Encryption with Cilium WireGuard

Cilium can encrypt all Pod traffic that travels between nodes with WireGuard. Each node creates a key pair and publishes its public key on its `CiliumNode` object. Traffic to other nodes then goes through the `cilium_wg0` interface on UDP port 51871. Traffic between two Pods on the same node never leaves the host, so it is not encrypted.

Turn it on with the Cilium CLI, then restart the agents so they pick up the change:

```bash
cilium upgrade --reuse-values \
  --set encryption.enabled=true \
  --set encryption.type=wireguard

kubectl -n kube-system rollout restart ds/cilium
```

On a new install, pass the same `--set` values to `cilium install`.

**Verify:**

The agent status should show WireGuard, and `Peers` should equal the number of other nodes:

```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg status | grep Encryption
kubectl -n kube-system exec ds/cilium -- cilium-dbg encrypt status
```

On a node, cross-node Pod traffic should only appear as UDP 51871 on the main interface:

```bash
tcpdump -i eth0 -nn udp port 51871 -c 10
```

**Common mistakes:**

- Changing the config but not restarting the `cilium` DaemonSet.
- Testing with two Pods on the same node and expecting to see encrypted packets.

---

## 9. Pod-to-Pod mTLS with Istio

Istio encrypts traffic between Pods with mutual TLS (mTLS). In sidecar mode, Istio adds an `istio-proxy` container to each Pod, and the proxies handle the certificates and the encryption.

### Enable sidecar injection

Label the namespace so that new Pods get a sidecar. Injection happens only when a Pod is **created**, so restart the existing workloads afterwards:

```bash
kubectl label namespace payments istio-injection=enabled --overwrite
kubectl -n payments rollout restart deployment
```

You can switch injection off for one Pod with the label `sidecar.istio.io/inject: "false"` in its template.

### Require mTLS with PeerAuthentication

By default, Istio uses `PERMISSIVE` mode, which accepts both mTLS and plain text. To accept only mTLS, create a PeerAuthentication with mode `STRICT`. In a workload namespace it applies to that namespace. In the Istio root namespace (`istio-system`), it applies to the whole mesh:

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

**Verify:**

Injected Pods show `2/2` ready containers, including `istio-proxy`. Then a client in a namespace without sidecars should fail to reach the service, because it sends plain text:

```bash
kubectl get namespace -L istio-injection
kubectl -n payments get pods
kubectl -n payments get pod <POD> -o jsonpath='{.spec.containers[*].name}'
kubectl -n payments get peerauthentication
kubectl -n legacy exec deploy/curl -- curl -s -o /dev/null -w '%{http_code}\n' http://api.payments:8080
```

The last command prints `000` and fails once `STRICT` is active.

**Common mistakes:**

- Labelling the namespace but not restarting the Pods, so they still run without a sidecar.
- Writing the label as `istio-injection=true`. The value must be `enabled`.
- Turning on `STRICT` while some clients still have no sidecar. Those clients lose access.

---

## 10. Hands-on Checklist

Before you move on, make sure you can do each of these tasks on a real cluster:

- [ ] Can you make a container run as a given UID and GID, with `runAsNonRoot`, no privilege escalation, a read-only root filesystem and all capabilities dropped?
- [ ] Can you spot the over-privileged settings in a manifest and fix them?
- [ ] Can you prove from inside a container that `NoNewPrivs` is set and that it has no capabilities?
- [ ] Can you label a namespace to enforce, audit and warn at a given Pod Security Standard level?
- [ ] Can you find why a Deployment has no Pods by reading the ReplicaSet events, and fix it to meet `restricted`?
- [ ] Can you create a Secret and use it as an environment variable and as a mounted volume?
- [ ] Can you find the missing TLS Secret behind a stuck Deployment and create it with `kubectl create secret tls`?
- [ ] Can you generate a valid encryption key, write an EncryptionConfiguration and wire it into the API server with the flag, mount and volume?
- [ ] Can you re-encrypt existing Secrets and prove with `etcdctl` that they are stored as `k8s:enc:...`?
- [ ] Can you bring back an API server that will not start after a manifest change?
- [ ] Can you create a RuntimeClass for gVisor and run a Pod in it?
- [ ] Can you list the controls that isolate tenants in a shared cluster, and apply a ResourceQuota?
- [ ] Can you write a CiliumNetworkPolicy that allows one app to reach another on one port?
- [ ] Can you block specific ports with `ingressDeny` and `egressDeny` without blocking everything else?
- [ ] Can you turn on Cilium WireGuard encryption and confirm it in the agent status?
- [ ] Can you enable Istio sidecar injection for a namespace and restart the workloads so they get the sidecar?
- [ ] Can you enforce `STRICT` mTLS with a PeerAuthentication and show that a plain-text client is refused?

---

## 11. References

- [Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Enforce Pod Security Standards with Namespace Labels](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/)
- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Distribute Credentials Securely Using Secrets](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/)
- [Encrypting Confidential Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
- [Multi-tenancy](https://kubernetes.io/docs/concepts/security/multi-tenancy/)
- [Runtime Class](https://kubernetes.io/docs/concepts/containers/runtime-class/)
- [gVisor with containerd](https://gvisor.dev/docs/user_guide/containerd/quick_start/)
- [Cilium Layer 4 Policy](https://docs.cilium.io/en/stable/security/policy/layer4/)
- [Cilium Deny Policies](https://docs.cilium.io/en/stable/security/policy/deny/)
- [Cilium WireGuard Encryption](https://docs.cilium.io/en/stable/security/network/encryption-wireguard/)
- [Istio Sidecar Injection](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/)
- [Istio Mutual TLS Migration](https://istio.io/latest/docs/tasks/security/authentication/mtls-migration/)
