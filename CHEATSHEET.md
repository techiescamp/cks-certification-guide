# CKS Command Cheatsheet

> This page gives you short, one-line reminders for each tool and topic, so you can find a command quickly during practice. For the full explanation and the YAML behind a command, open the study note for that domain.

## kubectl Security Checks

The `jq` lines below need `jq` installed. The exam hosts do not list it as a tool, so also know the `jsonpath` form.

```bash
kubectl auth whoami
kubectl auth can-i --list -n <ns> --as system:serviceaccount:<ns>:<sa>
kubectl auth can-i create pods -n <ns> --as <user>
kubectl get pods -A -o custom-columns=NS:.metadata.namespace,POD:.metadata.name,SA:.spec.serviceAccountName
kubectl get pods -A -o json | jq -r '.items[] | select(any(.spec.containers[]; .securityContext.privileged == true)) | "\(.metadata.namespace)/\(.metadata.name)"'
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.hostPID == true or .spec.hostNetwork == true) | "\(.metadata.namespace)/\(.metadata.name)"'
kubectl get pods -n <ns> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'
kubectl get rolebindings,clusterrolebindings -A -o wide | grep <subject>
kubectl describe rs -n <ns> -l app=<app> | grep -A10 Events
kubectl get events -n <ns> --sort-by=.lastTimestamp
kubectl explain pod.spec.securityContext
kubectl explain pod.spec.containers.securityContext
```

## Control Plane Flags and Static Pods

```bash
cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.bak
vim /etc/kubernetes/manifests/kube-apiserver.yaml
watch crictl ps
crictl ps -a | grep kube-apiserver
crictl logs <container-id>
tail -n 50 /var/log/pods/kube-system_kube-apiserver-*/kube-apiserver/*.log
journalctl -u kubelet --since "10 min ago" | grep -i apiserver
ps aux | grep kube-apiserver | tr ' ' '\n' | grep -- --
```

| File | What it controls |
|------|------------------|
| `/etc/kubernetes/manifests/kube-apiserver.yaml` | API server static pod (flags, volumes, mounts) |
| `/etc/kubernetes/manifests/etcd.yaml` | etcd static pod |
| `/var/lib/kubelet/config.yaml` | Kubelet settings, applied with `systemctl restart kubelet` |
| `/etc/kubernetes/pki/` | Cluster certificates and keys |

| Flag or field | Secure value |
|---------------|--------------|
| `kube-apiserver --anonymous-auth` | `false` |
| `kube-apiserver --authorization-mode` | `Node,RBAC` (never `AlwaysAllow`) |
| `kube-apiserver --enable-admission-plugins` | Includes `NodeRestriction`, plus any plugin the task names |
| `kube-apiserver --profiling` | `false` |
| `kube-apiserver --admission-control-config-file` | Path to the `AdmissionConfiguration` file |
| `kube-apiserver --encryption-provider-config` | Path to the `EncryptionConfiguration` file |
| `etcd --client-cert-auth` / `--peer-client-cert-auth` | `true` |
| `etcd --auto-tls` / `--peer-auto-tls` | `false` |
| kubelet `authentication.anonymous.enabled` | `false` |
| kubelet `authorization.mode` | `Webhook` |
| kubelet `readOnlyPort` | `0` |

## CIS Benchmark (kube-bench)

```bash
kube-bench run --targets master
kube-bench run --targets node
kube-bench run --targets master,node,etcd,policies
kube-bench --check="1.2.1,1.2.2"
kube-bench run --targets master | grep -E "^\[FAIL\]"
```

## Platform Binaries

```bash
kubelet --version
kubectl version --client
sha256sum <binary>
echo "$(cat kubelet.sha256)  kubelet" | sha256sum --check
```

## kubeadm Upgrade

```bash
kubectl get nodes
sed -i 's#/v1.36/#/v1.37/#' /etc/apt/sources.list.d/kubernetes.list
apt-mark unhold kubeadm kubelet kubectl
apt-get update && apt-cache madison kubeadm
apt-get install -y kubeadm='1.37.x-*'
kubeadm upgrade plan
kubeadm upgrade apply v1.37.x
kubeadm upgrade node
kubectl drain <node> --ignore-daemonsets
apt-get install -y kubelet='1.37.x-*' kubectl='1.37.x-*'
apt-mark hold kubeadm kubelet kubectl
systemctl daemon-reload && systemctl restart kubelet
kubectl uncordon <node>
```

Run `kubeadm upgrade apply` on the first control plane node only. Run `kubeadm upgrade node` on every other node.

## NetworkPolicy

```bash
kubectl get netpol -A
kubectl describe netpol <name> -n <ns>
kubectl get ns --show-labels
kubectl -n <ns> exec <pod> -- wget -qO- -T 3 <svc>.<ns>:<port>
kubectl -n <ns> exec <pod> -- nc -zv -w 3 <host> <port>
```

