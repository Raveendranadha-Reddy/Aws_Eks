# Kubernetes ConfigMap - Notes

## 1. What is a ConfigMap?

A ConfigMap is a Kubernetes API object that stores **non-confidential** configuration data as key-value pairs. It keeps application configuration separate from the container image, so you don't hard-code settings inside the image.

For passwords, tokens, or keys, use a **Secret** instead.

### Limits
- A ConfigMap can hold at most **1 MiB** of data.
- For larger configuration, store it in a volume, a database, or a file storage service.
- All values in `data` must be **strings**. Quote numbers and booleans, for example `"8080"` and `"true"`.

---

## 2. Ways to use a ConfigMap in a Pod

| # | Method | How the app sees it |
|---|--------|---------------------|
| 1 | Container command and arguments | Values are injected as env vars, then used in `command` / `args` |
| 2 | Environment variables | Plain env vars inside the container |
| 3 | Read-only volume | Files inside a mounted directory |
| 4 | Kubernetes API from app code | App reads the ConfigMap directly from the cluster |

---

## 3. Method 1: In container commands and arguments

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: practice-config
data:
  student_name: "Raveendra"
  course: "Kubernetes"
---
apiVersion: v1
kind: Pod
metadata:
  name: practice-pod
spec:
  containers:
    - name: practice-container
      image: busybox
      command: ["/bin/sh", "-c"]
      args:
        - echo "Hello $STUDENT_NAME, Welcome to $COURSE"
      env:
        - name: STUDENT_NAME
          valueFrom:
            configMapKeyRef:
              name: practice-config
              key: student_name
        - name: COURSE
          valueFrom:
            configMapKeyRef:
              name: practice-config
              key: course
```

```bash
kubectl apply -f configmap-command.yaml
kubectl apply -f pod-command.yaml
kubectl logs practice-pod
```

**Note:** In Kubernetes manifests, `$(VAR_NAME)` is expanded by Kubernetes. A plain `$VAR` is expanded by the shell. Both work here because the variables are already set in the container's environment.

---

## 4. Method 2: As environment variables

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_NAME: "Coffee-Shop"
  APP_ENV: "Development"
  APP_PORT: "8080"
---
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app-container
      image: busybox
      command: ["/bin/sh", "-c"]
      args:
        - echo "Application: $APP_NAME"; echo "Environment: $APP_ENV"; echo "Port: $APP_PORT"
      env:
        - name: APP_NAME
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_NAME
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV
        - name: APP_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_PORT
```

**Shortcut:** To load every key at once, use `envFrom` instead of listing each key:

```yaml
      envFrom:
        - configMapRef:
            name: app-config
```

---

## 5. Method 3: Read-only volume (config file)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-file
data:
  app.properties: |
    app.name=Coffee-Shop
    app.environment=Development
    app.port=8080
---
apiVersion: v1
kind: Pod
metadata:
  name: app-volume-pod
spec:
  containers:
    - name: app-container
      image: busybox
      command: ["/bin/sh", "-c"]
      args:
        - cat /etc/app-config/app.properties
      volumeMounts:
        - name: config-volume
          mountPath: /etc/app-config
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: app-config-file
```

```bash
kubectl apply -f configmap-volume.yaml
kubectl logs app-volume-pod
```

**Things to know:**
- Each key becomes a file. The file name is the key, and the content is the value.
- Mounting a volume **replaces the entire target directory**. Any existing files in `/etc/app-config` are hidden.
- To add one file without hiding the rest of the directory, use `subPath`. Files mounted with `subPath` do **not** receive automatic updates.
- Volume-mounted ConfigMaps **do update** when the ConfigMap changes (after a short sync delay). Env vars do not.

---

## 6. Method 4: Read the ConfigMap through the Kubernetes API

The app reads the ConfigMap directly using the Kubernetes API. The Pod needs a ServiceAccount with permission to read it.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  APP_NAME: "Coffee-Shop"
  APP_ENV: "Development"
  APP_PORT: "8080"
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: configmap-reader-binding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: default
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Pod
metadata:
  name: app-api-pod
  namespace: default
spec:
  serviceAccountName: app-sa
  containers:
    - name: app
      image: your-application-image:latest   # replace with your real image
```

**Things to know:**
- The Role needs only `get` for reading one ConfigMap. Add `list` or `watch` only if the app needs them.
- The `your-application-image:latest` line is a placeholder.
- This method is more work than the others. Use it only when the app must re-read config at runtime.

---

## 7. Mistakes and corrections in the original notes

| # | Problem in the original notes | Correction |
|---|-------------------------------|------------|
| 1 | Intro says "two ways" (command-line args or volume files), but the list below says "four ways". | Use the four-method list. |
| 2 | Method 1 is titled "container commands and arguments", but the example only uses env vars. | Method 1 is env injection plus `$(VAR)` or `$VAR` in `args`. The example is correct, but the title can mislead. |
| 3 | Method 1 and Method 2 use env vars in the same way, so the difference is not clear. | Method 1 focuses on using the value in the command. Method 2 focuses on the env var being available to the app. |
| 4 | Method 3 and Method 4 both use the name `app-config`, and the Pod name `app-pod` is reused. Applying them together causes a conflict. | Use separate names for each example. This version uses `app-config-file`, `app-volume-pod`, `app-api-pod`. |
| 5 | The notes don't say that env vars do not update after a ConfigMap changes. | Env vars need a Pod restart to pick up changes. Volume files update automatically. |
| 6 | The notes don't say that a volume mount hides existing files in the target directory. | Use `subPath` for single files. |
| 7 | The notes don't mention that values must be strings. | Quote all values, for example `"8080"`. |
| 8 | Some lines are formatting fragments, for example "ConfigMaps as environment variables." and "-----". | Removed in this version. |
| 9 | The note "Pods can consume the ConfigMaps environment variables" is unclear. | Rewritten as a plain statement in section 2. |

---

## 8. Quick command reference

```bash
kubectl apply -f <file>.yaml           # create or update
kubectl get configmap                  # list ConfigMaps
kubectl describe configmap <name>      # show contents
kubectl get configmap <name> -o yaml   # show full YAML
kubectl logs <pod-name>                # see app output
kubectl delete configmap <name>        # delete
```

After changing a ConfigMap used as env vars, restart the Pod. Deleting the Pod is enough if it's managed by a Deployment, because the Deployment recreates it.
