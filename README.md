# Codefresh GitOps Certification examples - Level 2 - GitOps at scale

This repository contains examples for the ArgoCD/GitOps
certification workshops (Level 2)

Take the certification yourself at https://codefresh.io/courses/get-gitops-certified/

---

### 2. Handling multiple Applications
#### App of Apps
```bash
argocd app create my-favorite-apps \
--repo https://github.com/codefresh-contrib/gitops-cert-level-2-examples \
--project default \
--sync-policy none \
--path ./app-of-apps/my-app-list \
--dest-namespace argocd \
--dest-server https://kubernetes.default.svc 
```
- Parent app `my-favourite-apps` has a **manual sync policy**, the child apps have **automatic sync policies**
- Changes in the child app <u> manifests </u> will trigger an auto syncornization, changes to the child app <u> definitions </u> will NOT becuase they are synced using our parent app.

#### Mutli Cluster Management
- After initial installation Argo CD can automatically deploy to its own cluster (referred to as “internal”)
- You can add additional external clusters (public/local) with the Argo CD CLI, you first need to ensure you have a valid context in your kubeconfig for the cluster.

```bash
# Creates new Service account on the target cluster and add that target cluster to ArgoCD
argocd cluster add <context name>

# List clusters
argocd cluster list

# Remove a cluster
# Removing a cluster does not remove the Argo CD Applications associated with it.
argocd cluster rm <server>
```

##### Managing grouped Argo CD instances with multiple clusters
- Argo CD **Projects** provide a way to group Argo CD applications when organizations have multiple teams using Argo
- Projects allow you to restrict what is deployed, where apps can be deployed, and what sort of objects (CRDs, RBAC, etc.) can be deployed.

```bash
# Adding a second cluster
# Login to the first cluster from the 2nd cluster
# kubernetes-vm:30443 represents main cluster
argocd login kubernetes-vm:30443 --insecure --username admin --password <passwd>

# Add the cluster - this results in argocd-manager service account created on cluster-2 in kube-system namespace
argocd cluster add default --name cluster-2

# Adding new app to cluster 2
# Executed from main cluster
argocd app create external-app \
--repo https://github.com/theJaxon/gitops-cert-level-2-examples \
--project default \
--sync-policy automatic \
--path ./simple-application \
--dest-namespace default \
--dest-server https://10.5.1.144:6443 # cluster-2 external IP

# Adding new app to our main cluster
argocd app create internal-app \
--repo https://github.com/theJaxon/gitops-cert-level-2-examples \
--project default \
--sync-policy automatic \
--path ./simple-application \
--dest-namespace default \
--dest-server https://kubernetes.default.svc 
```

#### ApplicationSet Generators
- An ApplicationSet uses templated automation to create, modify, and manage multiple Argo CD applications at once, while also targeting multiple clusters and namespaces.
- It's made up of [`generators`](https://argocd-applicationset.readthedocs.io/en/stable/Generators/) that generate the application.
- Generators are responsible for providing a set of key-value pairs, that are then passed into a template with {{param}} styled parameters.
- Template fields within an AppSet spec are used to generate the `application` resource (So the application is generated by combining the **Params** from the generator with fields from the template)
- Generators’ job is to <u> generate these `parameters` </u> and the template’s job is to consume them, and the template is then applied to a Kubernetes cluster as Argo CD Applications.

##### When to use an AppSet
When there's a need to 
- Deploy to multiple Kubernetes clusters
- Deploy to different namespaces
- Deploy to different namespaces on a single Kubernetes cluster
- Deploy from different Git repositories or folders/branches

##### AppSet Example

<details>
<summary> AppSet definition with List Generator </summary>
<p>

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
spec:
  generators:
    - list:
        elements:
          - cluster: engineering-dev
            url: 'https://kubernetes.default.svc'
          - cluster: engineering-prod
            url: 'https://kubernetes.default.svc'
  template:
    metadata:
      name: '{{cluster}}-guestbook'
    spec:
      project: default
      source:
        repoURL: 'https://github.com/argoproj/applicationset.git'
        targetRevision: HEAD
        path: 'examples/list-generator/guestbook/{{cluster}}'
      destination:
        server: '{{url}}'
        namespace: guestbook