| Pattern | Spec |
|---------|------|
| Default deny in a namespace | `podSelector: {}` with `policyTypes: [Ingress, Egress]` and no rules |
| Source must match namespace AND pod | `namespaceSelector` and `podSelector` in the **same** `from` item |
| Source may match namespace OR pod | Two separate `from` items, each with its own `-` |
| Select a namespace by name | `kubernetes.io/metadata.name: <ns>` |
| Block the cloud metadata endpoint | `ipBlock: {cidr: 0.0.0.0/0, except: [169.254.169.254/32]}` |
| Keep DNS working under egress deny | Allow UDP and TCP 53 to `k8s-app: kube-dns` in `kube-system` |

## Cilium

```bash
cilium status --wait
kubectl get cnp,ccnp -A
cilium upgrade --reuse-values --set encryption.enabled=true --set encryption.type=wireguard
kubectl -n kube-system rollout restart ds/cilium
kubectl -n kube-system exec ds/cilium -- cilium-dbg status | grep Encryption
kubectl -n kube-system exec ds/cilium -- cilium-dbg endpoint list
kubectl -n kube-system exec ds/cilium -- cilium-dbg monitor --type drop
```

| Need | CiliumNetworkPolicy field |
|------|---------------------------|
| Pick the pods the policy applies to | `endpointSelector` |
| Allow from pods (L3) | `ingress[].fromEndpoints` |
| Allow to a port (L4) | `ingress[].toPorts[].ports[]` with `port: "8080"` and `protocol: TCP` |
| Always block a port, even if another rule allows it | `ingressDeny` / `egressDeny` with `toPorts` |
| Match everything, the outside world or the node | `fromEntities` / `toEntities` with `all`, `world`, `host` |

## Istio

```bash
kubectl label namespace <ns> istio-injection=enabled --overwrite
kubectl -n <ns> rollout restart deployment
kubectl -n <ns> get pods
kubectl get peerauthentication -A
istioctl analyze -n <ns>
```

A `PeerAuthentication` (`security.istio.io/v1`) with `mtls.mode: STRICT` in a namespace rejects plain-text traffic to every workload in it.

## RBAC and Service Accounts

```bash
kubectl create role <name> -n <ns> --verb=get,list,watch --resource=pods,deployments.apps
kubectl create rolebinding <name> -n <ns> --role=<role> --serviceaccount=<ns>:<sa>
kubectl create clusterrole <name> --verb=get,list --resource=nodes
kubectl create clusterrolebinding <name> --clusterrole=<role> --user=<user>
kubectl create role <name> -n <ns> --verb=get --resource=pods --dry-run=client -o yaml | kubectl apply -f -
kubectl create serviceaccount <sa> -n <ns>
kubectl patch serviceaccount default -n <ns> -p '{"automountServiceAccountToken": false}'
kubectl create token <sa> -n <ns> --duration=20m
kubectl -n <ns> exec <pod> -- ls /var/run/secrets/kubernetes.io/serviceaccount
```

| Need | Field |
|------|-------|
| Stop token mounting for all pods of an SA | `automountServiceAccountToken: false` on the ServiceAccount |
| Stop token mounting for one pod | `spec.automountServiceAccountToken: false` on the Pod |
| Short-lived token at a custom path | `projected.sources[].serviceAccountToken` with `path`, `expirationSeconds` (minimum 600) and optional `audience` |

## Pod Security Admission

```bash
kubectl label ns <ns> pod-security.kubernetes.io/enforce=restricted pod-security.kubernetes.io/enforce-version=latest
kubectl label ns <ns> pod-security.kubernetes.io/warn=restricted pod-security.kubernetes.io/audit=restricted
kubectl label --dry-run=server --overwrite ns <ns> pod-security.kubernetes.io/enforce=restricted
kubectl get ns <ns> --show-labels
```

| Mode | Effect |
|------|--------|
| `enforce` | Rejects pods that break the standard |
| `warn` | Allows the pod, but shows a warning to the user |
| `audit` | Allows the pod, but adds an annotation to the audit event |

Among other rules, the `restricted` standard needs `runAsNonRoot: true`, `allowPrivilegeEscalation: false`, `capabilities.drop: ["ALL"]` and a `seccompProfile` of `RuntimeDefault` or `Localhost`.

## Secrets and Encryption at Rest

```bash
kubectl create secret generic <name> -n <ns> --from-literal=<key>=<value>
kubectl create secret tls <name> -n <ns> --cert=<file.crt> --key=<file.key>
kubectl get secret <name> -n <ns> -o jsonpath='{.data.<key>}' | base64 -d
head -c 32 /dev/urandom | base64
kubectl get secrets -A -o json | kubectl replace -f -
etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key get /registry/secrets/<ns>/<name> | hexdump -C | head
```

In the `EncryptionConfiguration` (`apiserver.config.k8s.io/v1`), the first provider encrypts new data, and `identity: {}` goes last so old plain-text data can still be read.

