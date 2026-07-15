# Kubernetes — TryHackMe DevSecOps Pathway

Writeup covering Kubernetes fundamentals, hands-on cluster interaction with `kubectl`, and a practical walkthrough of Kubernetes security hardening using RBAC (Role-Based Access Control).

**Environment:** Browser-based AttackBox, Minikube cluster
**TryHackMe username:** Ketaki2004

---

## Objective

Deploy a simple nginx web application onto a Minikube cluster, find and exploit a Kubernetes Secret to retrieve a flag, then fix the exposure by setting up RBAC so only the right identity can access that Secret going forward.

---

## Phase One: Explore — Deploying to the Cluster

The Minikube cluster is started first, and a quick check confirms the default control plane and worker node components are running:

```bash
minikube start
kubectl get pods -A
```

This shows the standard `kube-system` pods — `etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `kube-proxy`, `coredns`, and `storage-provisioner` — confirming the control plane and worker node processes are healthy.

Two configuration files are provided:

- `nginx-deployment.yaml` — defines a Deployment with a single nginx pod replica
- `nginx-service.yaml` — defines a `NodePort` Service, exposing the deployment on port `8080` and targeting container port `80`

The Service is applied before the Deployment, which is standard practice in Kubernetes since services populate environment variables in containers at startup:

```bash
kubectl apply -f nginx-service.yaml
kubectl apply -f nginx-deployment.yaml
```

A follow-up check confirms the pod is running:

```bash
kubectl get pods -A
```

**📸 Screenshot 1:** `kubectl get pods -A` output showing the `nginx-deployment` pod in `Running` state, along with the two apply confirmations above it.

---

## Phase Two: Interact — Finding and Exploiting the Secret

`kubectl port-forward` tunnels the Kubernetes service port to a local port so it's reachable from the browser:

```bash
kubectl port-forward service/nginx-service 8090:8080
```

**📸 Screenshot 2:** Terminal output of the `port-forward` command showing the forwarding confirmation.

Navigating to `http://localhost:8090/` brings up a themed login page — "Jurassic Land Terminal Access" — asking for credentials.

**📸 Screenshot 3:** Browser view of the login page, before any credentials are entered.

A check for Kubernetes Secrets in the default namespace turns up something useful:

```bash
kubectl get secrets
kubectl describe secret terminal-creds
```

A secret named `terminal-creds` is found, containing `username` and `password` fields. Kubernetes Secrets are base64-encoded, not encrypted, by default, so both values are decoded:

```bash
kubectl get secret terminal-creds -o jsonpath='{.data.username}' | base64 --decode
kubectl get secret terminal-creds -o jsonpath='{.data.password}' | base64 --decode
```

**📸 Screenshot 4:** Terminal showing the secret discovery and decode commands — **blur or black out the decoded username and password** before publishing.

The decoded credentials are used to log into the terminal application, and the flag is retrieved.

**📸 Screenshot 5:** Browser view of the successful login/flag page — **blur the flag value**.

**Flag:** `THM{REDACTED}`

---

## Phase Three: Secure — Fixing It with RBAC

Now that the secret has been exposed, the next step is locking it down with Role-Based Access Control, so only an authorized identity can read it.

Two service accounts are created — one standard, non-privileged identity, and one admin identity that legitimately needs access:

```bash
kubectl create sa terminal-user
kubectl create sa terminal-admin
```

Two configuration files handle the actual restriction:

**`role.yaml`** defines a Role (`secret-admin`) scoped to only the `get` verb on the specific secret `terminal-creds` — not a blanket grant across all secrets:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: secret-admin
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
  resourceNames: ["terminal-creds"]
```

**`role-binding.yaml`** binds that Role to the `terminal-admin` service account only:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-admin-binder
  namespace: default
subjects:
- kind: ServiceAccount
  name: terminal-admin
  namespace: default
roleRef:
  kind: Role
  name: secret-admin
  apiGroup: rbac.authorization.k8s.io
```

Both are applied:

```bash
kubectl apply -f role.yaml
kubectl apply -f role-binding.yaml
```

**📸 Screenshot 6:** Terminal showing `cat role.yaml`, `cat role-binding.yaml`, and both service account creation commands.

### Verification

`kubectl auth can-i` confirms whether the restriction is actually working:

```bash
kubectl auth can-i get secret/terminal-creds --as=system:serviceaccount:default:terminal-user
# → no

kubectl auth can-i get secret/terminal-creds --as=system:serviceaccount:default:terminal-admin
# → yes
```

`terminal-user` is correctly denied, while `terminal-admin` keeps the access it needs — confirming least-privilege access is now in place on the `terminal-creds` secret.

**📸 Screenshot 7:** Terminal showing both `apply` confirmations and both `kubectl auth can-i` results (no / yes) together.

---

## Key Takeaways

- Kubernetes Secrets are base64-encoded, not encrypted, by default. Anyone with `get` access to a secret can decode it in seconds, so encryption at rest and tight RBAC scoping both matter.
- Least privilege through RBAC doesn't take much effort — a Role can be scoped down to a single verb (`get`) on a single named resource (`resourceNames: ["terminal-creds"]`) instead of granting broad access.
- Service accounts are how pods and applications get an identity when interacting with the Kubernetes API. Separating "admin" and "standard" identities is a simple, practical first step toward least-privilege design.
- `kubectl auth can-i` is a quick, reliable way to check that an RBAC policy is actually behaving as intended before trusting it in a real environment.

---

## Tools Used

`minikube`, `kubectl` (apply, get, describe, logs, port-forward, exec, auth can-i, create sa), `base64`

## Notes

Flag values and decoded credentials are redacted from this writeup and its screenshots in line with TryHackMe's platform policy on public content.
