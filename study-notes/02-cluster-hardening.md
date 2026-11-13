# Domain 2: Cluster Hardening (15%)

> **Exam Weight: 15%**: This domain covers least-privilege RBAC, safe use of service accounts, restricting access to the Kubernetes API, and upgrading a cluster to fix known vulnerabilities.

Every request to the Kubernetes API passes three gates. First the API server finds out who is calling (authentication). Then it checks whether that caller may do this action (authorization). After that, admission controllers can still change or reject the request. Cluster hardening means tightening each gate. Give users and service accounts only the rights they need. Stop pods from carrying tokens they never use. Close anonymous access, and keep the cluster on a patched version. Many tasks in this domain edit the API server's static pod manifest. So you also need to know how to bring it back when it crashes.

## Index

1. [Use RBAC to Minimize Exposure](#1-use-rbac-to-minimize-exposure)
2. [Use Service Accounts with Care](#2-use-service-accounts-with-care)
3. [Restrict Access to the Kubernetes API](#3-restrict-access-to-the-kubernetes-api)
4. [Upgrade Kubernetes to Avoid Vulnerabilities](#4-upgrade-kubernetes-to-avoid-vulnerabilities)
5. [Hands-on Checklist](#5-hands-on-checklist)
6. [References](#6-references)

---

## 1. Use RBAC to Minimize Exposure

RBAC grants permissions with four objects. A **Role** holds rules for one namespace, and a **ClusterRole** holds rules for the whole cluster. A **RoleBinding** gives the rules to users, groups or service accounts in one namespace. A **ClusterRoleBinding** gives them in every namespace. RBAC only adds permissions. There are no deny rules, so a subject's rights are the sum of all its bindings.

| You want | Use |
|----------|-----|
| Rights in one namespace | Role + RoleBinding |
| The same reusable rules in several namespaces | ClusterRole + one RoleBinding per namespace |
| Rights on cluster-scoped resources (nodes, namespaces, PVs) or in all namespaces | ClusterRole + ClusterRoleBinding |

A RoleBinding that points at a ClusterRole only grants the rules inside the RoleBinding's namespace. That is a good way to reuse a role without giving cluster-wide access.

### Create a least-privilege Role

The imperative commands are the fastest way in the exam. This Role lets a subject read pods and their logs in `dev`, and nothing more:

```bash
kubectl -n dev create role pod-reader --verb=get,list,watch --resource=pods,pods/log
kubectl -n dev create rolebinding pod-reader-jane --role=pod-reader --user=jane
```

Here is the same Role in YAML, which you need when a task gives you a file to fix:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
```

### Spot over-privileged rules

Some rights are much stronger than they look. Remove them unless the task really needs them:

| Right | Why it is risky |
|-------|-----------------|
| `*` in `verbs`, `resources` or `apiGroups` | Grants everything, including resources added in the future |
| `get`/`list`/`watch` on `secrets` | Reads every Secret in scope, including service account tokens. `list` and `watch` return the Secret contents too |
| `create` on `pods` or workloads | The pod can mount any Secret or use any service account in the namespace |
| `pods/exec` or `pods/attach` | Runs commands in, or attaches to, existing containers |
| `escalate`, `bind`, `impersonate` | Lets a subject give itself, or act with, more rights than it has |
| `nodes/proxy` | Reaches the kubelet API, which can run commands in any container on the node |
| A binding to `cluster-admin` | Full control of the cluster |

To find who has strong rights, look at the bindings and the subjects they name:

```bash
kubectl get clusterrolebindings -o wide | grep -E "cluster-admin|system:anonymous|system:unauthenticated"
kubectl -n dev get rolebindings -o wide
kubectl describe clusterrole <role-name>
```

To tighten a role, edit it and remove the extra verbs or resources. If a binding should not exist at all, delete it:

```bash
kubectl -n dev edit role app-role
kubectl delete clusterrolebinding <binding-name>
```

### Create a user with a CertificateSigningRequest

Kubernetes has no User object. A "user" is just a name in a client certificate that the cluster CA signed. The CN becomes the user name, and each O becomes a group. You can get such a certificate through the CertificateSigningRequest API.

First create a private key and a certificate request for the user `jane`:

```bash
openssl genrsa -out jane.key 3072
openssl req -new -key jane.key -out jane.csr -subj "/CN=jane"
```

Then submit it as a CertificateSigningRequest. The `request` field takes the base64 content of the `.csr` file on one line. The heredoc below fills it in for you:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  request: $(base64 < jane.csr | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages:
  - client auth
EOF
```

`usages` must be `client auth`, and `expirationSeconds` cannot be less than 600. Next, approve the request and save the signed certificate:

```bash
kubectl get csr
kubectl certificate approve jane
kubectl get csr jane -o jsonpath='{.status.certificate}' | base64 -d > jane.crt
```

Finally, add the user and a context to your kubeconfig:

```bash
kubectl config set-credentials jane --client-key=jane.key --client-certificate=jane.crt --embed-certs=true
kubectl config set-context jane --cluster=kubernetes --user=jane
kubectl config get-contexts
```

**Verify:**

Check the identity and the rights of the new user. `--as` lets an admin test as another user without switching context:

```bash
kubectl --context jane auth whoami
kubectl auth can-i list pods -n dev --as jane
kubectl auth can-i delete pods -n dev --as jane
kubectl auth can-i --list -n dev --as jane
```

The first `can-i` should print `yes`, and the second one `no`.

**Common mistakes:**

- Using a Role and RoleBinding in the wrong namespace. The rights then exist, just not where the user works.
- Granting `secrets` read access "just for listing". `list` returns the Secret data too.
- Forgetting the `""` (core) API group for pods, Services, Secrets and ConfigMaps, or using `apps` for pods.
- Old commands from older guides that do not exist, such as `kubectl config set-contexts` or `kubectl config use-contexts`. The correct ones are `set-context`, `get-contexts` and `use-context`.

---

## 2. Use Service Accounts with Care

Pods use **service accounts** to talk to the API. Every namespace has a ServiceAccount named `default`, and a pod that does not name one gets `default`. By default, the kubelet also mounts a short-lived token for that account into every container at `/var/run/secrets/kubernetes.io/serviceaccount/`. Most applications never call the API, so that token is just something an attacker can steal.

With RBAC, service accounts outside `kube-system` get no permissions beyond basic API discovery. So the risk comes from two things: tokens mounted where they are not needed, and roles that grant more than the app uses.

### Turn off automounting

Set `automountServiceAccountToken: false` on the `default` ServiceAccount. Then pods that use it no longer get a token:

```bash
kubectl -n prod patch serviceaccount default -p '{"automountServiceAccountToken": false}'
```

You can also set it on a single pod. If the ServiceAccount and the pod both set the field, **the pod wins**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  namespace: prod
spec:
  serviceAccountName: default
  automountServiceAccountToken: false
  containers:
  - name: web
    image: nginx:1.27
```

The change only affects new pods. For a Deployment, edit the pod template so that new pods roll out.

### Give each app its own minimal service account

When an app does need the API, create a dedicated ServiceAccount and bind only the rights it uses. Do not add rights to `default`, because every other pod in the namespace would get them too:

```bash
kubectl -n prod create serviceaccount app-sa
kubectl -n prod create role app-pod-reader --verb=get,list --resource=pods
kubectl -n prod create rolebinding app-sa-pod-reader --role=app-pod-reader --serviceaccount=prod:app-sa
```

Then set `serviceAccountName: app-sa` in the pod template. You cannot change the service account of a running pod, so the pod must be recreated.

### Mount a projected token with a short lifetime

Sometimes a task wants the token, but with a set lifetime and at a set path. Turn off the automatic token, then add a `projected` volume with a `serviceAccountToken` source. `expirationSeconds` must be at least 600, and it defaults to 3600. The kubelet refreshes the file before it expires. This pod gets a token that is valid for 20 minutes (1200 seconds):

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

The token file ends up at `<mountPath>/<path>`, which here is `/var/run/secrets/tokens/token`. You can also set `audience` when the token is meant for a service other than the API server. Without it, the audience is the API server.

To get a short-lived token on the command line, for example to test an account, use `kubectl create token`:

```bash
kubectl -n prod create token app-sa --duration=20m
```

### Find long-lived token Secrets

Before v1.24, Kubernetes created a Secret with a token that never expires for every service account. Such Secrets can still exist in older clusters, or someone can create one by hand. List them and delete the ones nobody uses:

```bash
kubectl get secrets -A --field-selector type=kubernetes.io/service-account-token
```

**Verify:**

Check what the pod really got, and what the account can do:

```bash
kubectl -n prod get pod token-client -o jsonpath='{.spec.volumes}'
kubectl -n prod exec token-client -- ls /var/run/secrets/tokens
kubectl -n prod exec token-client -- ls /var/run/secrets/kubernetes.io/serviceaccount
kubectl auth can-i list pods -n prod --as system:serviceaccount:prod:app-sa
kubectl auth can-i list secrets -n prod --as system:serviceaccount:prod:app-sa
```

The second `ls` should fail, because the automatic token is turned off. The first `can-i` should print `yes`, and the second one `no`.

**Common mistakes:**

- Setting `automountServiceAccountToken: false` on the ServiceAccount while the pod spec sets it to `true`. The pod wins.
- Putting `automountServiceAccountToken` under `containers` instead of under the pod `spec`.
- Setting `expirationSeconds` below 600. The API server rejects the pod.
- Writing `--serviceaccount=app-sa` without the namespace. The format is `<namespace>:<name>`.
- Checking `can-i` with `--as app-sa`. A service account's user name is `system:serviceaccount:<namespace>:<name>`.

---

## 3. Restrict Access to the Kubernetes API

Most API access settings are flags on the kube-apiserver. On a kubeadm cluster they live in the static pod manifest `/etc/kubernetes/manifests/kube-apiserver.yaml`. First see what is set today:

```bash
grep -E "anonymous-auth|authorization-mode|authorization-config|admission-plugins" /etc/kubernetes/manifests/kube-apiserver.yaml
```

### Authentication: no anonymous requests

When anonymous access is on (the default), a request without credentials is treated as the user `system:anonymous` in the group `system:unauthenticated`. The default RBAC rules only let that group read a few public endpoints, such as `/version` and `/healthz`. But anyone who later binds a strong role to it opens the cluster to the network. Turning anonymous access off removes that risk. Unauthenticated requests then get `401 Unauthorized`:

```yaml
    - --anonymous-auth=false
```

Domain 1 shows how kube-bench checks this flag. It also covers the `AuthenticationConfiguration` file that some clusters use instead, and the health probes that can fail after this change.

### Authorization: Node and RBAC, never AlwaysAllow

The API server asks its authorizers in the order you list them. kubeadm sets `--authorization-mode=Node,RBAC`. The `Node` authorizer limits each kubelet to the objects of its own node. `RBAC` handles everyone else. `AlwaysAllow` allows every request, so it must never be in the list:

```yaml
    - --authorization-mode=Node,RBAC
```

A cluster can configure its authorizers with an `--authorization-config` file instead. You cannot combine `--authorization-mode` with `--authorization-config`. If you set both, the API server exits at startup. So check which one the manifest uses before you change anything.

### Admission: NodeRestriction and other plugins

Admission plugins run after authorization. Many useful ones are on by default, such as `PodSecurity`, `ServiceAccount` and `ValidatingAdmissionPolicy`. Others must be added with `--enable-admission-plugins`. The order in that flag does not matter.

`NodeRestriction` is not in the default list, so kubeadm adds it. With this plugin, a kubelet can only modify its own Node object and the pods bound to its node. It also cannot delete its own Node object, and it cannot set labels with the `node-restriction.kubernetes.io/` or `node-role.kubernetes.io/` prefix. The plugin only limits kubelets that use credentials in the `system:nodes` group with the user name `system:node:<nodeName>`. It works together with the `Node` authorizer.

To enable another plugin, add it to the existing list. Do not create a second `--enable-admission-plugins` line:

```yaml
    - --enable-admission-plugins=NodeRestriction,AlwaysPullImages
```

Some plugins, such as `ImagePolicyWebhook` and `EventRateLimit`, also need a config file passed with `--admission-control-config-file`. That file must be mounted into the pod with a `hostPath` volume. Domain 5 shows a full example.

### Keep the API off the public network

Flags only control what an authenticated request may do. The fewer networks that can reach port 6443 at all, the better. So keep the API server behind a firewall or security group that only allows admin networks and the nodes. Do not expose it, or a kubelet, with a public NodePort or LoadBalancer.

**Verify:**

After the API server restarts, check that each setting took effect:

```bash
kubectl -n kube-system get pod -l component=kube-apiserver
ps -ef | grep kube-apiserver | tr ' ' '\n' | grep -E "anonymous-auth|authorization-mode|admission-plugins"
curl -k https://localhost:6443/version
```

The `curl` without credentials should return `401 Unauthorized`.

### Fix a kube-apiserver that does not come back

After a change to the manifest, the API server is down for a short time while the kubelet recreates it. If `kubectl` still says `connection refused` after a minute or two, the new manifest has a problem. You cannot use `kubectl` to debug it, so go to the container runtime and the kubelet on the control plane node.

First check whether a container exists and whether it keeps exiting. Then read its logs:

```bash
crictl ps -a | grep kube-apiserver
crictl logs <container-id>
```

If there is no container at all, the kubelet could not even read the manifest. That usually means broken YAML. The kubelet logs say why:

```bash
journalctl -u kubelet --since "10 min ago" | grep -i apiserver
```

The container logs also stay on disk under `/var/log/pods/`, even after the container is gone:

```bash
ls /var/log/pods/ | grep kube-apiserver
tail -n 30 /var/log/pods/kube-system_kube-apiserver-*/kube-apiserver/*.log
```

The log usually names the problem directly. These are the usual causes:

| What you see | Likely cause |
|--------------|--------------|
| No container, and the kubelet log shows a YAML parse error | Bad indentation, a tab, or a missing `-` in the manifest |
| `unknown flag` in the container log | A typo in the flag name |
| `no such file or directory` for a config or cert file | A wrong path, or a file that is not mounted into the pod with `volumes` and `volumeMounts` |
| An error about an unknown admission plugin | A misspelled plugin name in `--enable-admission-plugins` |
| An error about a malformed config file | A bad `apiVersion` or `kind` in the referenced file |

Fix the manifest, or restore your backup, then watch the container come back:

```bash
cp /root/kube-apiserver.yaml.bak /etc/kubernetes/manifests/kube-apiserver.yaml
watch crictl ps
```

**Common mistakes:**

- Adding a flag a second time instead of editing the existing line. Check with `grep` that each flag appears once.
- Pointing a flag at a file on the host without also adding a `hostPath` volume and a `volumeMount`. The container cannot see host files that are not mounted.
- Removing `Node` from `--authorization-mode`. The kubelets then lose the permissions the Node authorizer gave them.
- Restarting the kubelet over and over. The kubelet notices manifest changes on its own, so be patient and read the logs instead.

---

## 4. Upgrade Kubernetes to Avoid Vulnerabilities

Old versions stop receiving security fixes. So keeping a cluster on a supported patch release is part of hardening. With kubeadm you upgrade **one minor version at a time**, for example from v1.36 to v1.37. Skipping a minor version is not supported. Upgrade the control plane first, then the worker nodes, one node at a time.

The packages come from `pkgs.k8s.io`, which has **one repository per minor version**. So before a minor upgrade, you must point each node at the new version's repository. In the examples below, the target is `1.37.2`. Use the patch release that `apt-cache madison` shows on your node.

### Switch the package repository (every node)

The repository line is in `/etc/apt/sources.list.d/kubernetes.list`. It looks like `deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /`. Change the minor version in the URL, then check that the new versions are visible:

```bash
cat /etc/apt/sources.list.d/kubernetes.list
sudo sed -i 's|/v1.36/|/v1.37/|' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
apt-cache madison kubeadm
```

The same signing key is used for all versions, so you do not need a new key.

### Upgrade the first control plane node

Upgrade `kubeadm` first. The packages are held so that a normal `apt upgrade` does not touch them. So unhold them, install the exact version, then hold them again:

```bash
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.37.2-*' && \
sudo apt-mark hold kubeadm

kubeadm version
```

Next, see what kubeadm plans to do, then apply it. `upgrade apply` upgrades the control plane static pods and applies the new CoreDNS and kube-proxy manifests. It also renews the certificates kubeadm manages:

```bash
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.37.2
```

On other control plane nodes, run `sudo kubeadm upgrade node` instead of `upgrade apply`. With several control plane nodes, kubeadm upgrades CoreDNS and kube-proxy only after the last one is done.

Then drain the node, upgrade the kubelet and kubectl, restart the kubelet, and make the node schedulable again:

```bash
kubectl drain controlplane --ignore-daemonsets

sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet='1.37.2-*' kubectl='1.37.2-*' && \
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet

kubectl uncordon controlplane
```

### Upgrade a worker node

The worker needs the same new repository (see above). Then run these steps **on the worker**. For a worker, `kubeadm upgrade node` only updates the local kubelet configuration:

```bash
sudo apt-mark unhold kubeadm kubectl && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.37.2-*' kubectl='1.37.2-*' && \
sudo apt-mark hold kubeadm kubectl

sudo kubeadm upgrade node
```

Drain the worker from a machine that has admin access to the cluster, such as the control plane node:

```bash
kubectl drain node01 --ignore-daemonsets
```

Back on the worker, upgrade the kubelet and restart it:

```bash
sudo apt-mark unhold kubelet && \
sudo apt-get update && sudo apt-get install -y kubelet='1.37.2-*' && \
sudo apt-mark hold kubelet

sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Finally, from the control plane node, make the worker schedulable again:

```bash
kubectl uncordon node01
```

In the exam you cannot ssh from one node to another. So go back to the base host with `exit` before you `ssh` to the next node.

**Verify:**

All nodes should be `Ready` and show the new version. The control plane pods should run the new image tag:

```bash
kubectl get nodes
kubectl -n kube-system get pods -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image | grep kube-
systemctl status kubelet
```

If a node stays `NotReady`, read the kubelet logs with `journalctl -xeu kubelet`.

**Common mistakes:**

- Forgetting to switch the `pkgs.k8s.io` repository. Then `apt-cache madison` only shows the old minor version.
- Upgrading only `kubeadm` and never the kubelet. `kubectl get nodes` shows the kubelet version, so the node still looks old.
- Forgetting `systemctl daemon-reload` and the kubelet restart.
- Leaving the node cordoned (`SchedulingDisabled`) after the upgrade.
- Running `kubeadm upgrade apply` on a worker. Workers use `kubeadm upgrade node`.

---

## 5. Hands-on Checklist

- [ ] Can you create a Role and RoleBinding that let one user only read pods in one namespace?
- [ ] Can you reuse a ClusterRole in a single namespace with a RoleBinding?
- [ ] Can you find a binding that grants `cluster-admin` or access to `secrets` and reduce it?
- [ ] Can you create a user with a key, a CertificateSigningRequest, an approval and a kubeconfig context?
- [ ] Can you test a user's or service account's rights with `kubectl auth can-i --as`?
- [ ] Can you turn off token automounting for the `default` ServiceAccount and for a single pod, and know which one wins?
- [ ] Can you create a dedicated ServiceAccount with only the rights an app needs and use it in a Deployment?
- [ ] Can you mount a projected service account token with `expirationSeconds: 1200` at a given path?
- [ ] Can you set `--anonymous-auth=false` and prove that unauthenticated requests get `401`?
- [ ] Can you confirm that `--authorization-mode` includes `Node` and `RBAC` and not `AlwaysAllow`?
- [ ] Can you add an admission plugin to `--enable-admission-plugins` without breaking the API server?
- [ ] Can you bring back a crashing kube-apiserver using `crictl`, `/var/log/pods` and `journalctl -u kubelet`?
- [ ] Can you switch the `pkgs.k8s.io` repository to v1.37 and upgrade the control plane with kubeadm?
- [ ] Can you upgrade a worker node to the same version as the control plane, including drain and uncordon?

---

## 6. References

- [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [RBAC Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
- [Issue a Certificate for a Kubernetes API Client Using a CertificateSigningRequest](https://kubernetes.io/docs/tasks/tls/certificate-issue-client-csr/)
- [Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
- [Projected Volumes](https://kubernetes.io/docs/concepts/storage/projected-volumes/)
- [Managing Service Accounts](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)
- [Controlling Access to the Kubernetes API](https://kubernetes.io/docs/concepts/security/controlling-access/)
- [Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [Admission Control in Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [Debugging Kubernetes nodes with crictl](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/)
- [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [Upgrading Linux nodes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/)
- [Changing the Kubernetes package repository](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/)
