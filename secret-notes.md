# Kubernetes Secrets, ServiceAccounts and Image Pull Secrets - Notes

## 1. What is a Secret?

A Secret stores a small amount of sensitive data, such as a password, token, or key. Using a Secret means you don't put confidential data in application code or in the image.

### Common uses
- Set environment variables in a container.
- Provide credentials such as SSH keys or passwords to Pods (usually as files).
- Let the kubelet pull images from private registries.

### Important
- Secrets are **base64 encoded, not encrypted**. Base64 is not security.
- For production, enable encryption at rest and restrict access with RBAC.
- Never print secret values in logs.
- Secrets are limited to 1 MiB each, like ConfigMaps.

---

## 2. Secret types

| Type | `type:` value | Used for |
|------|---------------|----------|
| Opaque (default) | `Opaque` | Passwords, API keys, general app data |
| ServiceAccount token | `kubernetes.io/service-account-token` | ServiceAccount API token |
| Docker registry | `kubernetes.io/dockerconfigjson` | Private image registry login |
| Basic auth | `kubernetes.io/basic-auth` | Username and password for HTTP Basic auth |
| SSH auth | `kubernetes.io/ssh-auth` | SSH private key |
| TLS | `kubernetes.io/tls` | Certificate and private key for HTTPS and Ingress |
| Bootstrap token | `bootstrap.kubernetes.io/token` | Node bootstrap and cluster join |
| Custom | e.g. `company.com/database-credentials` | Your own organization's format |

If you don't set `type`, Kubernetes uses `Opaque`.

---

## 3. Create a Secret

### Using kubectl
```bash
kubectl create secret generic backend-user --from-literal=backend-username='backend-admin'
kubectl get secret backend-user
kubectl get secret backend-user -o yaml
```

### Using YAML (`stringData` takes plain text, Kubernetes encodes it)
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  username: admin
  password: MyDB@123
```

With `data:` instead, values must already be base64 encoded.

```bash
kubectl apply -f db-secret.yaml
```

---

## 4. Use a Secret as environment variables

### 4a. One key with `env` + `secretKeyRef`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-single-secret
spec:
  containers:
    - name: envars-test-container
      image: nginx
      env:
        - name: SECRET_USERNAME
          valueFrom:
            secretKeyRef:
              name: backend-user
              key: backend-username
```

```bash
kubectl exec -i -t env-single-secret -- /bin/sh -c 'echo $SECRET_USERNAME'
```

### 4b. Several keys from several Secrets
```bash
kubectl create secret generic db-user --from-literal=db-username='db-admin'
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envvars-multiple-secrets
spec:
  containers:
    - name: envars-test-container
      image: nginx
      env:
        - name: BACKEND_USERNAME
          valueFrom:
            secretKeyRef:
              name: backend-user
              key: backend-username
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-user
              key: db-username
```

```bash
kubectl exec -i -t envvars-multiple-secrets -- /bin/sh -c 'env | grep _USERNAME'
```

### 4c. All keys at once with `envFrom` + `secretRef`
Available since Kubernetes v1.6. Each key becomes an environment variable with the same name.

```bash
kubectl create secret generic test-secret --from-literal=username='my-app' --from-literal=password='39528$vdg7Jb'
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envfrom-secret
spec:
  containers:
    - name: envars-test-container
      image: nginx
      envFrom:
        - secretRef:
            name: test-secret
```

```bash
kubectl exec -i -t envfrom-secret -- /bin/sh -c 'echo "username: $username"; echo "password: $password"'
```

**Note:** Keys must be valid environment variable names.

---

