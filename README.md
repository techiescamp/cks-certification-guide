# Ultimate Certified Kubernetes Security Specialist (CKS) Preparation Guide - V1.35 (2026)

## Hit the Star! :star:

If you are planning to use this repo for reference, please hit the star. Thanks!

## CKS Exam Overview

The Certified Kubernetes Security Specialist (CKS) exam has a duration of 2 hours.
To pass the exam, candidates need to achieve a score of at least 67%.
The exam will be on Kubernetes version 1.35.
Once the certificate is earned, the CKS certification remains valid for 2 years. The cost to take the exam is $445 USD.

## CKS Exam Coupon (30% Off Exclusive Discount)

To save a flat 30% on CKS exam registration, use the following coupon code.

**Coupon:** Use code **DCUBE30** at [kube.promo/cks](https://kube.promo/cks)

## Table of Contents

1. [Cluster Setup (15%)](#1-cluster-setup-15)
   - [Use Network security policies to restrict cluster level access](#)
   - [Use CIS benchmark to review the security configuration of Kubernetes components (etcd, kubelet, kubedns, kubeapi)](#)
   - [Properly set up Ingress with TLS](#)
   - [Protect node metadata and endpoints](#)
   - [Verify platform binaries before deploying](#)

2. [Cluster Hardening (15%)](#)
   - [Use Role Based Access Controls to minimize exposure](#)
   - [Exercise caution in using service accounts e.g. disable defaults, minimize permissions on newly created ones](#)
   - [Restrict access to Kubernetes API](#)
   - [Upgrade Kubernetes to avoid vulnerabilities](#)

3. [System Hardening (10%)](#)
   - [Minimize host OS footprint (reduce attack surface)](#)
   - [Using least-privilege identity and access management](#)
   - [Minimize external access to the network](#)
   - [Appropriately use kernel hardening tools such as AppArmor, seccomp](#)

4. [Minimize Microservice Vulnerabilities (20%)](#)
   - [Use appropriate pod security standards](#)
   - [Manage Kubernetes secrets](#)
   - [Understand and implement isolation techniques (multi-tenancy, sandboxed containers, etc.)](#)
   - [Implement Pod-to-Pod encryption using Cilium](#)

5. [Supply Chain Security (20%)](#)
   - [Minimize base image footprint](#)
   - [Understand your supply chain (e.g. SBOM, CI/CD, artifact repositories)](#)
   - [Secure your supply chain (permitted registries, sign and validate artifacts, etc.)](#)
   - [Perform static analysis of user workloads and container images (e.g. Kubesec, KubeLinter)](#)

6. [Monitoring, Logging and Runtime Security (20%)](#)
   - [Perform behavioral analytics to detect malicious activities](#)
   - [Detect threats within physical infrastructure, apps, networks, data, users and workloads](#)
   - [Investigate and identify phases of attack and bad actors within the environment](#)
   - [Ensure immutability of containers at runtime](#)
   - [Use Kubernetes audit logs to monitor access](#)

## CKS Exam Detailed Study Guide & References

CKS Certification Exam has the following key domains:

## 1. Cluster Setup (15%)

Following are the subtopics under Cluster Setup

### Restrict Pod to Pod communication using Network Policy
> [Network Policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) : Understand the restriction of Pod-to-Pod communication.

```yaml
# Create a Deny all Network Policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```
### Protecting Metadata Server access to the cloud provider Kubernetes cluster using Network Policy
> [Network Policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)  : Understand the IP Block parameter in the Network Policy.
```yaml
# Create a Network Policy with the IP Block
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: metadata-network-policy
  namespace: default
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
### CIS Benchmark to analyze the cluster components
> [CIS Benchmark](https://www.cisecurity.org/benchmark/kubernetes): Analyze the cluster components using CIS Benchmark tool Kube Bench.
```bash
# CIS Benchmark Best Practices
./kube-bench --config-dir /root/cfg --config /root/cfg/config.yaml
```
```bash
# Analyze the benchmark of specific check
./kube-bench --config-dir /root/cfg --config /root/cfg/config.yaml --check 1.4.1
```

### Secure the Ingress with TLS
> [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) : Creating an Ingress object with the TLS termination.
```bash
# Create a TLS Certificate & Key
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt
```
```bash
# Create a TLS secret with Certificate & Key
kubectl -n tls create secret tls tls-secret --cert=tls.crt --key=tls.key
```
```yaml
# Create Ingress object with TLS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: tls
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts:
      - dev.techiescamp.com
    secretName: tls-secret
  rules:
  - host: dev.techiescamp.com
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
### Verify Kubernetes Platform Binaries before deploying

```bash
# Check the current version of the Kubernetes component
kubectl --version

# Download Kubernetes v1.35 kubelet
curl -LO https://dl.k8s.io/release/v1.35.0/bin/linux/amd64/kubelet

# Download official checksum
curl -LO https://dl.k8s.io/release/v1.35.0/bin/linux/amd64/kubelet.sha256

# Verify binary
echo "$(cat kubelet.sha256)  kubelet" | sha256sum --check
```

## 2. Cluster Hardening (15%)

### RBAC, Certificate & Certificate Signing Request
> [Certificates and Certificate Signing Request](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/) : Create and issue a certificate for user
```bash
# Create private key
openssl genrsa -out myuser.key 2048
openssl req -new -key myuser.key -out myuser.csr -subj "/CN=myuser"

# Create Certificate Signing Request (CSR)
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: myuser
spec:
  request: 
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth

# Copy base64 encoded CSR file content
cat myuser.csr | base64 | tr -d "\n"

# Paste the content to `spec.request`
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: myuser
spec:
  request: <base64 encoded csr content>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth

# Apply the CSR manifest
kubectl apply -f csr.yaml

# Get the list of CSR
kubectl get csr

# Approve the CSR
kubectl certificate approve myuser

# Export the certificate
kubectl get csr myuser -o jsonpath='{.status.certificate}'| base64 -d > myuser.crt

# Create Role
kubectl create role developer --verb=create --verb=get --verb=list --verb=update --verb=delete --resource=pods

# Create Role Binding
kubectl create rolebinding developer-binding-myuser --role=developer --user=myuser

# Test the role to the user
kubectl auth can-i delete pods --as myuser
kubectl auth can-i delete deployments --as myuser

# Add new credentials
kubectl config set-credentials myuser --client-key=myuser.key --client-certificate=myuser.crt --embed-certs=true

# Add contexts
kubectl config set-contexts myuser --cluster=kubernetes --user=myuser

# List contexts
kubectl config get-context

# Change context
kubectl config use-contexts myuser
```

### Role Based Access Control
> [Bind RBAC with Service Account](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) : Understand the Role Based Access Control Authorization to Service Accounts
```bash
# Create SA
kubectl create sa app-sa

# Create Cluster Role
kubectl create clusterrole app-cr --verb list --resource pods

# Create Role Binding
kubectl create rolebinding app-rb --clusterrole app-cr --serviceaccount default:app-sa

# List Role Binding
kubectl get rolebinding

# Describe Role Binding
kubectl describe rolebinding app-rb

# Check the access
kubectl auth can-i list pods --as system:serviceaccount:default:app-sa

# Create a Pod with the Service Account
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: app-sa
  name: app-sa
spec:
  serviceAccountName: app-sa
  containers:
  - image: nginx
    name: app-sa
    ports:
    - containerPort: 80
```
### Service Account Token Automount
> [Service Account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/): Disable the automounting of the Service Account Token. 
```bash
# Disable Service Account Token Automounting
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: app-sa
  name: app-sa
spec:
  serviceAccountName: app-sa
  automountServiceAccountToken: false
  containers:
  - image: nginx
    name: app-sa
    ports:
    - containerPort: 80
```
### Disable the Service Account Token Automount at the Service Account level
```bash
# Create a new Service Account or edit an existing Service Account
apiVersion: v1
automountServiceAccountToken: false
kind: ServiceAccount
metadata:
  name: default
  namespace: two

# To check the automouting after the Pod deployment
k exec test-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```
### Upgrade Kubernetes clusters.
> [Perform Cluster Version upgrade Using Kubeadm](https://techiescamp.com/courses/certified-kubernetes-administrator-course/lectures/55120133): Managing the lifecycle involves upgrading clusters, managing control plane nodes, and ensuring consistency across versions.

## 3. System Hardening (10%)
### Disable Service
```bash
# Stop a running service
systemctl stop vsftpd

# Check the status of the service
systemctl status vsftpd
```
### Remove Unused Packages
```bash
# Remove the packages
sudo apt remove vsftpd
```
### Disable Open Ports 
```bash
# Identify open ports and related processes
ss -tlpn

# Filter a process using the open port number
ss -tlpn | grep :80

# List the Open files for the Port and the Process IDs
lsof -i :80

# Check the path of the package
ls -l /proc/12345/exe

# Kill the process using the PID
kill 12345

# Remove the package
rm /usr/bin/app01
```

### Kernel Hardening using AppArmor
>[Restrict container access to resources](https://kubernetes.io/docs/tutorials/security/apparmor/): Minimize the container access to reduce access to host resources.
```bash
# To list the default and custom loaded profiles
aa-status

# Profile modes - 'enforce' and 'complain'
# Example profile file which restricts the write function to nodes


#include <tunables/global>

profile k8s-apparmor-example-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  file,

  # Deny all file writes.
  deny /** w,
}

# To load the profile ('enforce' mode is default)

apparmor_parser /etc/apparmor.d/k8s-apparmor-example-deny-write

# Associate the profile to a Pod
apiVersion: v1
kind: Pod
metadata:
  name: hello-apparmor
spec:
  securityContext:
    appArmorProfile:
      type: Localhost
      localhostProfile: k8s-apparmor-example-deny-write
  containers:
  - name: hello
    image: busybox:1.28
    command: [ "sh", "-c", "echo 'Hello AppArmor!' && sleep 1h" ]

```
> Note: The profiles should be present on the worker nodes or where the workloads should be.

## 4. Minimize Microservice Vulnerabilities (20%)
### Run Container as Non-Root User 
> [Security context to enable non-root containers](https://kubernetes.io/docs/concepts/security/pod-security-standards/): Understand how to run a container with a non-root user.
```bash
# Adding security context to run the container as a non-root user
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: test-pod
  name: test-pod
spec:
  containers:
  - image: bitnami/nginx
    name: test-pod
  securityContext:
    runAsNonRoot: true
```

### Run a Container with Specific User ID and Group ID 
> [Run a container with a specific user](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/): Learn to deploy a container with least privilege and high security.
```bash
# Run container with specific user ID and group ID
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: test-pod
  name: test-pod
spec:
  containers:
  - image: busybox
    name: test-pod
    command: ["sh", "-c", "sleep 1d"]
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
```
### Create a Pod with lease privileges
> [Disable privilege escalation to a Pod](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) : Prevent containers from unauthorized access.
```bash
# Create a container with least privileges
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  containers:
  - name: sec-ctx-demo
    image: busybox:1.28
    command: [ "sh", "-c", "sleep 1h" ]
    securityContext:
      allowPrivilegeEscalation: false
      privileged: false
```
### Pod Security Admission
> [Enable Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/) : Enforce the Pod Security Standards using Pod Security Admission.
```bash
# Add label to the namespace for the PSA
k label namespace dev pod-security.kubernetes.io/enforce=restricted
```
### ETCD encryption
> [Encrypt the data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) : Secure the confidential data by encrypting.
```bash
# Create a random base64-encoded key
echo -n "encryptedsecret" | base64

# Create an Encryption configuration file with the encoded key
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aesgcm:
          keys:
            - name: key1
              secret: ZW5jcnlwdGVkc2VjcmV0
      - identity: {}

# Add Encryption Provider Config parameter and volumes on the Kube Api server

--encryption-provider-config=/etc/kubernetes/etcd/ec.yaml

spec
  volumes:
  - name: ec
    hostPath: 
      path: /etc/kubernetes/etcd
      type: DirectoryOrCreate

  containers:
    volumemounts:
    - name: ec
      mountPath: /etc/kubernetes/etcd
      readonly: true

# Wait to the Kube API server to restart
watch crictl ps

# Replace the secret
kubectl get secret test-secret -o json | kubectl replace -f -

# Check the encrypted secret 
ETCDCTL_API=3 etcdctl \
   --cacert=/etc/kubernetes/pki/etcd/ca.crt   \
   --cert=/etc/kubernetes/pki/etcd/server.crt \
   --key=/etc/kubernetes/pki/etcd/server.key  \
   get /registry/secrets/default/test-secret | hexdump -C
```
### Container Runtime sandboxed
> [Runtime Class](https://kubernetes.io/docs/concepts/containers/runtime-class/) : Improve isolation and security using sandbox container runtimes for important workloads
```bash
# Create a Runtime Class - gVisor
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc

# Create a Pod with RuntimeClass
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: rtc-pod
  name: rtc-pod
spec:
  runtimeClassName: gvisor
  containers:
  - image: nginx
    name: rtc-pod
    ports:
    - containerPort: 80

# To check the container runtime
kubectl get pod rtc-pod -o jsonpath='{.spec.runtimeClassName}'
kubectl get pod rtc-pod
```

### Cilium Network Policy 
> [Endpoint-Based Network Policy](https://docs.cilium.io/en/latest/security/policy/language/#endpoints-based): Configure CiliumNetworkPolicy to restrict outgoing traffic from frontend Pods to selected backend Pods.
```bash
# Create Cilium Network Policy for all outgoing traffic to a particular namespace and particular label
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: frontend
  namespace: frontend
spec:
  endpointSelector:
    matchLabels: {}
  egress:
  - toEndpoints:
    - matchLabels:
        io.kubernetes.pod.namespace: backend
        run: b-pod-1
```
## 5. Supply Chain Security (20%)

### Image Digest to run a Pod
> [Run a Pod with Image Digest](https://kubernetes.io/docs/concepts/containers/images/): Ensure the immutability and consistency of pulling the same image version using an image digest.
```bash
# Run a Pod using the image digest
k run digest-pod --image nginx@sha256:5ddf6decf65ea64c0492cd38098a9db11cb0da682478d3e0cfa8cdfdeb112f30

# To get the image digest of a Pod
k describe pod digest-pod | grep -iE "Image ID"
```

### Image Policy Webhook Admission Controller Plugin
> [Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/): Enforce the custom image validation policies through external webhook to improve security and complaince.
```bash
# Create Admission Controller configuration file
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
  - name: ImagePolicyWebhook
    configuration:
      imagePolicy:
        kubeConfigFile: <path-to-kubeconfig-file>
        allowTTL: 50
        denyTTL: 50
        retryBackoff: 500
        defaultAllow: false

# Find the kubeconfig path
find / -name kubeconf

# Update the Admission controller configuration and kubeconfig path
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
  - name: ImagePolicyWebhook
    configuration:
      imagePolicy:
        kubeConfigFile: /etc/kubernetes/policywebhook/kubeconf
        allowTTL: 50
        denyTTL: 50
        retryBackoff: 500
        defaultAllow: false

# Image Policy configuration file
apiVersion: v1
kind: Config
preferences: {}
clusters:
- name: image-check-webhook
  cluster:
    certificate-authority: /etc/kubernetes/policywebhook/external-cert.pem 
    server: https://localhost:1234    
contexts:
- context:
    cluster: image-checker-webhook
    user: api-server
  name: image-checker-webhook
current-context: image-checker-webhook

users:
- name: api-server
  user:
    client-certificate: /etc/kubernetes/policywebhook/apiserver-client-cert.pem 
    client-key:  /etc/kubernetes/policywebhook/apiserver-client-key.pem          

# Enable and configure Image Policy Admission Plugin

vim /etc/kubernetes/manifests/kube-apiserver

--enable-admission-plugins=ImagePolicyWebhook
--admission-control-config-file=/etc/kubernetes/policywebhook/admission-config.yaml
```
> Note: Admission Configuration could be in JSON format as well

### Scan Kubernetes manifests using Kubesec
```bash
# Scan manifest using binary
kubesec scan pod.yaml

# Scan using Kubesec Docker image
docker run -i kubesec/kubesec:512c5e0 scan /dev/stdin > pod.yaml
```
### Generate Software Bills of Materials (SBOMs) using bom
> [bom](https://github.com/kubernetes-sigs/bom):
Scan the container image using bom to generate an SBOM.
```bash
# Generate SBOM using bom
bom generate --image nginx:1.19.1-alpine-perl

# Scan the image and create Software Bills of Material
bom generate --image nginx:1.19.1-alpine-perl --output report.spdx 
```
## 6. Monitoring, Logging and Runtime Security (20%)
### Behavior analysis using Falco
> [Monitor and alert suspicious behavior in Kubernetes](https://falco.org/docs/reference/rules/supported-fields/): Monitor runtime security threats and create alerts to inform you of suspicious behavior.
```bash
# Check the Falco status
systemctl status falco

# To list the Falco parameters
falco --list

# Override existing rules before modifying 
cp /etc/falco/falco_rules.yaml /etc/falco/falco_rules.local.yaml

# Edit the rule and restart the service
systemctl restart falco

# Create a Pod and execute the shell for testing
k run test-pod --image nginx

k exec test-pod -- sh

# Check the falo logs
journalctl -fu falco

cat /var/log/syslog | grep falco
```

### Make Secret as environment variable inside the Pod
> [Secret as Env](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/) : Access the secret as an environment variable inside the container.
```bash
# Create a secret
k create secret generic secret-1 --from-literal password=admin@123

# Create Secret as an environment variable
apiVersion: v1
kind: Pod
metadata:
  name: env-single-secret
spec:
  containers:
  - name: envars-test-container
    image: nginx
    env:
    - name: SECRET_PASSWORD
      valueFrom:
        secretKeyRef:
          name: secret-1
          key: password

# To check the secret inside the Pod
k exec -it env-single-secret -- env
```

### Mount Secret as Volume 
> [Mount secret as volume](https://kubernetes.io/docs/concepts/configuration/secret/): Mount the Secret as a volume to improve security and enable dynamic updates.
```bash
# Mount secret as a volume in the Pod
apiVersion: v1
kind: Pod
metadata:
  name: volume-secret-pod
spec:
  volumes:
    - name: secret-volume
      secret:
        secretName: secret-2
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - name: secret-volume
          readOnly: true
          mountPath: "/etc/secret-volume"

# To check the mounted secret
k exec -it volume-secret-pod -- ls /etc/secret-volume
```

### Make the container immutable using the read-only root file system 
> [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/): Prevent the container root file system from being modified. 
```bash
# Create a Pod with ReadOnlyRootFilesystem security context
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: immutable-pod
  name: immutable-pod
spec:
  containers:
  - image: nginx
    name: immutable-pod
    securityContext:
      readOnlyRootFilesystem: true
```
> Note: If the container needs to write something, then if we are using the read-only root filesystem, the Pod won't work properly, so to resolve this, we need to create and mount an empty dir.
```bash
# Immutable deployment
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.21.6
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: nginx-run
      mountPath: /var/run
    - name: nginx-cache
      mountPath: /var/cache/nginx
  volumes:
  - name: nginx-run
    emptyDir: {}
  - name: nginx-cache
    emptyDir: {}
```

### Audit Policy
> [Audit Policy](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/): Learn how to monitor and log cluster activities
```bash
# Create and modify the Audit Policy configuration file
apiVersion: audit.k8s.io/v1 
kind: Policy
omitStages:
  - "RequestReceived"
rules:
  - level: RequestResponse
    resources:
    - group: ""
      resources: ["pods"]
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods/log", "pods/status"]

# Create a directory to store the log files.
mkdir -p /var/log/kubernetes/audit

# Add flags to the Kube API server manifest 
- --audit-policy-file=/etc/kubernetes/audit-policy.yaml
- --audit-log-path=/var/log/kubernetes/audit/audit.log
- --audit-log-maxage=30
- --audit-log-maxbackup=10
- --audit-log-maxsize=100

# Mount volumes of the Policy file and log path
volumeMounts:
  - mountPath: /etc/kubernetes/audit-policy.yaml
    name: audit
    readOnly: true
  - mountPath: /var/log/kubernetes/audit/
    name: audit-log
    readOnly: false

volumes:
- name: audit
  hostPath:
    path: /etc/kubernetes/audit-policy.yaml
    type: File
- name: audit-log
  hostPath:
    path: /var/log/kubernetes/audit/
    type: DirectoryOrCreate
```