## AppArmor and seccomp

```bash
aa-status
apparmor_parser -q /etc/apparmor.d/<profile-file>
apparmor_parser -r /etc/apparmor.d/<profile-file>
kubectl exec <pod> -- cat /proc/1/attr/current
ls /var/lib/kubelet/seccomp/
kubectl exec <pod> -- grep Seccomp /proc/1/status
```

| Need | securityContext field |
|------|-----------------------|
| AppArmor profile loaded on the node | `appArmorProfile: {type: Localhost, localhostProfile: <profile-name>}` |
| Runtime default AppArmor profile | `appArmorProfile: {type: RuntimeDefault}` |
| Runtime default seccomp profile | `seccompProfile: {type: RuntimeDefault}` |
| Custom seccomp profile | `seccompProfile: {type: Localhost, localhostProfile: profiles/<file>.json}`, relative to `/var/lib/kubelet/seccomp/` |

## Host Hardening

```bash
ss -lntp
ss -lntp | grep :<port>
lsof -i :<port>
ls -l /proc/<pid>/exe
systemctl status <pid>
systemctl list-units --type=service --state=running
systemctl disable --now <service>
apt list --installed | grep <name>
apt remove <package>
id <user>
gpasswd -d <user> docker
getent group docker
userdel -r <user>
chgrp root /var/run/docker.sock
ls -l /var/run/docker.sock
systemctl cat docker | grep ExecStart
cat /etc/docker/daemon.json
systemctl daemon-reload && systemctl restart docker
```

## Image Scanning and SBOM (trivy, bom)

```bash
trivy image <image>
trivy image --severity HIGH,CRITICAL <image>
trivy image --severity CRITICAL --quiet --exit-code 1 <image>
trivy image --format spdx --output <file>.spdx <image>
trivy image --format spdx-json --output <file>.spdx.json <image>
trivy image --format cyclonedx --output <file>.cdx.json <image>
trivy sbom <file>.spdx
trivy config <dir-or-file>
trivy fs <dir>
bom generate --image <image>
bom generate --image <image> --output <file>.spdx
bom generate --image <image> --format json --output <file>.spdx.json
bom document outline <file>.spdx
```

## Static Analysis (kubesec, kube-linter)

```bash
kubesec scan <manifest>.yaml
kube-linter lint <manifest>.yaml
kube-linter lint <dir>/
kube-linter checks list
```

## Image Signing (cosign)

```bash
cosign generate-key-pair
cosign sign --key cosign.key <registry>/<image>@sha256:<digest>
cosign verify --key cosign.pub <registry>/<image>@sha256:<digest>
```

## Falco

```bash
systemctl list-units 'falco*'
systemctl restart falco
journalctl -u falco-modern-bpf.service -f
falco --dry-run
falco -L
falco --list
crictl ps --id <container-id>
crictl inspect <container-id> | grep io.kubernetes.pod.name
```

| File | Purpose |
|------|---------|
| `/etc/falco/falco.yaml` | Main config: `rules_files`, `file_output`, `json_output` |
| `/etc/falco/falco_rules.yaml` | Default rules, which you should not edit |
| `/etc/falco/falco_rules.local.yaml` | Your custom rules and overrides, loaded after the defaults |
| `/etc/falco/rules.d/` | Extra rule files, loaded last |

| Output field | Value |
|--------------|-------|
| `%evt.time` | Event time |
| `%user.name` / `%user.uid` | User name or UID |
| `%proc.name` / `%proc.cmdline` | Process name or full command line |
| `%fd.name` | File or connection name |
| `%container.id` / `%container.name` | Container ID or name |
| `%k8s.pod.name` / `%k8s.ns.name` | Pod name or namespace |

## Audit Logs

```bash
tail -f /var/log/kubernetes/audit/audit.log
grep '"resource":"secrets"' /var/log/kubernetes/audit/audit.log | tail -n 5
jq -c 'select(.objectRef.resource=="secrets") | {user: .user.username, verb, ns: .objectRef.namespace, name: .objectRef.name}' /var/log/kubernetes/audit/audit.log
```

| kube-apiserver flag | Meaning |
|---------------------|---------|
| `--audit-policy-file` | Path to the `Policy` (`audit.k8s.io/v1`) file |
| `--audit-log-path` | Where to write the log file |
| `--audit-log-maxage` | Days to keep old log files |
| `--audit-log-maxbackup` | Number of old log files to keep |
| `--audit-log-maxsize` | Size in MB before the log rotates |

| Audit level | What is logged |
|-------------|----------------|
| `None` | Nothing |
| `Metadata` | User, time, resource and verb, but no request or response body |
| `Request` | Metadata plus the request body |
| `RequestResponse` | Metadata plus the request and response bodies |

Both the policy file and the log folder need a `hostPath` volume and a `volumeMount` in the API server manifest. Without them, the API server will not start.
