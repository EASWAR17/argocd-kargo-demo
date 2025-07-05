# Auto Promotion and Deployment of Application across multiple environments using Kargo.io

### Overview
This PoC showcases a GitOps-driven CI/CD flow where Kargo detects new frontend image versions and automatically promotes them across dev, test, and prod stages, triggering deployments via ArgoCD at each stage.

### Prerequisites 
- Docker is installed and running
- Minikube
- DockerHub
- Kargo
- ArgoCD

### POC Flow
Step 1:
A developer builds a new version of the frontend application and pushes the Docker image to a DockerHub repository.

Step 2:
Kargo’s warehouse continuously scans the DockerHub repo and detects the newly pushed image version.

Step 3:
With auto-promotion enabled, Kargo promotes the new image across multiple environments—Dev → Staging → Production—without manual intervention.

Step 4:
For each promotion, Kargo updates the GitOps repository with the latest image version and configuration changes.

Step 5:
ArgoCD, integrated with the GitOps repository, automatically syncs the updates and deploys the application to the Minikube Kubernetes cluster.

Result:
A fully automated, promotion-driven CI/CD pipeline powered by Kargo and ArgoCD, enabling zero-touch deployments from code to production. cluster 

### Implementation
- Create a Dockerhub repo and a Github repo
- yml file to create a warehouse in Kargo
```
apiVersion: kargo.akuity.io/v1alpha1
kind: Warehouse
metadata:
  name: nginx-warehouse
  namespace: kargo
spec:
  freightCreationPolicy: Automatic
  interval: 5m0s
  subscriptions:
    - image:
        repoURL: easwar17/frontend
        discoveryLimit: 5
        imageSelectionStrategy: SemVer
        strictSemvers: true
```

- Use some front-end Application and build a docker image for that
- Push the built image to our Dockerhub repo (to check if warehouse detects for new freights)
- Create individual git branch for every single environments and add deployment code for every single branch

deployment.yml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: easwar17/frontend:1.4.0
          ports:
            - containerPort: 8081
```
Service.yml
```
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 8081
      nodePort: 30081
```
Kustomization.yml
```
resources:
  - deployment.yml
  - service.yml
 ```

Create ArgoCD app.yml file for Deployment(varies for each environment)

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/EASWAR17/argocd-kargo-demo.git
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
 ```

- Create Dev stage in Kargo for promotion of image in Dev environment

dev-stage.yml
```
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: dev-stage
  namespace: kargo
spec:
  promotionTemplate:
    spec:
      steps:
        - as: git-clone
          uses: git-clone
          config:
            repoURL: https://github.com/EASWAR17/argocd-kargo-demo.git
            insecureSkipTLSVerify: true
            checkout:
              - branch: main
                path: repo
        - as: yaml-update
          uses: yaml-update
          config:
            path: repo/k8s/deployment.yml
            updates:
              - key: spec.template.spec.containers.0.image
                value: easwar17/frontend:${{ imageFrom("easwar17/frontend", warehouse("nginx-warehouse")).Tag }}
        - as: git-commit
          uses: git-commit
          config:
            path: repo
            message: "test"
        - as: git-push
          uses: git-push
          config:
            path: repo
  requestedFreight:
    - origin:
        kind: Warehouse
        name: nginx-warehouse
      sources:
        direct: true
 ```

- Similarly create yml files for test-stage, prod-1-stage, prod-2-stage for different environments 
- Add git PAT token in Kargo project secrets for accessing GitOps repo
- Create a promotion.yml file for setting up Auto Promotion of artifacts or images in all stages in Kargo

Promotion.yml
```
apiVersion: kargo.akuity.io/v1alpha1
kind: Project
metadata:
  name: kargo
spec:
  promotionPolicies:
    - stage: dev-stage
      autoPromotionEnabled: true
    - stage: test-stage
      autoPromotionEnabled: true
    - stage: prod-1
      autoPromotionEnabled: true
    - stage: prod-2
      autoPromotionEnabled: true
``` 

Test Auto Promotion by building a new image and pushing it to Dockerhub
![image](https://github.com/user-attachments/assets/50973a64-eb8e-4817-bf5a-fc950a6bdcc4)

 
Kargo UI
![image](https://github.com/user-attachments/assets/8ce99389-ee36-49f0-87f0-be5e6f7df1f8)
 

ArgoCD Applications for each environments
![image](https://github.com/user-attachments/assets/7d64e204-dc9b-44c3-8081-2bd3ca8173ba)

 
Deployment Result

![image](https://github.com/user-attachments/assets/41ad48fa-9dc7-4d55-adfd-47ac4b8d11e5)

