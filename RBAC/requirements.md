
### 1. Alice — Pod Manager

Requirement

Alice is a developer.

She can:

```sh
get pods
list pods
watch pods
create pods
delete pods
```

Only in:

```sh
dev
```

She cannot access Deployments or production.

### 2. Bob — Developer

Bob manages the applications.

He can manage:

```sh
Pods
Deployments
Services
```

with:

```sh
get
list
watch
create
update
patch
delete
```

`But he cannot access Secrets.`

### 3. David — DevOps

For learning purposes, we'll give him full permissions over common application resources.

### 4. ServiceAccount — Monitoring

Now we'll move from humans to applications.

### 5. CI/CD ServiceAccount
Now create a ServiceAccount for your CI/CD pipeline.

Requirement:

```sh
deploy-sa

Can:
  get deployments
  list deployments
  watch deployments
  create deployments
  update deployments
  patch deployments

Cannot:
  delete deployments
  access secrets
  create pods
```

### 6. ClusterRole — Monitoring Everywhere

Now suppose monitoring needs to read resources from all namespaces.

Instead of creating a Role in every namespace, create a ClusterRole.

