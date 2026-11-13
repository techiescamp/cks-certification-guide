# Domain 3: System Hardening (10%)

> **Exam Weight: 10%**: This domain covers shrinking the host OS attack surface, giving users only the access they need, closing network exposure on the node, and confining containers with AppArmor and seccomp.

This is the smallest domain, but its tasks are quick wins if you know your Linux basics. Most of the work happens on a node, not in `kubectl`. So you `ssh` to the host named in the task, become root with `sudo -i`, and use tools such as `systemctl`, `ss`, `apt` and `apparmor_parser`. The kernel hardening part then brings you back to Kubernetes, because you attach AppArmor and seccomp profiles to Pods through the `securityContext`.

## Index

1. [Minimize the Host OS Footprint](#1-minimize-the-host-os-footprint)
2. [Least-privilege Identity and Access Management](#2-least-privilege-identity-and-access-management)
3. [Minimize External Access to the Network](#3-minimize-external-access-to-the-network)
4. [Kernel Hardening with AppArmor](#4-kernel-hardening-with-apparmor)
5. [Kernel Hardening with seccomp](#5-kernel-hardening-with-seccomp)
6. [Hands-on Checklist](#6-hands-on-checklist)
7. [References](#7-references)

---

## 1. Minimize the Host OS Footprint

Every service and package on a node is something an attacker can use. A Kubernetes node only needs the container runtime, the kubelet and a few system services. So anything else, such as an FTP server, a web server or a compiler, should be stopped and removed.

### Disable services you do not need

First, list what is running. Then stop the service and disable it, so that it does not come back after a reboot:

```bash
systemctl list-units --type=service --state=running

systemctl stop vsftpd
systemctl disable vsftpd

# Or do both in one step
systemctl disable --now vsftpd
```

### Remove packages you do not need

Stopping a service is not enough if the task says to remove it. Find the package, then remove it. `apt purge` also deletes its config files:

```bash
apt list --installed | grep -i vsftpd
apt purge -y vsftpd
apt autoremove -y
```

If you only know the binary, ask `dpkg` which package owns it:

```bash
dpkg -S /usr/sbin/vsftpd
```

### Block kernel modules you do not need

The kernel can load modules on demand, and some rarely used network protocols have had security bugs. The Kubernetes docs use DCCP as an example of a module that is unlikely to be needed. You can stop a module from loading automatically with a `blacklist` line in a file under `/etc/modprobe.d/`:

```bash
lsmod | grep dccp
echo "blacklist dccp" >> /etc/modprobe.d/blacklist.conf
```

**Verify:**

The service should show `inactive (dead)` and `disabled`, and the package should be gone:

```bash
systemctl status vsftpd
systemctl is-enabled vsftpd
dpkg -l | grep vsftpd
```

**Common mistakes:**

- Running `systemctl stop` but not `disable`. The service then starts again on the next boot.
- Removing a package the node needs, such as `containerd`, `kubelet` or `openssh-server`. Read the task carefully before you remove anything.
- Doing the work on the wrong host. Check the hostname in your prompt before you start.

---

## 2. Least-privilege Identity and Access Management

On a node, the most powerful identities are `root`, members of the `sudo` group and members of the `docker` group. Membership of the `docker` group is the same as root access, because anyone who can talk to the Docker daemon can start a privileged container that mounts the host's `/`. So the rule is simple: give each user only the groups they need, and remove everyone else.

### Review users and groups

These commands show who can log in, who has sudo rights and who is in the `docker` group:

```bash
grep -v -E "nologin|false" /etc/passwd
getent group sudo
getent group docker
id alice
```

### Remove a user from a group

`gpasswd -d` removes one user from one group and leaves their other groups alone:

```bash
gpasswd -d alice docker
```

You can also lock an account so that nobody can log in with its password, or delete it completely:

```bash
usermod -L bob
userdel -r bob
```

### Restrict the Docker socket

The Docker CLI talks to the daemon through the Unix socket `/var/run/docker.sock`. By default the socket is owned by `root:docker` with mode `660`, so every member of the `docker` group can use it. To change who can use it, change the group that owns it:

```bash
ls -l /var/run/docker.sock
chgrp root /var/run/docker.sock
```

This change does not survive a restart of Docker. On systemd hosts the socket is created by the `docker.socket` unit, which sets the owner group with `SocketGroup=docker`. To make the change permanent, override that value:

```bash
systemctl edit docker.socket
```

In the editor, add these lines and save:

```ini
[Socket]
SocketGroup=root
```

Then reload systemd. Stop Docker and its socket, then start them again so the socket is created with the new group:

```bash
systemctl daemon-reload
systemctl stop docker.service docker.socket
systemctl start docker.socket docker.service
```

**Verify:**

The user should no longer be in the group, and the socket should show the new group:

```bash
id alice
getent group docker
ls -l /var/run/docker.sock
```

**Common mistakes:**

- Using `usermod -G docker alice` when you meant to remove a group. Without `-a`, `usermod -G` replaces *all* of the user's supplementary groups. Use `gpasswd -d` to remove one group.
- Expecting the change to apply to sessions that are already open. Group membership is read at login, so the user's current shell keeps the old groups until they log in again.
- Changing the socket with `chgrp` only, when the task asks for a lasting change.

---

## 3. Minimize External Access to the Network

Every listening port on a node is a way in. So you need to find each open port, work out which process owns it, and then close it. Usually you close it by stopping or removing the program, not by killing it, because a killed service can be restarted by systemd.

### Find open ports and the processes behind them

`ss -tulpn` lists listening TCP and UDP sockets together with the process name and PID. `lsof -i` gives the same answer from the process side:

```bash
ss -tulpn
ss -tlpn | grep :8080
lsof -i :8080
```

Once you have the PID, find the binary and the package or unit it belongs to:

```bash
ls -l /proc/<PID>/exe
systemctl status <PID>
dpkg -S /usr/local/bin/app01
```

Then stop it properly. If it is a service, disable it (see section 1). If it is a stray binary, kill it and delete it:

```bash
kill <PID>
rm /usr/local/bin/app01
```

### Stop dockerd from listening on TCP

The Docker daemon can listen on a TCP port, such as `tcp://0.0.0.0:2375`, as well as on its Unix socket. Without TLS, anyone who can reach that port can get root on the node. The TCP listener can be set in two places, so check both:

```bash
ss -tlpn | grep dockerd
systemctl cat docker.service | grep ExecStart
cat /etc/docker/daemon.json
```

First look at the `ExecStart` line. It is often in an override file under `/etc/systemd/system/docker.service.d/`. If it has a `-H tcp://...` flag, remove that flag and keep `-H fd://`. Then look at `daemon.json`. If it has a `hosts` key with a `tcp://` entry, remove that entry. Do not set hosts in both places. Docker does not start when `-H` is on the command line and `hosts` is also in `daemon.json`. The default unit already passes `-H fd://`, so it is usually simplest to delete the whole `hosts` key.

A clean `daemon.json` that only uses the default socket has no `hosts` key at all:

```json
{
  "log-driver": "json-file"
}
```

Reload systemd so it reads the edited unit, then restart Docker:

```bash
systemctl daemon-reload
systemctl restart docker
```

### Block ports with a host firewall

If a port must stay closed even when something starts listening on it, block it with the host firewall. On Ubuntu, `ufw` is the simplest option. Allow SSH first, so you do not lock yourself out:

```bash
ufw allow 22/tcp
ufw deny 2375/tcp
ufw default deny incoming
ufw --force enable
ufw status numbered
```

**Verify:**

The port should no longer appear, and Docker should still work through the socket:

```bash
ss -tlpn | grep -E ":2375|:8080"
docker ps
```

**Common mistakes:**

- Killing a process that systemd restarts a few seconds later. Stop and disable the unit instead.
- Forgetting `systemctl daemon-reload` after editing a unit file, so the old `ExecStart` is still used.
- Leaving the `hosts` key in `daemon.json` while the unit also passes `-H`, which stops Docker from starting.
- Enabling `ufw` with a default deny before allowing SSH.

---

## 4. Kernel Hardening with AppArmor

AppArmor is a Linux Security Module. It confines a program with a profile that lists which files, capabilities and network access the program may use. A profile runs in one of two modes. In `enforce` mode, AppArmor blocks and logs anything the profile does not allow. In `complain` mode, it only logs. Kubernetes does not load profiles for you, so the profile must already be loaded on every node where the Pod could run.

### Write the profile

Save this profile on the node, for example as `/etc/apparmor.d/k8s-deny-write`. It allows normal file access but denies every write:

```
#include <tunables/global>

profile k8s-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  file,

  # Deny all file writes.
  deny /** w,
}
```

### Load the profile

`apparmor_parser` loads a profile into the kernel, in enforce mode by default. Use `-r` to replace a profile that is already loaded, and `-C` to load it in complain mode:

```bash
apparmor_parser -q /etc/apparmor.d/k8s-deny-write
apparmor_parser -r /etc/apparmor.d/k8s-deny-write
```

Check that AppArmor is enabled and that the profile is loaded:

```bash
cat /sys/module/apparmor/parameters/enabled
aa-status | grep k8s-deny-write
```

### Apply the profile to a Pod

Since Kubernetes v1.30, you set AppArmor with the `appArmorProfile` field in the `securityContext`. The old `container.apparmor.security.beta.kubernetes.io` annotation is deprecated. The `type` is `RuntimeDefault`, `Localhost` or `Unconfined`. For `Localhost`, `localhostProfile` is the **profile name** declared inside the file, not the file path. You can set it for the whole Pod or per container, and the container setting wins:

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

To apply it to one container of a Deployment, put the same block under that container's `securityContext`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: busybox:1.36
        command: ["sh", "-c", "sleep 1h"]
        securityContext:
          appArmorProfile:
            type: Localhost
            localhostProfile: k8s-deny-write
```

**Verify:**

The process inside the container should report the profile, and a write should fail:

```bash
kubectl exec hello-apparmor -- cat /proc/1/attr/current
kubectl exec hello-apparmor -- touch /tmp/test
```

The first command prints `k8s-deny-write (enforce)`. The second fails with `Permission denied`.

**Common mistakes:**

- Loading the profile on the control plane but not on the worker node where the Pod is scheduled. The Pod then stays in `Pending`, and its events show `apparmor profile not found`. Use `kubectl get pod -o wide` to see the node.
- Putting the file name or path in `localhostProfile` when the name after `profile` inside the file is different.
- Using the deprecated annotation instead of the `appArmorProfile` field.

---

## 5. Kernel Hardening with seccomp

seccomp filters the system calls a process can make. By default, a container without a `seccompProfile` runs as `Unconfined`, which means no filter at all. The exception is a node where the kubelet runs with `--seccomp-default` (or `seccompDefault: true` in its config file). There the default is `RuntimeDefault`. You have three choices for the `seccompProfile.type`:

| Type | What it does |
|------|--------------|
| `RuntimeDefault` | Uses the container runtime's default profile, which blocks a set of dangerous syscalls |
| `Localhost` | Uses a profile file you put on the node |
| `Unconfined` | No seccomp filtering |

### Use the runtime default profile

This is the easiest win and is also required by the `restricted` Pod Security Standard:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-seccomp
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 1h"]
```

### Use a custom profile from the node

The kubelet looks for local profiles under `/var/lib/kubelet/seccomp/`. The `localhostProfile` value is a path **relative to that directory**. So first create the profile on the node where the Pod will run. This one logs every syscall and blocks nothing, which is useful to learn what a program needs:

```bash
mkdir -p /var/lib/kubelet/seccomp/profiles
cat > /var/lib/kubelet/seccomp/profiles/audit.json <<EOF
{
  "defaultAction": "SCMP_ACT_LOG"
}
EOF
```

A blocking profile uses `SCMP_ACT_ERRNO` as the default and lists the allowed syscalls under `syscalls`. With `"defaultAction": "SCMP_ACT_ERRNO"` and no allow list, every syscall fails. So the container cannot start, and the Pod ends up in `CrashLoopBackOff`.

Then point the Pod at the file with its relative path:

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

**Verify:**

The `Seccomp` line in the process status is `2` when a filter is active, and `0` when the container runs unconfined. You can also check the profile from the node with `crictl inspect`:

```bash
kubectl exec audit-pod -- grep Seccomp /proc/1/status
crictl ps --name app
crictl inspect <CONTAINER_ID> | grep -i -A3 seccomp
```

**Common mistakes:**

- Writing the full path `/var/lib/kubelet/seccomp/profiles/audit.json` in `localhostProfile`. It must be relative, so `profiles/audit.json`.
- Creating the profile on a different node from the one where the Pod runs. The container then fails to start, and `kubectl describe pod` shows the error.
- Writing invalid JSON. Check the file with `python3 -m json.tool <file>` if the container will not start.
- Setting `localhostProfile` together with `type: RuntimeDefault`. It is only allowed with `type: Localhost`.

---

## 6. Hands-on Checklist

Before you move on, make sure you can do each of these tasks on a real node:

- [ ] Can you list running services, then stop and disable one so that it stays off after a reboot?
- [ ] Can you find which package owns a binary and purge that package?
- [ ] Can you find which process listens on a given port with `ss` and `lsof`, and remove it for good?
- [ ] Can you remove a user from the `docker` group with `gpasswd` and confirm it with `id`?
- [ ] Can you change the group that owns `/var/run/docker.sock`, and make the change survive a Docker restart?
- [ ] Can you find where `dockerd` is told to listen on TCP, remove it, and restart Docker without breaking it?
- [ ] Can you block a port with `ufw` without locking yourself out of SSH?
- [ ] Can you write an AppArmor profile, load it with `apparmor_parser`, and confirm it with `aa-status`?
- [ ] Can you apply an AppArmor profile to a Pod or a single container with `securityContext.appArmorProfile`?
- [ ] Can you explain why a Pod is stuck when its AppArmor profile is not loaded on its node?
- [ ] Can you set `seccompProfile: RuntimeDefault` on a Pod and prove a filter is active with `/proc/1/status`?
- [ ] Can you place a custom seccomp profile under `/var/lib/kubelet/seccomp` and reference it with a relative `localhostProfile`?

---

## 7. References

- [Restrict a Container's Access to Resources with AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/)
- [Restrict a Container's Syscalls with seccomp](https://kubernetes.io/docs/tutorials/security/seccomp/)
- [Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Linux kernel security constraints for Pods and containers](https://kubernetes.io/docs/concepts/security/linux-kernel-security-constraints/)
- [Docker Engine security](https://docs.docker.com/engine/security/)
- [Configure remote access for Docker daemon](https://docs.docker.com/engine/daemon/remote-access/)
- [dockerd reference](https://docs.docker.com/reference/cli/dockerd/)
- [Securing a Cluster](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/)
- [AppArmor project](https://gitlab.com/apparmor/apparmor/-/wikis/home)
