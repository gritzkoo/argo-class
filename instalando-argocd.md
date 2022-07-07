# How to setup local machine to work with kubernetes helm kind and argo cd using argo autopilot

## Install golang

Visit <https://golang.org/download> and then extract the `tar`file in you user home folder, then export PATH to include `$HOME/go/bin`

## Install kind

```sh
go install sigs.k8s.io/kind@latest
```

## Install kubectl

```sh
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

## Install helm

```sh
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
## Init kind cluster

```sh
kind create cluster
```

## Install ARGO-CD in a kubernetes cluster

```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/codefresh-contrib/gitops-certification-examples/main/argocd-noauth/install.yaml
kubectl get pods -n argocd
```

When you decide how to access your cluster, the example below will create a simple node port foward to first access

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: argocd-server
  managedFields:
  name: argocd-server-nodeport
  namespace: argocd
spec:
  ports:
  - nodePort: 30443
    port: 8080
    protocol: TCP
  selector:
    app.kubernetes.io/name: argocd-server
  type: NodePort
``` 

Now, using the `yaml` above, apply it to k8s using the command below:

```sh
kubectl apply -f service.yml
```

## Getting admin password to first login into argocd UI

```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d > admin-pass.txt
```

## installing argocd cli

```sh
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.1.5/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

> login in localhost

```sh
argocd login localhost:30443 --insecure
```

>use the password you get into the `admin-pass.txt` to complete login


## creating an app via cli

```sh
argocd app create {APP NAME} \
  --project {PROJECT} \
  --repo {GIT REPO} \
  --path {APP FOLDER} \
  --dest-namespace {NAMESPACE} \
  --dest-server {SERVER URL}
```

### Explaning argo strategies

| Policy        | A      | B        | C        | D        | E       |
| :------------ | :----- | :------- | :------- | :------- | :------ |
| Sync Strategy | Manual | Auto     | Auto     | Auto     | Auto    |
| Auto-prune    | N/A    | Disabled | Enabled  | Disabled | Enabled |
| Self-heal     | N/A    | Disabled | Disabled | Enabled  | Enabled |

### verify the reconciliation period

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  timeout.reconciliation: 240s # <<<<<here goes the time to polling github
```

## defaul argo app

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: canary-demo
spec:
  destination:
    server: 'https://kubernetes.default.svc' 
    namespace: demo
  project: default
  source:
    repoURL: 'https://github.com/kostis-codefresh/summer-of-k8s-app-manifests
    path: ./
    targetRevision: HEAD
```

## argo rollout

Is an application that has to be installed in the cluster before adding new projects.

The step to it is:

- create application `argo-rollouts-controller` 
  - pointing to the: <https://github.com/gritzkoo/gitops-certification-examples>
  - to path: `./argo-rollouts-controller`
  - using: `automatic` healling
  - [x] auto create namespaces
  - namespace: `argo-rollouts`

Then you can create new applications

```sh
kubectl argo rollouts list rollouts
kubectl argo rollouts status simple-rollout
kubectl argo rollouts get rollout simple-rollout
# commands to interact with
kubectl argo rollouts get rollout simple-rollout
kubectl argo rollouts promote simple-rollout
kubectl argo rollouts get rollout simple-rollout --watch
```
## canary deploys

