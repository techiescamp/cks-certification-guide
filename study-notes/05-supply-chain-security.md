# Domain 5: Supply Chain Security (20%)

> **Exam Weight: 20%**: This domain tests whether you can build small and safe images, know what is inside them, allow only trusted images into the cluster, and find risky settings before they reach production.

The supply chain is every step between source code and a running container. That includes the base image, the build, the registry and the admission checks in the API server. So the tasks in this domain mix several tools. You will edit Dockerfiles and run scanners such as Trivy, Kubesec and KubeLinter. You will also create SBOMs with `bom` or Trivy, and configure admission control on the control plane. Many of these tools are not covered by the docs you can open during the exam, so learn their `--help` output well.

## Index

1. [Minimize the Base Image Footprint](#1-minimize-the-base-image-footprint)
2. [Fixing an Over-privileged Dockerfile](#2-fixing-an-over-privileged-dockerfile)
3. [Image Tags and Digests](#3-image-tags-and-digests)
4. [SBOMs with bom and Trivy](#4-sboms-with-bom-and-trivy)
5. [Worked Example: Find and Remove a Container by Package Version](#5-worked-example-find-and-remove-a-container-by-package-version)
6. [Private Registries and Image Pull Secrets](#6-private-registries-and-image-pull-secrets)
7. [Permitted Registries with ImagePolicyWebhook](#7-permitted-registries-with-imagepolicywebhook)
8. [Permitted Registries with ValidatingAdmissionPolicy](#8-permitted-registries-with-validatingadmissionpolicy)
9. [Signing and Verifying Images with Cosign](#9-signing-and-verifying-images-with-cosign)
10. [Scanning Images with Trivy](#10-scanning-images-with-trivy)
11. [Static Analysis with Kubesec and KubeLinter](#11-static-analysis-with-kubesec-and-kubelinter)
12. [Hands-on Checklist](#12-hands-on-checklist)
13. [References](#13-references)

---

## 1. Minimize the Base Image Footprint

Every package in an image is something an attacker can use and something a scanner can flag. A shell lets an attacker run commands. A package manager lets them install tools. So the smaller the image, the smaller the attack surface and the shorter the vulnerability report.

| Practice | Why it helps |
|----------|--------------|
| Start from a small base (`-slim`, `alpine`, distroless or `scratch`) | Fewer packages, so fewer CVEs and fewer tools for an attacker |
| Use multi-stage builds | Compilers and build tools stay in the build stage and never reach the final image |
| Pin the base image to a version or digest | Builds are repeatable, and `latest` cannot change under you |
| Install only what the app needs, without recommended extras | Smaller image and fewer packages to patch |
| Combine `apt-get update` and `apt-get install` in one `RUN`, then clean the cache | Fewer layers and no stale package lists left behind |
| Use `COPY` instead of `ADD` | `ADD` can fetch remote URLs and unpack archives, which hides what goes into the image |
| Run as a non-root `USER` | A container escape starts with fewer privileges |
| Never put secrets in `ENV`, `ARG` or copied files | Anyone who can pull the image can read every layer |

### A multi-stage build

This Dockerfile compiles a Go program in a full build image. Then it copies only the binary into a distroless base, which has no shell and no package manager, and runs it as a non-root user.

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

### Verify

Compare the image sizes, then confirm that the final image has no shell.

```bash
docker build -t app:1.0 .
docker images app
docker run --rm --entrypoint sh app:1.0
```

The last command should fail with an error that `sh` was not found. That is what you want.

### Common mistakes

- Copying the whole build stage with `COPY --from=build / /`, which brings every build tool back.
- Pinning the app version but leaving `FROM ubuntu:latest` unpinned.
- Removing packages in a later `RUN` line. The files still exist in the earlier layer, so the image does not get smaller.

---

## 2. Fixing an Over-privileged Dockerfile

A common task gives you a Dockerfile and asks you to fix the settings that give the container more power than it needs. Read it line by line and look for these problems:

- `FROM` with `latest` or a large general-purpose base.
- `USER root`, or no `USER` line at all, so the process runs as UID 0.
- Tools that the app does not need, such as `sudo`, `curl`, `vim` or `netcat`.
- Secrets in `ENV` or `ARG`.
- `ADD` with a remote URL.

Here is a Dockerfile with several of these problems.

```dockerfile
FROM ubuntu:latest
RUN apt-get update
RUN apt-get install -y python3 python3-pip curl vim sudo
ADD https://example.com/app.tar.gz /app/
ENV DB_PASSWORD=supersecret
USER root
CMD ["python3", "/app/app.py"]
```

The fixed version uses a pinned slim base, installs nothing extra, copies local files only, creates an unprivileged user and switches to it. The password now comes from a Kubernetes Secret at runtime instead of the image.

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

### Verify

Build the image and check which user the container runs as.

```bash
docker build -t app:1.1 .
docker run --rm app:1.1 id
```

The output should show `uid=10001`, not `uid=0(root)`.

### Common mistakes

- Changing `USER root` to `USER appuser` without creating that user first. Then the container fails to start.
- Placing `USER` before a `RUN` line that needs root, such as `useradd` or a package install. Put `USER` near the end.
- Deleting the secret from `ENV` but leaving it in a copied `.env` file.

The same review also applies to Kubernetes manifests. Section 11 shows how to find and fix over-privileged settings in a Deployment.

---

## 3. Image Tags and Digests

A tag such as `nginx:1.27` is a movable name. The registry owner can push new content under the same tag. A digest such as `nginx@sha256:<digest>` is the hash of the image manifest, so it always points at exactly the same content. Use a digest when a task asks for an image that cannot change.

Run a pod by digest. Replace `<digest>` with the real 64-character hash.

```bash
kubectl run digest-pod --image=nginx@sha256:<digest>
```

You can also find the digest that a running container uses. The kubelet records it in the pod status.

```bash
kubectl get pod digest-pod -o jsonpath='{.status.containerStatuses[*].imageID}'
```

On a node, `crictl` lists the digests of the images it has pulled.

```bash
crictl images --digests
```

### Verify

`kubectl describe pod digest-pod` shows the `Image` field with the `@sha256:` reference and an `Image ID` that matches it.

### Common mistakes

- Writing `nginx:sha256:...` instead of `nginx@sha256:...`. The `@` is required.
- Copying the image ID shown by `crictl images` (the config hash) instead of the manifest digest.

---

## 4. SBOMs with bom and Trivy

A Software Bill of Materials (SBOM) is a list of every package inside an image, with versions and licences. You use it to answer questions such as "which of our images contain this vulnerable library?". SPDX is the format the exam is most likely to ask for. SPDX has two encodings: tag-value (plain text) and JSON.

### bom

`bom` is a Kubernetes SIG Release tool. Its CLI reference is one of the docs you may open in the exam. `bom generate` writes SPDX tag-value by default, and `--format json` switches to SPDX JSON. Without `--output`, it prints to standard output.

```bash
bom generate --image registry.k8s.io/kube-apiserver:v1.37.0 --output /opt/sbom/kube-apiserver.spdx
bom generate --image registry.k8s.io/kube-apiserver:v1.37.0 --format json --output /opt/sbom/kube-apiserver.spdx.json
```

To get a quick tree view of an SBOM file, use `bom document outline`.

```bash
bom document outline /opt/sbom/kube-apiserver.spdx
```

> [!NOTE]
> `bom` looks for OS packages in container images only on Debian-based images (`--scan-images` is "currently debian only"). For an Alpine image, the SBOM may list the image layers but not every `apk` package. Use Trivy if you need the full package list of an Alpine image.

### Trivy

Trivy can write SPDX tag-value (`spdx`), SPDX JSON (`spdx-json`) and CycloneDX (`cyclonedx`).

```bash
trivy image --format spdx --output /opt/sbom/nginx.spdx nginx:1.27-alpine
trivy image --format spdx-json --output /opt/sbom/nginx.spdx.json nginx:1.27-alpine
```

Trivy can also scan an existing SBOM for vulnerabilities, which is useful when you cannot pull the image.

```bash
trivy sbom /opt/sbom/nginx.spdx.json
```

### Verify

Check that the file exists, is not empty and is the format you were asked for.

```bash
head -5 /opt/sbom/nginx.spdx
grep -c "PackageName:" /opt/sbom/nginx.spdx
```

A tag-value file starts with an `SPDXVersion:` line. A JSON file starts with `{` and contains `"spdxVersion"`.

### Common mistakes

- Writing JSON when the task asks for tag-value, or the other way round. Read the requested file name and format carefully.
- Using `--format cyclonedx` when the task says SPDX.
- Forgetting `--output`, so the SBOM only goes to the terminal.

---

## 5. Worked Example: Find and Remove a Container by Package Version

A realistic task looks like this. The Deployment `web` in namespace `apps` has three containers. One of them uses an image that contains `libcrypto3` version `3.3.2-r0`. You must find that container, save its SBOM in SPDX format to `/opt/sbom/web.spdx`, and then remove that container from the Deployment.

**Step 1: List each container and its image.**

```bash
kubectl -n apps get deploy web -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"\t"}{.image}{"\n"}{end}'
```

**Step 2: Check the package version in each image.** The SPDX tag-value output puts `PackageVersion` a few lines below `PackageName`, so print a few lines of context.

```bash
trivy image --format spdx nginx:1.27-alpine | grep -A3 "PackageName: libcrypto3"
trivy image --format spdx redis:7.4-alpine | grep -A3 "PackageName: libcrypto3"
trivy image --format spdx busybox:1.37 | grep -A3 "PackageName: libcrypto3"
```

If the pods are running, you can also ask the package manager inside each container. Use `apk` on Alpine images and `dpkg` on Debian images.

```bash
kubectl -n apps exec deploy/web -c cache -- apk info -v | grep libcrypto3
kubectl -n apps exec deploy/web -c proxy -- dpkg -l | grep libssl
```

**Step 3: Generate the SBOM for the image you found.** Use the tool the task names. Both commands below write SPDX tag-value.

```bash
bom generate --image redis:7.4-alpine --output /opt/sbom/web.spdx
trivy image --format spdx --output /opt/sbom/web.spdx redis:7.4-alpine
```

**Step 4: Remove the container.** Save the Deployment, delete that container's block (from `- name:` down to just before the next container), then apply it.

```bash
kubectl -n apps get deploy web -o yaml > web.yaml
vim web.yaml
kubectl apply -f web.yaml
```

### Verify

The new pods should show one container fewer, and the removed name should be gone.

```bash
kubectl -n apps rollout status deploy web
kubectl -n apps get pods
kubectl -n apps get deploy web -o jsonpath='{.spec.template.spec.containers[*].name}'
ls -l /opt/sbom/web.spdx
```

### Common mistakes

- Removing the container from a running pod. Pods are mostly immutable, so you must change the Deployment template.
- Matching the package name only and not the version. Several images may contain `libcrypto3` with different versions.
- Leaving a `volumeMounts` reference or an environment variable that depended on the removed container.

---

## 6. Private Registries and Image Pull Secrets

Most teams pull images from their own registry instead of public ones, because they can scan and sign what goes in. A pod needs credentials to pull from a private registry, so you store them in a `docker-registry` Secret.

```bash
kubectl -n apps create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=ci-bot \
  --docker-password='S3cr3tT0ken'
```

Then reference the Secret in the pod spec.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-app
  namespace: apps
spec:
  imagePullSecrets:
  - name: regcred
  containers:
  - name: app
    image: registry.example.com/team-a/app:1.4.2
```

You can also add `imagePullSecrets` to a ServiceAccount, so every pod that uses that ServiceAccount gets it automatically.

### Verify

`kubectl -n apps get pod private-app` should reach `Running`. If it shows `ErrImagePull` or `ImagePullBackOff`, `kubectl describe pod` shows the registry error.

### Common mistakes

- Creating the Secret in a different namespace from the pod. Pull secrets must be in the same namespace.
- Using a `generic` Secret instead of `docker-registry`. The kubelet expects the `kubernetes.io/dockerconfigjson` type.

---

## 7. Permitted Registries with ImagePolicyWebhook

ImagePolicyWebhook is an admission controller in the API server. For each pod, it sends the image names to an external webhook service. The webhook answers allow or deny, for example based on the registry. The plugin is disabled by default, so you must turn it on in the kube-apiserver manifest.

The setup has four parts, and a broken setup usually has a mistake in one of them:

| Part | Typical location |
|------|------------------|
| AdmissionConfiguration file | `/etc/kubernetes/policywebhook/admission_config.yaml` |
| kubeconfig for the webhook backend | `/etc/kubernetes/policywebhook/kubeconf` |
| Certificates referenced by the kubeconfig | `/etc/kubernetes/policywebhook/*.pem` |
| kube-apiserver flags and volume mounts | `/etc/kubernetes/manifests/kube-apiserver.yaml` |

### The AdmissionConfiguration

The `imagePolicy` block sets the webhook kubeconfig and the cache times. `allowTTL` and `denyTTL` are in seconds, and `retryBackoff` is in milliseconds. `defaultAllow` decides what happens when the webhook cannot be reached. Set it to `false` so images are denied when the backend is down.

```yaml
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

### The webhook kubeconfig

In this kubeconfig, `clusters` describes the remote webhook service and `users` describes how the API server authenticates to it. The server must use `https`. Every name must match across `clusters`, `users` and `contexts`.

```yaml
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

### The kube-apiserver flags and mounts

Add `ImagePolicyWebhook` to the existing plugin list and point the API server at the AdmissionConfiguration. Keep the plugins that are already there, such as `NodeRestriction`.

```yaml
- --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
- --admission-control-config-file=/etc/kubernetes/policywebhook/admission_config.yaml
```

The API server runs as a static pod, so it cannot see host files unless you mount them. Mount the whole directory, because the AdmissionConfiguration, the kubeconfig and the certificates all live there.

```yaml
volumeMounts:
- name: policywebhook
  mountPath: /etc/kubernetes/policywebhook
  readOnly: true
volumes:
- name: policywebhook
  hostPath:
    path: /etc/kubernetes/policywebhook
    type: DirectoryOrCreate
```

The two snippets above are fragments of the kube-apiserver manifest. Add the flags under `spec.containers[0].command`, the mount under that container's `volumeMounts`, and the volume under `spec.volumes`.

### Fixing a misconfigured setup

When a task says the setup "does not work", check these in order:

1. Is `ImagePolicyWebhook` in `--enable-admission-plugins`?
2. Does `--admission-control-config-file` point to the right file, and is that path mounted?
3. Does `kubeConfigFile` point to a file that exists on the node?
4. Do the cluster, user and context names in the kubeconfig match each other?
5. Is `server` an `https` URL with the right host and port?
6. Is `defaultAllow` set as the task asks (usually `false`)?

After you save the manifest, the kubelet restarts the API server. If it does not come back, read its logs on the control plane node.

```bash
watch crictl ps
crictl ps -a | grep kube-apiserver
crictl logs <container-id>
ls /var/log/pods/ | grep kube-apiserver
journalctl -u kubelet --since "10 min ago" | grep -i apiserver
```

### Verify

Create a pod with an image that the policy should reject. The API server should refuse it with a `Forbidden` error that mentions the image policy webhook.

```bash
kubectl run policy-test --image=docker.io/library/nginx:1.27
kubectl get pods -A | grep kube-apiserver
```

### Common mistakes

- Replacing the plugin list with only `ImagePolicyWebhook`, which drops `NodeRestriction` and fails CIS checks.
- Adding the flags but no volume mount. The API server then crashes because it cannot open the file.
- Setting `defaultAllow: true`. Every image is then allowed whenever the webhook is down, which defeats the policy.
- Editing a backup copy of the manifest instead of `/etc/kubernetes/manifests/kube-apiserver.yaml`. Also, never leave backups in that folder, because the kubelet runs every manifest it finds there.

---

## 8. Permitted Registries with ValidatingAdmissionPolicy

ValidatingAdmissionPolicy is a built-in alternative that needs no external service. It has been stable (`admissionregistration.k8s.io/v1`) since Kubernetes v1.30 and is enabled by default. You write the rule as a CEL expression in a policy, then attach the policy to resources with a binding.

This policy checks every container and init container image in new or updated pods. It only allows images from `registry.example.com`.

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
```

A policy does nothing until a binding uses it. This binding applies the policy only to the `prod` namespace and denies requests that fail it.

```yaml
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

### Verify

A pod from another registry should be rejected, and a pod from the allowed registry should be created.

```bash
kubectl -n prod run bad --image=docker.io/library/nginx:1.27
kubectl -n prod run good --image=registry.example.com/team-a/app:1.4.2
```

### Common mistakes

- Creating the policy without a binding.
- Matching only `pods` and then wondering why a Deployment shows no error. The Deployment is accepted, but its ReplicaSet cannot create pods. You see the denial in `kubectl describe rs`.
- Forgetting the trailing `/` in the prefix, so `registry.example.com.evil.io/...` also matches.

---

## 9. Signing and Verifying Images with Cosign

Signing proves who built an image and that nobody changed it afterwards. Cosign, from the Sigstore project, stores the signature in the registry next to the image. Sign the image by digest, so the signature refers to exact content and not to a tag that can move.

First create a key pair. This writes `cosign.key` (private, password protected) and `cosign.pub` (public).

```bash
cosign generate-key-pair
```

Sign the image with the private key, then verify it with the public key.

```bash
cosign sign --key cosign.key registry.example.com/team-a/app@sha256:<digest>
cosign verify --key cosign.pub registry.example.com/team-a/app@sha256:<digest>
```

`cosign verify` checks that the signature matches the public key and that the signed digest matches the image. In a cluster, an admission controller (for example a policy engine or a custom webhook) runs this check before pods start.

### Verify

`cosign verify` prints the verified payload as JSON and exits with code 0. Verifying with a different public key, or an image that was never signed, fails with a non-zero exit code.

### Common mistakes

- Signing a tag and then pushing a new image under the same tag. The signature no longer matches.
- Sharing `cosign.key` instead of `cosign.pub`. Only the public key is needed to verify.

---

## 10. Scanning Images with Trivy

Trivy compares the packages in an image with vulnerability databases. A typical task asks you to scan several images and report or remove the ones with `HIGH` or `CRITICAL` findings.

Scan an image and show only high and critical issues.

```bash
trivy image --severity HIGH,CRITICAL nginx:1.27-alpine
```

Hide issues that have no fix yet, which keeps the report focused on what you can act on.

```bash
trivy image --severity HIGH,CRITICAL --ignore-unfixed nginx:1.27-alpine
```

By default Trivy exits with code 0 even when it finds problems. Use `--exit-code 1` to fail a CI job.

```bash
trivy image --exit-code 1 --severity CRITICAL nginx:1.27-alpine
```

To check every image that runs in a namespace, list the images first and then scan each one.

```bash
kubectl -n apps get pods -o jsonpath='{range .items[*].spec.containers[*]}{.image}{"\n"}{end}' | sort -u
```

Trivy can also check Kubernetes manifests and Dockerfiles for misconfigurations.

```bash
trivy config --severity HIGH,CRITICAL deploy.yaml
trivy config Dockerfile
```

### Verify

Read the `Total:` line at the top of each result. It shows how many findings exist for each severity, so you can compare images quickly.

### Common mistakes

- Scanning `nginx` when the pod runs `nginx:1.27-alpine`. Always scan the exact image reference from the pod spec.

---

## 11. Static Analysis with Kubesec and KubeLinter

Static analysis reads a manifest without running it and flags risky settings. Kubesec gives a security score. KubeLinter reports failed checks with a hint on how to fix them.

### Spotting over-privileged settings in a manifest

This Deployment has almost every risky setting a task can include.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-app
  namespace: team-a
spec:
  replicas: 1
  selector:
    matchLabels:
      app: legacy-app
  template:
    metadata:
      labels:
        app: legacy-app
    spec:
      hostPID: true
      hostNetwork: true
      containers:
      - name: app
        image: nginx:latest
        securityContext:
          privileged: true
          runAsUser: 0
          allowPrivilegeEscalation: true
          capabilities:
            add: ["SYS_ADMIN"]
        volumeMounts:
        - name: host-root
          mountPath: /host
      volumes:
      - name: host-root
        hostPath:
          path: /
```

| Setting | Why it is dangerous |
|---------|---------------------|
| `hostPID: true` | The container can see and signal every process on the node |
| `hostNetwork: true` | The container shares the node network and can reach node-only ports |
| `privileged: true` | Almost no isolation from the host |
| `runAsUser: 0` | Runs as root |
| `allowPrivilegeEscalation: true` | A process can gain more privileges than its parent, for example through setuid binaries |
| `capabilities.add: SYS_ADMIN` | The most powerful capability, close to full root |
| `hostPath` of `/` | Read and write access to the whole node file system |
| `image: nginx:latest` | Unpinned image |

The fixed version removes the host access, pins an unprivileged image, and locks down the security context. This image runs as UID 101 and listens on port 8080. It also writes temporary files to `/tmp`. The root file system is now read-only, so an `emptyDir` gives it a writable `/tmp`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-app
  namespace: team-a
spec:
  replicas: 1
  selector:
    matchLabels:
      app: legacy-app
  template:
    metadata:
      labels:
        app: legacy-app
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

### Kubesec

Kubesec scans a manifest and returns JSON with a `score` and a `scoring` object. Inside `scoring`, the `critical` list shows settings that lower the score, and the `advise` list shows settings that would raise it.

```bash
kubesec scan deploy.yaml
```

If the binary is not installed but Docker is, run the published image and pass the file on standard input.

```bash
docker run -i kubesec/kubesec:v2 scan /dev/stdin < deploy.yaml
```

### KubeLinter

KubeLinter checks a file, a folder, a Helm chart or a Kustomize folder.

```bash
kube-linter lint deploy.yaml
kube-linter checks list
```

Some checks that are enabled by default and match the problems above:

| Check | Fails when |
|-------|-----------|
| `privileged-container` | A container runs privileged |
| `privilege-escalation-container` | Privilege escalation is allowed |
| `run-as-non-root` | The pod is not forced to run as non-root |
| `no-read-only-root-fs` | The root file system is writable |
| `host-network` | `hostNetwork` is used |
| `sensitive-host-mounts` | A sensitive host path is mounted |
| `latest-tag` | The image uses `latest` |

To skip a check on purpose for one object, add the annotation `ignore-check.kube-linter.io/<check-name>` with a short reason as the value.

### Verify

Run both scanners on the original and the fixed file. The Kubesec score should go up, and the KubeLinter errors for the settings you fixed should disappear.

```bash
kubesec scan deploy-fixed.yaml | grep '"score"'
kube-linter lint deploy-fixed.yaml
```

### Common mistakes

- Setting `runAsNonRoot: true` on an image that runs as root. The pod fails with `CreateContainerConfigError`, so pick a non-root image or set `runAsUser`.
- Adding `readOnlyRootFilesystem: true` without an `emptyDir` for paths the app writes to. The container then crashes.
- Adding `drop: ["ALL"]` and then also `add: ["SYS_ADMIN"]` back. Only add the capabilities the app really needs, such as `NET_BIND_SERVICE`.

---

## 12. Hands-on Checklist

Before you move on, make sure you can do each of these tasks on a real cluster:

- [ ] Can you rewrite a Dockerfile as a multi-stage build on a small, pinned base image?
- [ ] Can you spot `USER root`, `latest`, unneeded tools and secrets in a Dockerfile and fix them?
- [ ] Can you find the digest of a running container and start a pod using `image@sha256:...`?
- [ ] Can you generate an SPDX SBOM to a given path with `bom generate`, in both tag-value and JSON?
- [ ] Can you generate an SPDX SBOM with `trivy image --format spdx` and `--format spdx-json`?
- [ ] Can you find which container in a multi-container Deployment has a given package version, then remove only that container?
- [ ] Can you create a `docker-registry` Secret and use it through `imagePullSecrets`?
- [ ] Can you enable ImagePolicyWebhook with an AdmissionConfiguration, a webhook kubeconfig, the API server flags and the volume mounts?
- [ ] Can you find and fix a broken ImagePolicyWebhook setup from the kube-apiserver logs?
- [ ] Can you restrict image registries with a ValidatingAdmissionPolicy and a binding?
- [ ] Can you sign an image with Cosign and verify it with the public key?
- [ ] Can you scan images with Trivy for `HIGH,CRITICAL` findings and decide which workload to remove?
- [ ] Can you scan a manifest with Kubesec and KubeLinter and fix every over-privileged setting they report?

---

## 13. References

- [Dockerfile Best Practices](https://docs.docker.com/build/building/best-practices/)
- [Kubernetes Images](https://kubernetes.io/docs/concepts/containers/images/)
- [bom CLI Reference](https://kubernetes-sigs.github.io/bom/cli-reference/)
- [Trivy SBOM](https://trivy.dev/latest/docs/supply-chain/sbom/)
- [Trivy Filtering](https://trivy.dev/latest/docs/configuration/filtering/)
- [Pull an Image from a Private Registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
- [ImagePolicyWebhook Admission Controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#imagepolicywebhook)
- [Validating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)
- [Cosign: Signing Containers](https://docs.sigstore.dev/cosign/signing/signing_with_containers/)
- [Cosign: Verifying Signatures](https://docs.sigstore.dev/cosign/verifying/verify/)
- [Kubesec](https://kubesec.io/)
- [KubeLinter](https://docs.kubelinter.io/)
- [Configure a Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
