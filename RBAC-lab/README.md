
## Learning Order

```sh
1. Role
   ↓
2. RoleBinding
   ↓
3. User
   ↓
4. kubectl auth can-i
   ↓
5. Read vs Write permissions
   ↓
6. ServiceAccount
   ↓
7. Role + ServiceAccount
   ↓
8. ClusterRole
   ↓
9. ClusterRoleBinding
   ↓
10. ClusterRole + RoleBinding
   ↓
11. Complete real-world project
```

## RBAC Scenario: "TechCorp Kubernetes Platform"

```sh
| User / ServiceAccount | Role       | What they should do                        |
| --------------------- | ---------- | ------------------------------------------ |
| `alice`               | Developer  | Manage Pods in `dev`                       |
| `bob`                 | Developer  | Manage Deployments/Services in `dev`       |
| `david`               | DevOps     | Manage everything in `dev`                 |
| `monitoring-sa`       | Monitoring | Read Pods/Deployments across namespaces    |
| `deploy-sa`           | CI/CD      | Create/update Deployments but nothing else |

```

### The key mental model

Don't memorize YAML first. Think about every RBAC problem as:

```sh
WHO?
  ↓
User / ServiceAccount

WHAT?
  ↓
Resource

DO WHAT?
  ↓
Verb

WHERE?
  ↓
Namespace / Cluster

HOW?
  ↓
Role + Binding
```

For example:

"Bob should be able to update Deployments in dev."

Translate it:

```sh
WHO?
Bob

WHAT?
Deployments

DO WHAT?
update

WHERE?
dev

HOW?
Role + RoleBinding
```

### Create the kind cluster

```sh
kind create cluster --name rbac-lab
```

Check:

```sh
kubectl get nodes
kubectl config current-context
```

Expected context:

```sh
kind-rbac-lab
```

### Create namespaces deployment and service

Create `namespaces.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev

---
apiVersion: v1
kind: Namespace
metadata:
  name: production
```
 Apply:

 ```sh
 kubectl apply -f namespaces.yaml
 ```

### Create `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: nginx
          image: nginx:1.27

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```

check

```sh
kubectl get deployments -n dev
```

output:

```sh
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
backend    2/2     2            2           27s
frontend   2/2     2            2           27s
```

```sh
kubectl apply -f applications.yaml
```

```sh
kubectl get all -n dev
```


 ### Task 1:

 Create `alice.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-manager
  namespace: dev
rules:
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - get
      - list
      - watch
      - create
      - delete

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-pod-manager
  namespace: dev
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-manager
  apiGroup: rbac.authorization.k8s.io

```

Apply:
```sh
kubectl apply -f alice.yaml
```

Test Alice

```sh
kubectl auth can-i get pods --as=alice -n dev
```

`Output: yes`

```sh
kubectl auth can-i create pods --as=alice -n dev
```

`Output: yes`

But:

```sh
kubectl auth can-i get deployments --as=alice -n dev
```

`Output: no`

And:

```sh
kubectl auth can-i get pods --as=alice -n production
```
`Output: no`

 ### Task 2:

Create `bob.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: dev
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - services
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
      - delete

  - apiGroups:
      - apps
    resources:
      - deployments
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
      - delete

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bob-developer
  namespace: dev
subjects:
  - kind: User
    name: bob
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```


Apply:

```sh
kubectl apply -f bob.yaml
```

Test:

```sh
kubectl auth can-i create deployments --as=bob -n dev
```
`Output: yes`

```sh
kubectl auth can-i delete pods --as=bob -n dev
```
`Output: yes`

But:

```sh
kubectl auth can-i get secrets --as=bob -n dev
```
`Output: no`

### Task 3:

Create `david.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: devops-admin
  namespace: dev
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - services
      - configmaps
      - secrets
      - persistentvolumeclaims
    verbs:
      - "*"

  - apiGroups:
      - apps
    resources:
      - deployments
      - replicasets
      - statefulsets
      - daemonsets
    verbs:
      - "*"

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: david-devops
  namespace: dev
subjects:
  - kind: User
    name: david
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: devops-admin
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```sh
kubectl apply -f david.yaml
```

Test:

```sh
kubectl auth can-i delete pods --as=david -n dev
```
`Output: yes`

```sh
kubectl auth can-i create deployments --as=david -n dev
```

`Output: yes`

But:

```sh
kubectl auth can-i delete pods --as=david -n production
```

`Output: no`

### Task 4:

Create `monitoring.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitoring-sa
  namespace: dev

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: monitoring-reader
  namespace: dev
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - services
    verbs:
      - get
      - list
      - watch

  - apiGroups:
      - apps
    resources:
      - deployments
    verbs:
      - get
      - list
      - watch

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: monitoring-reader
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: monitoring-sa
    namespace: dev
roleRef:
  kind: Role
  name: monitoring-reader
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```sh
kubectl apply -f monitoring.yaml
```

Now test the ServiceAccount:

```sh
kubectl auth can-i get pods \
  --as=system:serviceaccount:dev:monitoring-sa \
  -n dev
```

`Output: yes`

Try:

```sh
kubectl auth can-i delete pods \
  --as=system:serviceaccount:dev:monitoring-sa \
  -n dev
```

`Output: no`

### Task 5:


Create `cicd.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deploy-sa
  namespace: dev

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-manager
  namespace: dev
rules:
  - apiGroups:
      - apps
    resources:
      - deployments
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deploy-sa-binding
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: deploy-sa
    namespace: dev
roleRef:
  kind: Role
  name: deployment-manager
  apiGroup: rbac.authorization.k8s.io
```

Test:

```sh
kubectl auth can-i create deployments \
  --as=system:serviceaccount:dev:deploy-sa \
  -n dev
```

`Output: yes`


```sh
kubectl auth can-i update deployments \
  --as=system:serviceaccount:dev:deploy-sa \
  -n dev
```

`Output: yes`

But:

```sh
kubectl auth can-i delete deployments \
  --as=system:serviceaccount:dev:deploy-sa \
  -n dev
```
`Output: no`

And:

```sh
kubectl auth can-i get secrets \
  --as=system:serviceaccount:dev:deploy-sa \
  -n dev
```

`Output: no`


### Task 6:
Create `monitoring-cluster.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-cluster-reader
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - services
      - namespaces
    verbs:
      - get
      - list
      - watch

  - apiGroups:
      - apps
    resources:
      - deployments
      - replicasets
      - daemonsets
      - statefulsets
    verbs:
      - get
      - list
      - watch

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-cluster-reader
subjects:
  - kind: ServiceAccount
    name: monitoring-sa
    namespace: dev
roleRef:
  kind: ClusterRole
  name: monitoring-cluster-reader
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```sh
kubectl apply -f monitoring-cluster.yaml
```

Now:

```sh
kubectl auth can-i get pods \
  --as=system:serviceaccount:dev:monitoring-sa \
  --all-namespaces
```

Expected:

`Output: yes`

And:

```sh
kubectl auth can-i get deployments \
  --as=system:serviceaccount:dev:monitoring-sa \
  --all-namespaces
```

Expected:

`Output: yes`

But:

```sh
kubectl auth can-i delete pods \
  --as=system:serviceaccount:dev:monitoring-sa \
  --all-namespaces
```

Expected:

`Output: no`
