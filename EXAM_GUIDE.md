# CKS Exam Guide: Environment, Strategy and FAQ

> This guide explains what the CKS exam environment looks like, how to spend your 2 hours, and the answers to questions candidates often ask.

## Index

1. [The Exam Environment](#1-the-exam-environment)
2. [What Candidates Should Expect](#2-what-candidates-should-expect)
3. [First Minutes of the Exam](#3-first-minutes-of-the-exam)
4. [Time Management](#4-time-management)
5. [Per-task Workflow](#5-per-task-workflow)
6. [Common Traps](#6-common-traps)
7. [FAQ](#7-faq)

---

## 1. The Exam Environment

The points below are a summary of the official [Important Instructions: CKS](https://docs.linuxfoundation.org/tc-docs/certification/important-instructions-cks):

- The exam has **15 to 20 performance-based tasks**, and you have **2 hours** to finish them. The cluster runs **Kubernetes v1.37**.
- You work in a **Linux remote desktop** that runs inside the PSI Secure Browser. The desktop gives you a terminal, Firefox, VSCodium and a virtual keyboard.
- Every task must be done on a **designated host**. An infobox at the start of the task gives you the host name, and you reach it with `ssh <host>`. When you finish the task, run `exit` to go back to `base` before you start the next one, because nested SSH is not supported.
- On any task host, `sudo -i` gives you a root shell. You can also put `sudo` in front of a single command.
- The task hosts already have `kubectl` (with the `k` alias and Bash completion), `yq`, `curl`, `wget` and `man` installed. The `base` machine has none of these, so always run your commands on the task host.
- Never reboot `base`. Rebooting it does not restart your exam environment, so it only costs you time.
- The timer warns you when 30, 15 and 5 minutes are left.

The remote desktop uses some keyboard shortcuts that differ from a normal machine:

| Action | Shortcut |
|--------|----------|
| Copy / paste in the **terminal** | `Ctrl+Shift+C` / `Ctrl+Shift+V` |
| Copy / paste in **other apps** | `Ctrl+C` / `Ctrl+V` |
| `Ctrl+W` inside the remote desktop | Use `Ctrl+Alt+W` instead |
| Find the mouse cursor | `Ctrl+Alt+K` |
| Enter insert mode in `vim` | Press `i`, because the `INSERT` key is disabled |

The [Resources Allowed](https://docs.linuxfoundation.org/tc-docs/certification/certification-resources-allowed) page lists the documentation you may open during the exam. For the CKS, you may open:

- The Kubernetes docs and blog
- The Falco docs
- The bom CLI reference
- The etcd docs
- The ingress-nginx user guide
- The Cilium docs
- The Istio docs
- The links in each task's **Quick Reference** box

You may also read the documentation installed on the hosts under `/usr/share`, and the `man` pages.

---

## 2. What Candidates Should Expect

These points shape how you work during the exam, so prepare for each of them.

**Every task names a host.** You must do the work for each task on the host named in its infobox, and different tasks use different hosts. So check your prompt before each change, because a correct fix on the wrong host scores nothing.

**Control plane fixes mean editing static pod manifests.** Audit logging, encryption at rest, admission plugins and most CIS fixes for the API server and etcd are set in `/etc/kubernetes/manifests/kube-apiserver.yaml` or `etcd.yaml`. When you save one of these files, the kubelet restarts that component. While the API server restarts, `kubectl` fails, which is normal. If it does not come back, debug it with `crictl` and the logs, because `kubectl` cannot help you.

**Security tools run on the hosts.** The curriculum covers CIS benchmarks, SBOMs, static analysis of workloads and container images (for example with Kubesec and KubeLinter), and behavioral analytics with tools like Falco. So practice the common commands and flags of Falco, bom, trivy, kube-bench, kubesec and kube-linter until you do not need to look them up. The [cheatsheet](CHEATSHEET.md) lists them.

**Paste carefully.** In the terminal, copy with `Ctrl+Shift+C` and paste with `Ctrl+Shift+V`. YAML breaks when the indentation is wrong, so check it in `vim` after every paste. It also helps to know the short skeletons by heart: NetworkPolicy, securityContext, audit Policy, EncryptionConfiguration and projected volume.

**The timer never stops.** The Pause Exam option does not stop the timer. The timer also keeps running during a disconnect, and lost time is not added back. So only pause when you really have to.

---

## 3. First Minutes of the Exam

The `k` alias and completion already exist on the task hosts. Because each task starts in a fresh SSH session, keep any extra setup short enough to fit on one line that you can retype on each host:

```bash
export do="--dry-run=client -o yaml" now="--force --grace-period=0"
```

These `vim` settings make YAML editing easier. Run them once on any host where you will edit a lot of files:

```bash
printf 'set tabstop=2 shiftwidth=2 expandtab\n' >> ~/.vimrc
```

Next, skim **all** the tasks. Tasks vary a lot in effort. For example, adding PSA labels to a namespace takes about 2 minutes. Setting up ImagePolicyWebhook or audit logging with all its volume mounts can take 10 minutes or more. So a quick look at every task helps you decide what to do first.

---

## 4. Time Management

The suggested time for each domain follows its weight in the exam:

| Domain | Weight | Suggested time |
|--------|:------:|:--------------:|
| Cluster Setup | 15% | ~18 min |
| Cluster Hardening | 15% | ~18 min |
| System Hardening | 10% | ~12 min |
| Minimize Microservice Vulnerabilities | 20% | ~24 min |
| Supply Chain Security | 20% | ~24 min |
| Monitoring, Logging and Runtime Security | 20% | ~24 min |

A good approach is to work through the exam in **three passes:**

1. **Pass 1 (~60 min):** Collect the quick wins first, such as NetworkPolicies, RBAC, PSA labels, securityContext fixes, TLS Secrets, service account settings and trivy scans.
2. **Pass 2 (~45 min):** Then move on to the multi-step tasks that edit static pod manifests or node services: audit logging, encryption at rest, ImagePolicyWebhook, kubeadm upgrades, Falco rules and AppArmor.
3. **Buffer (~15 min):** Finally, use the remaining time to check every task again, especially that the API server and all static pods are running.

If a task takes more than ~8 minutes without visible progress, note its number and move on. That time is better spent on tasks you can finish.

---

## 5. Per-task Workflow

Follow the same sequence for every task:

```text
1. ssh <designated-host> from the task infobox
2. sudo -i when you need root
3. Read the whole task. Note host, namespace, names, users, ports, paths, versions
4. Back up any file before you edit it, e.g.
   cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/
5. Make the change, using the linked docs
6. Verify: watch crictl ps, kubectl get, curl, or the tool the task names
7. exit, then exit again to return to base
```

**Back up to `/root/`, not next to the original.** The kubelet runs every file in `/etc/kubernetes/manifests/` as a static pod. So a backup copy in that folder starts a second API server, which breaks the first one.

**Read for exact values.** Scoring is automatic, and tasks often depend on precise names, namespaces, file paths, user names, versions or output formats. So a working solution with the wrong name or path can still fail the check.

---

## 6. Common Traps

These are the mistakes that most often cost points, together with how to avoid each one:

| Trap | Prevention |
|------|-----------|
| The API server does not come back after a manifest edit | Check `crictl ps -a`, `crictl logs <id>`, `/var/log/pods/` and `journalctl -u kubelet`, then compare with your backup in `/root/` |
| A new config file is not visible to the API server | Every file the API server reads (audit policy, encryption config, admission config) needs a `hostPath` volume and a `volumeMount` |
| Working on the wrong host | Check the hostname in your prompt before every change |
| A backup left in `/etc/kubernetes/manifests/` | Always back up to `/root/` |
| Custom Falco rules are lost or ignored | Put them in `/etc/falco/falco_rules.local.yaml` or `/etc/falco/rules.d/`, never in `falco_rules.yaml`, and restart Falco |
| PSA labels set but pods are still admitted | `warn` and `audit` only report, so use `enforce` when pods must be rejected |
| PSA rejects pods, but the Deployment shows no error | Look at the ReplicaSet events with `kubectl describe rs` |
| A default-deny NetworkPolicy breaks DNS | When you deny egress, also allow UDP and TCP 53 to `kube-dns` |
| `namespaceSelector` and `podSelector` in separate items | Put them in the same `from` item when a source must match both |
| A kubelet config change has no effect | Run `systemctl restart kubelet` after editing `/var/lib/kubelet/config.yaml` |
| A Docker socket change is undone | A restart can recreate the socket. So restart Docker first, then change the socket group |

---

## 7. FAQ

**Q: Do I need the CKA first?**

A: Yes. You must pass the CKA before you can take the CKS.

**Q: What is the passing score?**

A: The passing score is 67%. Results are emailed to you within 24 hours.

**Q: How many simulator attempts do I get?**

A: You get two Killer.sh attempts per exam registration. Each one gives you 36 hours of access from the moment you activate it. The simulator is not part of the CKS-SINGLE offer, so check what your purchase includes. Save both attempts for the last week, after you have worked through the practice tasks.

**Q: Is there a retake?**

A: Yes, one retake is included. You must schedule and take the exam within 12 months of purchase. Once you pass, the certification is valid for 2 years.

For more official answers, see [Frequently Asked Questions: CKA, CKAD and CKS](https://docs.linuxfoundation.org/tc-docs/certification/faq-cka-ckad-cks).
