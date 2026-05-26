# k8s-visibility-checklist
> **A beginner-friendly checklist for observing what's happening inside your Kubernetes cluster before you lock anything down.**

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![CNCF Landscape](https://img.shields.io/badge/CNCF-Kubernetes-326CE5.svg)](https://www.cncf.io/)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](CONTRIBUTING.md)
[![KubeCon 2026](https://img.shields.io/badge/KubeCon%20NA-2026-purple.svg)](https://events.linuxfoundation.org/kubecon-cloudnativecon-north-america/)

---

## Why This Exists

When you first deploy an application to Kubernetes, the instinct is to ask: *is it running?*

The more important question one most beginners never think to ask is:

> *Do I know what it's doing, what it can reach, and what it's allowed to do?*

The gap between those two questions is the gap between deployment and security.

This checklist exists because that gap is real, it is common, and no beginner-friendly resource currently bridges it using only tools you already have. Everything here works with standard `kubectl` commands. No paid platforms. No observability stack. No prior security experience required.

---

## Who This Is For

- Developers deploying to Kubernetes for the first time
- Students and early-career engineers exploring cloud native
- Anyone who has gotten something running in Kubernetes and is now asking "but is it secure?"
- Practitioners who want a simple, repeatable pre-deployment observation habit

---

## The Core Idea: See Before You Secure

Most Kubernetes security advice tells you what to *configure*. This checklist teaches you what to *observe* first. You cannot secure what you cannot see and visibility is the first security activity, not a follow-on one.

The checklist is organised around four questions you should be able to answer before any deployment is considered done:

| # | Question | Why It Matters |
|---|----------|----------------|
| 1 | What is actually running in my cluster? | Unknown workloads are unmanaged attack surface |
| 2 | What can my workloads communicate with? | Open network paths are open doors |
| 3 | What are my workloads allowed to do? | Overpermissioned identities are a breach waiting to happen |
| 4 | Where did my container images come from? | Unverified images may contain vulnerabilities or malicious code |

---

## The Checklist

### 1. What Is Running?

**Command:**
```bash
kubectl get pods --all-namespaces
```

**What to look for:**
- All pods should be in `Running` or `Completed` state
- No unexpected namespaces or workloads you did not deploy
- No pods with names you do not recognise

**Red flags:**
- Pods in `CrashLoopBackOff` — something is misconfigured or broken
- Workloads running in the `default` namespace — everything should be namespaced intentionally
- More pods than you expect — something may have been deployed without your knowledge

**Dig deeper:**
```bash
kubectl describe pod <pod-name> -n <namespace>
```

---

### 2. What Are the Network Exposure Points?

**Command:**
```bash
kubectl get services --all-namespaces
```

**What to look for:**
- Services with `TYPE: LoadBalancer` or `TYPE: NodePort` are publicly or externally reachable
- Only services that are intentionally public should have these types
- `ClusterIP` services are internal only — this is the safest default

**Red flags:**
- A service exposed as `LoadBalancer` that you did not intend to make public
- No `NetworkPolicy` resources in your namespace (see below)

**Check for network policies:**
```bash
kubectl get networkpolicy --all-namespaces
```
If this returns nothing, all pods in your cluster can communicate with all other pods freely. That is almost never what you want.

---

### 3. What Are Your Workloads Allowed to Do?

**Command:**
```bash
kubectl get serviceaccounts --all-namespaces
```

**What to look for:**
- Every namespace has a `default` service account — and by default it may have more permissions than it needs
- Any pod that does not explicitly need to call the Kubernetes API should not be mounted with a service account token

**Check what roles are bound to your service accounts:**
```bash
kubectl get rolebindings --all-namespaces
kubectl get clusterrolebindings
```

**Red flags:**
- Service accounts bound to `cluster-admin` — this grants full control of the cluster
- Workloads using the `default` service account without an explicit, scoped role
- More `ClusterRoleBindings` than you created

**Check what a specific service account can do:**
```bash
kubectl auth can-i --list --as=system:serviceaccount:<namespace>:<serviceaccount>
```

---

### 4. Where Did Your Images Come From?

**Command:**
```bash
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{range .spec.containers[*]}{.image}{"\n"}{end}{end}'
```

**What to look for:**
- All images should come from a registry you trust and control
- No images tagged `:latest` — this tag is mutable and unpredictable in production
- Images should be from a specific, pinned digest or version tag

**Red flags:**
- Images pulled from public registries without any scanning
- `:latest` tags anywhere in production workloads
- Images from registries you do not recognise

**Scan an image for vulnerabilities (using Trivy — free and open source):**
```bash
trivy image <your-image-name>:<tag>
```

---

## Quick Reference: The Four Commands to Run First

```bash
# 1. What is running?
kubectl get pods --all-namespaces

# 2. What is exposed?
kubectl get services --all-namespaces
kubectl get networkpolicy --all-namespaces

# 3. What permissions exist?
kubectl get rolebindings,clusterrolebindings --all-namespaces

# 4. What images are in use?
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' | sort | uniq
```

---

## After Visibility: Your First Three Security Actions

Once you can see clearly, here is where to act first:

**1. Namespace your workloads**
Put workloads into dedicated namespaces by trust boundary, not all in `default`.
```bash
kubectl create namespace <your-app>
```

**2. Write a baseline NetworkPolicy**
Deny all traffic by default, then explicitly allow only what is needed.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: <your-app>
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**3. Scope your service account permissions**
Create a dedicated service account for each workload and bind only the roles it needs.
```bash
kubectl create serviceaccount <app-name>-sa -n <your-app>
```

---

## This Is a Living Document

This checklist was started by a developer learning Kubernetes and cloud native security from scratch. It is not written from authority, it is written from the experience of being a beginner who needed exactly this and could not find it.

If something is wrong, incomplete, or could be explained more clearly for a beginner, please open an issue or submit a pull request. Contributions from beginners are especially welcome. If something confused you, fixing it here helps the next person who is exactly where you were.

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines. This project follows the [CNCF Code of Conduct](https://github.com/cncf/foundation/blob/main/code-of-conduct.md).

---
## License

Licensed under the [Apache License 2.0](LICENSE).