![image canary deploy1](https://lwfiles.mycourse.app/codefresh-public/6f68ef344282a66847d474de69eaf951.png)
![image canary deploy2](https://lwfiles.mycourse.app/codefresh-public/8fa31fd54fb3427301bce4f0f3c1f9b2.png)
![image canary deploy3](https://lwfiles.mycourse.app/codefresh-public/b77a8baf0569b53be3357c6f6c9d5efc.png)
![image canary deploy4](https://lwfiles.mycourse.app/codefresh-public/9a73456a2c6a6e6b5078b018b12b3174.png)


## automated rollouts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
  - name: service-name
  metrics:
  - name: success-rate
    interval: 2m
    count: 2
    # NOTE: prometheus queries return results in the form of a vector.
    # So it is common to access the index 0 of the returned array to obtain the value
    successCondition: result[0] >= 0.95
    provider:
      prometheus:
        address: http://prom-release-prometheus-server.prom.svc.cluster.local:80
        query: sum(response_status{app="{{args.service-name}}",role="canary",status=~"2.*"})/sum(response_status{app="{{args.service-name}}",role="canary"}
```

## example of deployment

 ```yaml
 apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: canary-demo
spec:
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: demo
  project: default
  source:
    repoURL: 'https://github.com/kostis-codefresh/summer-of-k8s-app-manifests'
    path: ./
    targetRevision: HEAD
```

### concepts

### Part 1 About GitOps

- What is GitOps? : 
  - `It is a set of best-practices for deployments.`
- How are GitOps and DevOps related?: 
  - `DevOps is a paradigm/mindset. GitOps is a set of best practices.`
- Is the following statement true or false?
  - GitOps is only for Kubernetes applications.: `false`
- What is the major advantage of GitOps?
  - `Eliminating configuration drift.` 
- What is a major disadvantage of GitOps?
  - `GitOps handles only deployments`

### Part 2 Argo CD Basics

- What is Argo CD?
  - `A GitOps Agent` 
- Which other Argo products does Argo CD need to function correctly?
  - `None. Argo CD is a standalone project. It works great with the other Argo projects, but it does not depend on them.`
- How does ArgoCD interact with clusters?
  - `You can have any combination of clusters and ArgoCD instances. ArgoCD can deploy applications on the cluster it is installed on, or other external clusters that are authenticated correctly`
- How can you install ArgoCD on your cluster?
  - `You can use any of the above including other community methods`
    - You must use a Kubernetes manifest from Git
    - You must use the official Helm chart
    - You must use a friendly installer such as Argo Autopilot
- What is the relationship between the ArgoCD Web interface and the Argo CD Command line executable?
  - `The Argo CD UI and the CLI can be used interchangeably according to your needs.`


### Part 3 Ugins ArgoCD

- Is the following statement true or false?
  - If you have enabled the "auto-sync" option in an Argo CD application and something is changed manually in the cluster, then Argo CD will automatically discard the change.: `false` 
    - >Correct! Unless you have also setup the "self-heal" option, Argo CD will never discard manual changes
- Is the following statement true or false?
  - If you have enabled the "auto-sync" option in an Argo CD application and you delete a resource in Git, then Argo CD will automatically delete that resource from the cluster as well. `false`
    - >Correct! Unless you have also setup the "auto-prune" option, Argo CD will never remove resources from the live cluster
- What is the proper way to handle application secrets via GitOps?
  - Encrypt them and store them in Git. Then decrypt them during runtime.
    - >Correct! That is the proper way to handle secrets with GitOps
- If you use Bitnami Sealed Secrets, then where does encryption and decryption take place?
  - `Encryption happens via the kubeseal executable. Decryption happens via the Sealed Secrets controller.`
- You have just logged in the Argo CD UI and created an application using a Git repository that holds your Helm chart. You sync the application, and everything is fine. What is the next step that you should take?
  - `Create a declarative file of the application and other resources (e.g. Argo CD project) used and store them in Git.`
- You just created a Helm application using the Argo CD web interface. Now you go the command line and you enter helm list. To your surprise nothing is printed.
  - `The helm command will never work no matter what you do in Argo CD`
- What kind of applications can Argo CD deploy?
  - ArgoCD can only deploy plain Kubernetes manifests.
  - ArgoCD can only deploy Helm applications
  - ArgoCD can only deploy Kustomize applications
  - >ArgoCD can deploy all of the above

### Part 4 Progressive Delivery

- What is Progressive Delivery?
  - `A way to gradually deploy applications minimizing downtime`
- What is Argo Rollouts
  - `A Kubernetes controller for progressive delivery`
- What is the relationship between Argo CD and Argo Rollouts
  - `Argo CD and Argo Rollouts can either be deployed individually or both at the same time.`
- What are Blue/Green deployments?
  - `A deployment method where the new version is launched while the old version is still running. Both version exist during the deployment`
- What are Canary deployments
  - `A deployment method where only a subset of live users get access to the new version of the application`
- How does Argo Rollout work?
  - `You need to convert your Kubernetes Deployment to a Rollout resource to enable progressive delivery`


6rp9S5%h07MB1qjv