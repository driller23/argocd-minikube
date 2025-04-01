# GitOps with ArgoCD - A Practical Guide

## What is ArgoCD?

Argo CD is a Kubernetes-native continuous deployment (CD) tool. Unlike traditional external CD tools that rely on push-based deployments, Argo CD can pull updated code from Git repositories and deploy it directly to Kubernetes resources, enabling a GitOps workflow.

![ArgoCD Architecture](https://raw.githubusercontent.com/youruser/yourrepo/main/images/argocd.png)

## What is GitOps?

GitOps is a modern approach to implementing Continuous Deployment for cloud-native applications. It centers around a developer-friendly experience for managing infrastructure by leveraging familiar tools such as Git and existing CD tooling.

The fundamental concept of GitOps is maintaining a Git repository that contains declarative descriptions of the infrastructure desired in your production environment, coupled with an automated process that ensures the production environment matches the described state in the repository. With this approach, deploying or updating applications simply requires updating the repository—the automated process handles everything else. It's essentially cruise control for your production application management.

![GitOps Flow](https://raw.githubusercontent.com/youruser/yourrepo/main/images/gitops.png)

## How to Implement GitOps using Argo CD

Most organizations use Git for source code management. With GitOps, developers commit infrastructure configurations (like Kubernetes resource definitions) to Git repositories to create environments for application deployment.

The workflow typically looks like this:

1. Developer implements a feature (with new application code and Kubernetes configurations)
2. Changes are merged to the main branch, triggering the CI process to build and test an image
3. After review and approval, the pull request is merged to the main branch
4. The GitOps agent (ArgoCD) detects the new configuration versions and compares them with the running application in the target environment
5. If there's a mismatch, ArgoCD highlights an out-of-sync status
6. ArgoCD uses the Kubernetes controller to reconcile the new changes to cluster resources
7. Once the Kubernetes resources are updated, ArgoCD reports that the application is in sync

ArgoCD continuously monitors the environment using an agent that compares the current state with the declared state in Git. This ensures new configurations are correctly deployed and allows for easy rollback to previous states with a single click, as all change records are stored in Git.

## Benefits of Argo CD

### 1. Improved Developer Productivity
ArgoCD provides self-service environments for application deployment, allowing development teams to focus on creating business logic rather than spending time on manual deployment tasks.

### 2. Enhanced Software Delivery Compliance
Enable all teams (Dev, Ops, and DevOps) to use a unified platform for infrastructure change management. Apply organizational policies to control access to Kubernetes resources and minimize application downtime.

### 3. Increased Collaboration in SDLC
When using ArgoCD, team members work from the same system to implement GitOps and monitor individual process statuses. The single Git repository promotes collaboration by enabling task assignment and code deployment from each contributor as needed.

### 4. Faster Deployments
ArgoCD enables faster deployments to Kubernetes clusters across multi-cloud environments. Quicker application updates mean reduced time to market and greater flexibility in responding to customer needs.

## Prerequisites

- A running Kubernetes cluster

## Installation Guide

For this tutorial, you need a running Kubernetes cluster (like minikube).

### 1. Create a namespace for ArgoCD

```bash
kubectl create namespace argocd
```

### 2. Install ArgoCD

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 3. Verify the installation

Check what resources have been created:

```bash
kubectl get all -n argocd
```

You should see output similar to:

```
NAME                                                    READY   STATUS    RESTARTS   AGE
pod/argocd-application-controller-0                     1/1     Running   0          106m
pod/argocd-applicationset-controller-787bfd9669-4mxq6   1/1     Running   0          106m
pod/argocd-dex-server-bb76f899c-slg7k                   1/1     Running   0          106m
pod/argocd-notifications-controller-5557f7bb5b-84cjr    1/1     Running   0          106m
pod/argocd-redis-b5d6bf5f5-482qq                        1/1     Running   0          106m
pod/argocd-repo-server-56998dcf9c-c75wk                 1/1     Running   0          106m
pod/argocd-server-5985b6cf6f-zzgx8                      1/1     Running   0          106m

NAME                                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/argocd-applicationset-controller          ClusterIP   10.102.163.101   <none>        7000/TCP,8080/TCP            106m
service/argocd-dex-server                         ClusterIP   10.101.227.215   <none>        5556/TCP,5557/TCP,5558/TCP   106m
service/argocd-metrics                            ClusterIP   10.111.59.189    <none>        8082/TCP                     106m
service/argocd-notifications-controller-metrics   ClusterIP   10.96.102.185    <none>        9001/TCP                     106m
service/argocd-redis                              ClusterIP   10.97.229.117    <none>        6379/TCP                     106m
service/argocd-repo-server                        ClusterIP   10.102.16.58     <none>        8081/TCP,8084/TCP            106m
service/argocd-server                             ClusterIP   10.98.71.135     <none>        80/TCP,443/TCP               106m
service/argocd-server-metrics                     ClusterIP   10.109.248.207   <none>        8083/TCP                     106m

NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/argocd-applicationset-controller   1/1     1            1           106m
deployment.apps/argocd-dex-server                  1/1     1            1           106m
deployment.apps/argocd-notifications-controller    1/1     1            1           106m
deployment.apps/argocd-redis                       1/1     1            1           106m
deployment.apps/argocd-repo-server                 1/1     1            1           106m
deployment.apps/argocd-server                      1/1     1            1           106m

NAME                                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/argocd-applicationset-controller-787bfd9669   1         1         1       106m
replicaset.apps/argocd-dex-server-bb76f899c                   1         1         1       106m
replicaset.apps/argocd-notifications-controller-5557f7bb5b    1         1         1       106m
replicaset.apps/argocd-redis-b5d6bf5f5                        1         1         1       106m
replicaset.apps/argocd-repo-server-56998dcf9c                 1         1         1       106m
replicaset.apps/argocd-server-5985b6cf6f                      1         1         1       106m

NAME                                             READY   AGE
statefulset.apps/argocd-application-controller   1/1     106m
```

### 4. Access the ArgoCD UI

To access the ArgoCD UI, you need to modify the service to use NodePort:

```bash
kubectl edit svc argocd-server -n argocd  # Change service type to NodePort
```

### 5. Get login credentials

The default username is `admin`. To get the password:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o yaml
```

The password is base64 encoded, so decode it:

```bash
echo "<secret-value>" | base64 --decode
```

Now you can log in to the ArgoCD UI using these credentials.

## Sample Application Deployment

Let's deploy a sample application using ArgoCD. Here are the manifest files:

### Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: swiggy-app
  labels:
    app: swiggy-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: swiggy-app
  template:
    metadata:
      labels:
        app: swiggy-app
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: swiggy-app
        image: veeranarni/hotstar:latest
        imagePullPolicy: "Always"
        ports:
        - containerPort: 3000
```

### Service YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: swiggy-app
  labels:
    app: swiggy-app
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 3000
  selector:
    app: swiggy-app
```

## Setting Up ArgoCD Application

To configure ArgoCD to sync with your Git repository, create an Application manifest:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-argo-application
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/CloudTechDevOps/Kubernetes.git
    targetRevision: HEAD
    path: day-14-argocd
  destination: 
    server: https://kubernetes.default.svc
    namespace: myapp

  syncPolicy:
    syncOptions:
    - CreateNamespace=true

    automated:
      selfHeal: true
      prune: true
```

### Understanding the Application Manifest

- `argoproj.io/v1alpha1` is the API version of ArgoCD (check the documentation for the latest version)
- `repoURL` specifies your repository URL
- `targetRevision` is set to HEAD to fetch the latest commit
- `path` specifies the folder containing your application manifests
- `destination.server` is set to `https://kubernetes.default.svc`, the internal Kubernetes API Server service
- `destination.namespace` defines where to deploy your application
- `syncPolicy.syncOptions.CreateNamespace=true` allows ArgoCD to create the namespace if it doesn't exist
- `automated.selfHeal: true` ensures ArgoCD overrides manual changes with Git definitions
- `automated.prune: true` enables ArgoCD to delete resources when they're removed from Git

Apply this configuration:

```bash
kubectl apply -f application.yaml
```

## Working with ArgoCD

Once deployed, you can view your application in the ArgoCD UI and see details like:
- Application status
- Deployment workflow
- Manifest files
- Pod creation events

### Making Changes

To update your application, simply commit changes to your Git repository. For example, to scale your application, modify the `replicas` value in your deployment YAML:

```yaml
spec:
  replicas: 4  # Changed from 3 to 4
```

After committing this change, ArgoCD will detect it (typically within 3 minutes) and apply it to your cluster. You can verify the change:

```bash
kubectl get pods -n myapp
```

Output should show 4 pods instead of 3:

```
NAME                                READY   STATUS    RESTARTS   AGE
myapp-deployment-544dd58bc4-4sntz   1/1     Running   0          13h
myapp-deployment-544dd58bc4-wkf5j   1/1     Running   0          13h
myapp-deployment-544dd58bc4-xt7hb   1/1     Running   0          13h
myapp-deployment-544dd58bc4-zjmn8   1/1     Running   0          13h
```

## Advanced Configuration

For faster synchronization, you can implement webhooks to trigger ArgoCD immediately after Git commits instead of waiting for the default 3-minute polling interval.

## Conclusion

ArgoCD provides a powerful way to implement GitOps for Kubernetes environments. By following the principle of declarative configuration stored in Git, you can achieve more reliable, auditable, and automated deployments while improving team collaboration and productivity.

---

Feel free to star ⭐ this repository if you found it helpful!
