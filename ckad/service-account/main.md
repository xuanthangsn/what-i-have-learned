---
aliases: [K8s Service Accounts, SA]
tags: [kubernetes, security, infrastructure, access-control]
date: 2026-06-19
---

# Kubernetes Service Accounts (SA)

## Overview
A **Service Account (SA)** is the cryptographic identity assigned to a *machine* (a Pod) rather than a *human* (a User). It provides the exact same Mutual TLS (mTLS) authentication capabilities we use in a `kubeconfig` file, but automated, injected, and managed entirely by the Kubernetes Control Plane.

> **Key Paradigm:** If `kubeconfig` is a developer's physical ID badge to enter the building, a Service Account is the automated API key a server uses to talk to the internal database.

---

## The Two Types of Kubernetes Identities

| Feature | User Accounts (`kubeconfig`) | Service Accounts |
| :--- | :--- | :--- |
| **Target Audience** | Humans (Administrators, Developers) | Machines (Pods, Operators, CI/CD pipelines) |
| **Managcut: Press Ctrl + Shift + V (Windows/Linux) or Cmd + Shift + V (Mac).ement** | External (Corporate Active Directory, AWS IAM, OIDC) | Internal (Managed natively by the Kubernetes API) |
| **Scope** | Global (Cluster-wide) | Namespace-bound (Specific to one project/team) |
| **Credentials** | X.509 Certificates (`client-certificate-data`) | JWT (JSON Web Tokens) injected directly into Pod RAM |

---

## Architecture: How Service Accounts are Provisioned

When a Pod needs to talk to the Kubernetes API (e.g., to read a Secret or scale a Deployment), it cannot use a human's `kubeconfig`. Instead, it uses a Service Account through a specific, automated lifecycle:

### 1. The Definition
The Service Account is created as a standalone Kubernetes object within a specific namespace.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rook-ceph-default
  namespace: rook-ceph
```

### 2. The Assignment

In the Pod or Deployment YAML, the developer assigns the SA using the `serviceAccountName` directive. *(Note: If omitted, Kubernetes automatically assigns an SA named `default` with near-zero permissions).*

```yaml
spec:
  serviceAccountName: rook-ceph-default
  containers:
    - name: rook-direct-mount
      image: ceph/rook:v1.17.4
```

### 3. The Injection (The RAM Disk)

When the Kubelet spins up the Pod on the physical Worker Node, it intercepts the startup process. The Kubelet generates a secure **JSON Web Token (JWT)** and a copy of the cluster's **Certificate Authority (CA)** public key.

Instead of writing these to the physical hard drive, the Kubelet creates an in-memory `tmpfs` (RAM disk) and mounts the credentials directly into the container's filesystem at a hardcoded, universal path:

* `/run/secrets/kubernetes.io/serviceaccount/token` (The ID card)
* `/run/secrets/kubernetes.io/serviceaccount/ca.crt` (The CA validation data)

---

## Real-World Case Study: The Rook-Ceph Toolbox

In our infrastructure, the `rook-direct-mount` pod required god-mode privileges to bypass Kubernetes and interact directly with the underlying Ceph storage engine.

1. **The Request:** The deployment specified `serviceAccountName: rook-ceph-default`.
2. **The Mount:** Running `df -h` inside the pod revealed:
`tmpfs  252G 12K 252G 1% /run/secrets/kubernetes.io/serviceaccount`
3. **The Execution:** When the Ceph CLI commands were executed inside that terminal, the binaries automatically looked inside that `tmpfs` folder, grabbed the injected JWT, and used it to cryptographically prove to the Kubernetes API Server that the pod was authorized to read the Ceph Admin Secrets and Monitor IPs.

---

## Security Best Practices

* **Disable Automounting:** By default, Kubernetes mounts a Service Account token into *every single pod*, even if the application (like a frontend web server) never needs to talk to the Kubernetes API. This is a massive security risk if the pod is compromised.
* *Fix:* Set `automountServiceAccountToken: false` in the Pod spec for dumb applications.


* **Least Privilege (RBAC):** A Service Account natively has no permissions. It must be explicitly bound to a `Role` or `ClusterRole` using a `RoleBinding`. Never bind a broad `Cluster-Admin` role to an application's SA.