## 5. Use a Secret as files (volume)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: secret-file-lab
type: Opaque
stringData:
  username: admin
  password: MyPassword123
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-file-pod
  namespace: secret-file-lab
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/app-secret
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: app-secret
```

```bash
kubectl create namespace secret-file-lab
kubectl apply -f secret-file.yaml
kubectl exec -it secret-file-pod -n secret-file-lab -- ls -l /etc/app-secret
```

**Things to know:**
- Each key becomes a file. The file contents are the decoded value.
- Volume-mounted Secrets **do update** when the Secret changes (after a sync delay). Env vars do not.
- Mounting a volume hides any existing files in the target directory.

---

## 6. SSH auth Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ssh-auth-secret
  namespace: default
type: kubernetes.io/ssh-auth
stringData:
  ssh-privatekey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    YOUR_PRIVATE_KEY
    -----END OPENSSH PRIVATE KEY-----
```

- `ssh-privatekey` is the required key name.
- Kubernetes does not create SSH keys. Generate them with `ssh-keygen`.
- **Never commit the private key to Git.**
- When mounting into a Pod, use `readOnly: true`.

---

## 7. ServiceAccount basics

- A **ServiceAccount** is the identity of a Pod or application. Humans use User Accounts.
- A ServiceAccount is a namespaced API object.
- A ServiceAccount **does not grant permissions**. RBAC (Role and RoleBinding) does that.
- Attach one to a Pod with `serviceAccountName`.

```
ServiceAccount = Who is the Pod?
RBAC           = What is the Pod allowed to do?
```

### Default ServiceAccount
- Every namespace gets a ServiceAccount named `default`.
- A Pod with no `serviceAccountName` uses `default` in its namespace.
- `default` has no special permissions. It only gives the Pod an identity.
- If `default` is deleted, Kubernetes recreates it.

### Common use cases
1. Calling the Kubernetes API.
2. Reading resources such as ConfigMaps, Secrets, or Pods (with RBAC).
3. Cross-namespace access (with RBAC).
4. Authenticating to external services such as cloud APIs.
5. Pulling images from private registries (`imagePullSecrets` on the ServiceAccount).
6. CI/CD tools authenticating to the cluster.
7. Security tools grouping Pods for policy.

---

## 8. Lab: ServiceAccount + Role + RoleBinding + Pod

```bash
kubectl create namespace sa-lab
kubectl create serviceaccount app-sa -n sa-lab
kubectl get serviceaccount -n sa-lab
```

**sa-role.yaml**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: sa-lab
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

**sa-rolebinding.yaml**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-sa-binding
  namespace: sa-lab
subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: sa-lab
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**pod.yaml**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: sa-lab
spec:
  serviceAccountName: app-sa
  containers:
    - name: test
      image: curlimages/curl
      command: ["/bin/sh"]
      args: ["-c", "sleep 3600"]
```

```bash
kubectl apply -f sa-role.yaml
kubectl apply -f sa-rolebinding.yaml
kubectl apply -f pod.yaml
kubectl get role,rolebinding -n sa-lab
kubectl get pod test-pod -n sa-lab -o jsonpath='{.spec.serviceAccountName}'   # expect: app-sa
```

### Inspect the token inside the Pod
```bash
kubectl exec -it test-pod -n sa-lab -- ls /var/run/secrets/kubernetes.io/serviceaccount/
# expect: ca.crt  namespace  token
```

### Call the API from inside the Pod
```bash
kubectl exec -it test-pod -n sa-lab -- sh
```

Inside the Pod:
```sh
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -k \
  -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc/api/v1/namespaces/sa-lab/pods
```

You should get JSON listing the Pods in `sa-lab`.

**Note:** `-k` skips TLS verification. It's fine for a lab. In real use, pass the CA with `--cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt`.

---

## 9. Lab: ServiceAccount token from a manually created Secret

Since Kubernetes 1.24, token Secrets are no longer created automatically. You create one yourself with the annotation below.

```bash
kubectl create namespace sa-secret-lab
kubectl create serviceaccount app-sa -n sa-secret-lab
```

**service-token-secret.yaml**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-sa-token
  namespace: sa-secret-lab
  annotations:
    kubernetes.io/service-account.name: app-sa
type: kubernetes.io/service-account-token
```

```bash
kubectl apply -f service-token-secret.yaml
kubectl describe secret app-sa-token -n sa-secret-lab   # Data: ca.crt, namespace, token
```

