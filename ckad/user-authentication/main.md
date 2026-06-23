## Overview

A common misconception for Kubernetes newcomers is that the cluster natively manages "User" objects (e.g., `kubectl create user`). In reality, **Kubernetes does not have native User objects**. Instead, it relies on external identity providers (like OIDC, Active Directory, cloud IAM) or **X.509 Client Certificates** for authentication. 

This document outlines the standard procedure for provisioning a new user using X.509 certificates and granting them permissions using RBAC (Role-Based Access Control).

---

## Phase 1: Creating a New User (X.509 Certificates)

This phase relies on Asymmetric Cryptography (Public Key Infrastructure). You will generate a private key, create a Certificate Signing Request (CSR), and have the Kubernetes Cluster Certificate Authority (CA) sign it.

### Step 1: Generate a Private Key
The private key is the mathematical core of the user's identity. It must be kept strictly confidential. 

```bash
openssl genrsa -out johndoe.key 2048

```

* **Why it's needed:** The private key is used to digitally sign messages and prove identity during the TLS handshake with the API server.

### Step 2: Create a Certificate Signing Request (CSR)

The CSR packages the public key (derived automatically from the private key) along with the user's metadata.

```bash
openssl req -new -key johndoe.key -out johndoe.csr -subj "/CN=johndoe/O=developers"

```

* **Subject Mapping (`-subj`):** * `CN` (Common Name) is interpreted by Kubernetes as the **Username** (`johndoe`).
* `O` (Organization) is interpreted as the **Group** (`developers`).



### Step 3: Submit the CSR to Kubernetes

Create a Kubernetes `CertificateSigningRequest` object using the base64-encoded contents of the `.csr` file.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: johndoe-csr
spec:
  request: $(cat johndoe.csr | base64 | tr -d '\\n')
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # 1 day
  usages:
  - client auth
EOF

```

### Step 4: Approve the CSR and Extract the Certificate

A cluster administrator must approve the request so the cluster CA signs the certificate.

```bash
# Approve the request
kubectl certificate approve johndoe-csr

# Extract the signed certificate to a local file
kubectl get csr johndoe-csr -o jsonpath='{.status.certificate}' | base64 --decode > johndoe.crt

```

### Step 5: Add Credentials to Kubeconfig

Link the user's newly signed certificate and private key to your local `kubeconfig` file.

```bash
kubectl config set-credentials johndoe \\
  --client-certificate=johndoe.crt \\
  --client-key=johndoe.key

```

---

## Phase 2: Granting Permissions (RBAC)

At this point, `johndoe` is authenticated, but not authorized to do anything. We must bind them to a **Role**.

### Step 1: Create a Role

A `Role` defines a set of permissions (verbs) on specific resources within a specific namespace. (For cluster-wide permissions, use a `ClusterRole`).

Create a file named `role.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
spec:
  rules:
  - apiGroups: [""] # "" indicates the core API group
    resources: ["pods"]
    verbs: ["get", "watch", "list"]

```

Apply it to the cluster:

```bash
kubectl apply -f role.yaml

```

### Step 2: Create a RoleBinding

A `RoleBinding` attaches the `Role` (the permissions) to a **Subject** (the user or group).

Create a file named `rolebinding.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-johndoe
  namespace: default
subjects:
- kind: User
  name: johndoe # "name" is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role 
  name: pod-reader # This must match the name of the Role
  apiGroup: rbac.authorization.k8s.io

```

Apply it to the cluster:

```bash
kubectl apply -f rolebinding.yaml

```

---

## Phase 3: Switching Contexts and Verification

Kubernetes manages different user sessions via **Contexts** (a combination of a cluster, a user, and a default namespace).

### Step 1: Create a Context

```bash
kubectl config set-context johndoe-context \\
  --cluster=kubernetes \\
  --user=johndoe \\
  --namespace=default

```

### Step 2: Switch to the New User Context

```bash
kubectl config use-context johndoe-context

```

*All subsequent `kubectl` commands will now run as `johndoe`.*

### Step 3: Verify Access

Check your current identity and permissions:

```bash
# Verify current context
kubectl config current-context

# Verify current user (K8s v1.31+)
kubectl auth whoami

# Check if you have permission to list pods
kubectl auth can-i list pods

```