```

</p>
</details>

- List generator is passing the {{cluster}} and {{url}} fields into the Application template as the parameters.
- These fields will be rendered as two corresponding Argo CD Applications - one for each defined cluster.

##### Generators
- A Generator informs ApplicationSet on how to generate multiple Applications and how to deploy them. 
- There are currently 6 primary Generators in addition to 2 generators that can be used for combining the primary ones.

###### List Generator
- Generates parameters based on a fixed list. 
- It passes key-value pairs specified within the elements section of the template.
- Enables a manual approach to **control the Application destination** 

###### Cluster Generator
- For each registered cluster, this generator will produce params based on the list of values within the cluster secret.
- These params will then be provided to the application template for each of the clusters


###### Git Generator
- Includes 2 Sub-Types
  1. Directory Generator: Generates params using directory structure of the Git repository
  2. File Generator: Generates params using the **content within JSON/YAML file in Git repository** and reads a configuration file

<details>
<summary> Directory Generator </summary>
<p>

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: appset-demo
spec:
  generators:
    - git:
        repoURL: 'https://github.com/hseligson1/appset-demo.git'
        revision: HEAD
        directories:
          - path: examples/git-dir-generator/apps
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: 'https://github.com/hseligson1/appset-demo.git'
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: 'https://kubernetes.default.svc'
        namespace: default
```

</p>
</details>

```bash
# Directory structure
appset-demo/examples/git-dir-generator/apps

├── app-set.yaml
├── app-1
│   ├── templates
│   ├── Chart.yaml
│   └── values.yaml
└── app-2
    ├── templates
  ├── Chart.yaml
  └── values.yaml
```

- App name is generated based on the directory name

<details>
<summary> File Generator </summary>
<p>

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: appset-demo
spec:
  generators:
    - git:
        repoURL: 'https://github.com/hseligson1/appset-demo.git'
        revision: HEAD
        files:
          - path: examples/git-file-generator/cluster-config/**/config.json
  template:
    metadata:
      name: '{{cluster.name}}-app-1'
    spec:
      project: default
      source:
        repoURL: 'https://github.com/hseligson1/appset-demo.git'
        targetRevision: HEAD
        path: examples/git-file-generator/apps/app-1
      destination:
        server: '{{cluster.address}}'
        namespace: argocd
```

</p>
</details>

<details>
<summary> config.json </summary>
<p>

```json
{
  "aws_account": "123456",
  "asset_id": "11223344",
  "cluster": {
    "owner": "cluster-admin@codefresh.com",
    "name": "environments-staging",
    "address": "https://1.2.3.4"
  }
}
```

</p>
</details>

- Whenever a change is made to the config.json file within the cluster-config folder, it will automatically be discovered by the git-generator.

###### SCM Provider Generator
- Used to discover and iterate over whole Git repositories (For example: whole repos of your organization)
- Can be used to automate the creation of an environment/application whenever somebody creates a new repository in the organization

###### Pull Request Generator
- Can iterate over PRs from Github and then create environments per PR (Great for creating temporary environment when a PR is created)

###### Cluster Decision Resource Generator
- Can discover cluster based on specific K8s resources of any type that satisfy a set of criteria

###### Matrix/Merge Generator
- Used to combine primary generators 
- Example: if you already have a Cluster generator for all your clusters and Git generator for all your apps, you can combine them with a matrix generator to deploy all your apps to all your clusters.


```bash
# Generates apps based on list generator
argocd app create my-application-sets \
--repo https://github.com/theJaxon/gitops-cert-level-2-examples \
--project default \
--sync-policy automatic \
--path ./application-sets/my-application-sets/ \
--dest-namespace default \
--dest-server https://kubernetes.default.svc 
```

<details>
<summary> Git Generator example 2 </summary>
<p>

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: many-apps-application-set
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/codefresh-contrib/gitops-cert-level-2-examples.git
      revision: HEAD
      directories:
      - path: application-sets/example-apps/*
  template:      
    metadata:
      name: '{{path.basename}}'
    spec:
      # The project the application belongs to.
      project: default

      # Source of the application manifests
      source:
        repoURL: https://github.com/codefresh-contrib/gitops-cert-level-2-examples.git
        targetRevision: HEAD
        path: '{{path}}'
      
      # Destination cluster and namespace to deploy the application
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'

      # Sync policy
      syncPolicy:
        syncOptions:
          - CreateNamespace=true  
        automated: # automated sync by default retries failed attempts 5 times with following delays between attempts ( 5s, 10s, 20s, 40s, 80s ); retry controlled using `retry` field.
          prune: true # Specifies if resources should be pruned during auto-syncing ( false by default ).
          selfHeal: true # Specifies if partial app sync should be executed when resources are changed only in target Kubernetes cluster and no git change detected ( false by default ).
      
```

</p>
</details>

- `path.basename` matches exactly the name of the last directory so here full `path` evaluates to `application-sets/example-apps/prometheus` in case of prometheus, basename will evaluate only to `prometheus`