The controller fills in the token because of the `service-account.name` annotation.

### Read the token
```bash
TOKEN=$(kubectl get secret app-sa-token -n sa-secret-lab -o jsonpath='{.data.token}' | base64 -d)
echo $TOKEN
```
On macOS, use `base64 -D`.

### Give permissions (a token alone has none)
Use the same `sa-role.yaml` and `sa-rolebinding.yaml` as section 8, with `namespace: sa-secret-lab` in both files.

```bash
kubectl apply -f role.yaml
kubectl apply -f rolebinding.yaml
```

### Test permissions
```bash
kubectl auth can-i list pods --as=system:serviceaccount:sa-secret-lab:app-sa -n sa-secret-lab      # yes
kubectl auth can-i get secrets --as=system:serviceaccount:sa-secret-lab:app-sa -n sa-secret-lab    # no
```

### Mount the token Secret into a Pod
**token-secret-pod.yaml**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: token-secret-pod
  namespace: sa-secret-lab
spec:
  containers:
    - name: test
      image: curlimages/curl
      command: ["/bin/sh"]
      args: ["-c", "sleep 3600"]
      volumeMounts:
        - name: sa-token
          mountPath: /etc/my-token
          readOnly: true
  volumes:
    - name: sa-token
      secret:
        secretName: app-sa-token
```

```bash
kubectl apply -f token-secret-pod.yaml
kubectl exec -it token-secret-pod -n sa-secret-lab -- ls -l /etc/my-token
kubectl exec token-secret-pod -n sa-secret-lab -- cat /etc/my-token/token
```

**Note:** `serviceAccountName: app-sa` in a Pod already mounts a short-lived projected token. Mounting the long-lived token Secret is only needed for special cases. Treat it as sensitive.

---

## 10. Lab: Pull a private image (imagePullSecrets)

```bash
kubectl create namespace imagepull-lab
kubectl config set-context --current --namespace=imagepull-lab
```

Create a Docker Hub access token with pull-only permission. Don't use your password.

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=YOUR_DOCKERHUB_USERNAME \
  --docker-password='YOUR_DOCKERHUB_ACCESS_TOKEN' \
  --docker-email=YOUR_EMAIL \
  -n imagepull-lab

kubectl get secret regcred
kubectl describe secret regcred     # Type: kubernetes.io/dockerconfigjson
```

**Note:** The token is stored in shell history. Use a secure method in production.

**private-pod.yaml**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-nginx-pod
  namespace: imagepull-lab
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: nginx
      image: YOUR_DOCKERHUB_USERNAME/private-nginx:v1
      imagePullPolicy: Always
      ports:
        - containerPort: 80
```

```bash
kubectl apply -f private-pod.yaml
kubectl get pods                    # expect Running, 1/1
kubectl describe pod private-nginx-pod
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Requirements
1. The Secret must be in the **same namespace** as the Pod.
2. The credentials must be valid.
3. The image name and tag must be correct.
4. The worker node must be able to reach the registry.
5. The token must have pull permission.

### Troubleshooting `ImagePullBackOff` / `ErrImagePull`
```bash
kubectl describe pod private-nginx-pod
kubectl get events -n imagepull-lab
kubectl get secret regcred -n imagepull-lab
```
Common causes: wrong username or token, wrong repository or tag, token without access, Secret in another namespace, network or DNS issues.

### Attach the pull Secret to a ServiceAccount (instead of each Pod)
**serviceaccount.yaml**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: private-registry-sa
  namespace: imagepull-lab
imagePullSecrets:
  - name: regcred
```

**private-pod-sa.yaml**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-nginx-sa-pod
  namespace: imagepull-lab
spec:
  serviceAccountName: private-registry-sa
  containers:
    - name: nginx
      image: YOUR_DOCKERHUB_USERNAME/private-nginx:v1
      imagePullPolicy: Always
```

```bash
kubectl apply -f serviceaccount.yaml
kubectl apply -f private-pod-sa.yaml
kubectl describe pod private-nginx-sa-pod
kubectl delete namespace imagepull-lab
```

---

## 11. Secrets with Static Pods

- A **Static Pod** is run directly by the kubelet from a manifest file on the node. The API server doesn't manage it like a normal Pod.
- A Static Pod **cannot** use `secretKeyRef`, `envFrom`, or a Secret volume directly.
- To give it secret data, you need another mechanism, such as a file on the node that you manage yourself.

---

## 12. Env vars vs volumes: summary

| Feature | Env vars (`env` / `envFrom`) | Volume (files) |
|---------|------------------------------|----------------|
| Loads when | Container starts | Mounted at Pod start |
| Updates after Secret changes | No, restart the Pod | Yes, after a sync delay |
| Good for | Short values | Files, certificates, keys, SSH keys |
| Visible in `env` / logs | Yes, can leak | Less exposure |

---

## 13. Cleanup

```bash
kubectl delete pod secret-env-pod secret-envfrom-pod
kubectl delete secret app-secret
kubectl delete namespace secret-env-lab sa-lab sa-secret-lab secret-file-lab imagepull-lab
```

---

## 14. Mistakes and corrections in the original notes

| # | Problem in the original notes | Correction |
|---|-------------------------------|------------|
| 1 | The RoleBinding YAML has `roleRef` indented under the `subjects` item. This is invalid. | `roleRef` must be at the top level, at the same indent as `subjects`. |
| 2 | The SSH Secret YAML has `type:` twice. Duplicate keys are invalid. | Keep one `type: kubernetes.io/ssh-auth` at the top. |
| 3 | The "Using Secrets as files" section has a file block and a heading in the middle of a YAML file, and no apply commands. | Split into a clear Secret + Pod YAML with apply commands. |
| 4 | The "envFrom + secretRef" lab appears twice, with the same file name `secret-envfrom.yaml`. | Keep one copy. |
| 5 | Multi-line `kubectl` commands (`curl -k`, `kubectl auth can-i`) have no `\` at line ends, so they won't run. | Added `\` continuation. |
| 6 | Several file names are used but never created. Examples: `pod.yaml` is used for two Pods, `private-pod-sa.yaml` is used but never written, `role.yaml` and `rolebinding.yaml` are named differently from `sa-role.yaml` and `sa-rolebinding.yaml`. | Each file name is listed before it's applied. |
| 7 | The "Lab: ServiceAccount Token using Secret" step says "Secret lo token automatically populate chestundi" without mentioning that token Secrets are not auto-created since 1.24. | Added the note. |
| 8 | "When a Kubernetes cluster is created..." duplicates the "new namespace" point. | Merged into one line. |
| 9 | Stray text such as "Interview:awns", "Copy ... to clipboard", "Expalin opaque:" and "Using Secrets as Environment Variables" heading in the middle of content. | Removed. |
| 10 | Secret types are listed by name only. The `type:` strings are not given. | Added a table with the exact `type` values. |
| 11 | Notes say "Secrets are encrypted" in some places, and "not automatically encrypted" in others. | Clarified: base64 is encoding, not encryption. Enable encryption at rest. |
| 12 | Volume-mounted Secret updates are not mentioned. | Added in section 5 and 12. |
| 13 | The Static Pod lab has concept only and no steps. | Kept as a concept section. Actual steps depend on the setup. |
| 14 | `curl -k` is used without a warning. | Added a note. |
| 15 | `base64 -d` doesn't work on macOS. | Added `base64 -D` note. |

---

## 15. Quick command reference

```bash
kubectl create secret generic <name> --from-literal=key=value
kubectl create secret docker-registry <name> --docker-server=... --docker-username=... --docker-password=...
kubectl get secrets
kubectl describe secret <name>
kubectl get secret <name> -o jsonpath='{.data.<key>}' | base64 -d
kubectl create serviceaccount <name> -n <ns>
kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<ns>:<sa> -n <ns>
```
