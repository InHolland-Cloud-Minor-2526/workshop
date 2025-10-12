### OpenShift CI/CD Workshop Workbook

#### Who this is for
- English-speaking HBO (master’s level) students
- Comfortable with OOP (Java, .NET)
- Some cloud exposure (AWS/Azure)
- Everyone is in the GitHub org: [InHolland-Cloud-Minor-2526](https://github.com/InHolland-Cloud-Minor-2526)

#### What you’ll build
By the end, you’ll have:
- An OpenShift setup created by a PaasOperator PaaS.yaml
- Multiple OpenShift projects with your team members and basic capabilities
- CI with GitHub Actions that builds and pushes a container to GHCR
- A separate deploy repo (projectname-deploy) with manifests: Deployment, Service, Route, ConfigMap
- ArgoCD watching that deploy repo and auto-deploying changes

#### Prep before we start (instructor or ops)
- OpenShift cluster ready and PaasOperator installed: [opr-paas](https://github.com/belastingdienst/opr-paas)
- Students have OpenShift access and permissions
- ArgoCD installed and reachable
- GitHub org ready: [InHolland-Cloud-Minor-2526](https://github.com/InHolland-Cloud-Minor-2526)
- GHCR enabled at org level
- Optional templates (nice to have):
  - App template (simple Java/.NET + Dockerfile)
  - projectname-deploy template (manifests + optional kustomize)
  - projectname-argocd template (Application YAML)
- Student laptops: Git, browser, optional Docker/Podman, Java/.NET SDK if building locally

---

### Schedule at a glance
- Part 0: Intro + architecture (20–30 min)
- Part 1: PaasOperator + PaaS.yaml (45–60 min)
- Part 2: CI with GitHub Actions + GHCR (60 min)
- Part 3: Deployment repo with manifests (45–60 min)
- Part 4: ArgoCD for CD (45–60 min)
- Part 5: Validate + troubleshoot + stretch goals (30–45 min)
- Wrap-up (10–15 min)

Each part starts with a short mini-talk, then hands-on.

---

### Part 0: Intro and the big picture

#### Quick talk
- What OpenShift adds on top of Kubernetes
- Why GitOps and why ArgoCD
- What the PaasOperator does (declarative platform setup)
- Flow:
  - Push code → GitHub Actions builds image → push to GHCR
  - ArgoCD notices changes in the deploy repo → applies to OpenShift
- Repos you’ll have:
  - App repo (code + Dockerfile + CI)
  - Deploy repo (manifests)
  - ArgoCD repo (Application definition)

Outcome: You know what you’re building and how the pieces fit.

---

### Part 1: Set up OpenShift with PaasOperator

#### Quick talk
- What the PaaS.yaml controls
- Creating projects, adding people, turning on capabilities
- Basic security and permissions

#### Hands-on goals
- Create a PaaS.yaml that:
  - Creates a few projects: dev, test, prod
  - Adds team members
  - Enables basic capabilities (routes, builds, monitoring if available)

#### Steps
1) Create a repo (or folder) for platform files, e.g. projectname-platform
2) Add a PaaS.yaml. Use the operator docs for exact fields. Example structure:
   ```
   apiVersion: paas.belastingdienst.nl/v1
   kind: PaaS
   metadata:
     name: projectname
   spec:
     projects:
       - name: projectname-dev
         capabilities:
           - routes
           - builds
           - monitoring
         members:
           - github:username1
           - github:username2
       - name: projectname-test
         capabilities:
           - routes
           - builds
         members:
           - github:username1
           - github:username2
       - name: projectname-prod
         capabilities:
           - routes
         members:
           - github:username1
           - github:username2
   ```
3) Apply it:
   - oc apply -f PaaS.yaml
4) Check it worked:
   - oc get namespaces | grep projectname
   - oc get rolebindings -n projectname-dev

Common gotchas:
- Field names must match the CRD. Copy from the operator’s examples.
- Make sure you have permissions to apply the CR.

---

### Part 2: CI with GitHub Actions + GHCR

#### Quick talk
- GitHub Actions basics (triggers, jobs, permissions)
- GHCR (registry naming, auth, visibility)
- We’ll build with Docker using Actions

#### Hands-on goals
- App repo with Dockerfile
- CI workflow that builds and pushes to GHCR on push to main

#### Steps
1) Create app repo: projectname-app
2) Add a tiny app:
   - Java Spring Boot “Hello” or .NET minimal API
3) Dockerfile examples

   Java:
   ```
   FROM eclipse-temurin:21-jre-alpine
   WORKDIR /app
   COPY target/app.jar app.jar
   EXPOSE 8080
   CMD ["java", "-jar", "app.jar"]
   ```

   .NET:
   ```
   FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine
   WORKDIR /app
   EXPOSE 8080
   COPY ./publish/ .
   ENTRYPOINT ["dotnet", "YourApp.dll"]
   ```
4) Confirm GHCR is allowed at org level
5) Add .github/workflows/ci.yml
   ```
   name: Build and Push to GHCR
   on:
     push:
       branches: [ "main" ]
   jobs:
     build:
       runs-on: ubuntu-latest
       permissions:
         contents: read
         packages: write
       steps:
         - uses: actions/checkout@v4

         - name: Log in to GHCR
           uses: docker/login-action@v3
           with:
             registry: ghcr.io
             username: ${{ github.actor }}
             password: ${{ secrets.GITHUB_TOKEN }}

         - name: Extract metadata (tags, labels)
           id: meta
           uses: docker/metadata-action@v5
           with:
             images: ghcr.io/${{ github.repository }}

         - name: Build and push
           uses: docker/build-push-action@v6
           with:
             context: .
             push: true
             tags: ${{ steps.meta.outputs.tags }}
             labels: ${{ steps.meta.outputs.labels }}
   ```
6) Push to main and watch the workflow
7) Verify the image under Packages in the repo/org

Common gotchas:
- 401 on push: check org permissions for packages
- Wrong Docker context or path

---

### Part 3: Deployment repo (projectname-deploy)

#### Quick talk
- Why a separate deploy repo (GitOps)
- The four basics: Deployment, Service, Route, ConfigMap
- Tag strategy: “latest” is fine for a workshop; real life prefers immutable tags

#### Hands-on goals
- Repo with base manifests that reference your GHCR image

#### Steps
1) Create repo: projectname-deploy
2) Create manifests/ folder
3) ConfigMap:
   ```
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: app-config
     namespace: projectname-dev
   data:
     MESSAGE: "Hello from ConfigMap"
   ```
4) Deployment:
   ```
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: projectname-app
     namespace: projectname-dev
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: projectname-app
     template:
       metadata:
         labels:
           app: projectname-app
       spec:
         containers:
           - name: app
             image: ghcr.io/inholland-cloud-minor-2526/projectname-app:latest
             ports:
               - containerPort: 8080
             env:
               - name: MESSAGE
                 valueFrom:
                   configMapKeyRef:
                     name: app-config
                     key: MESSAGE
   ```
5) Service:
   ```
   apiVersion: v1
   kind: Service
   metadata:
     name: projectname-app
     namespace: projectname-dev
   spec:
     selector:
       app: projectname-app
     ports:
       - name: http
         port: 80
         targetPort: 8080
     type: ClusterIP
   ```
6) Route:
   ```
   apiVersion: route.openshift.io/v1
   kind: Route
   metadata:
     name: projectname-app
     namespace: projectname-dev
   spec:
     to:
       kind: Service
       name: projectname-app
     port:
       targetPort: http
     tls:
       termination: edge
   ```
7) Commit and push

Common gotchas:
- Namespace must match what PaasOperator created
- If image is private, add an imagePullSecret and reference it in the Deployment

---

### Part 4: ArgoCD application repo

#### Quick talk
- ArgoCD watches Git and syncs your cluster
- Application CR points to your deploy repo and namespace
- Automated sync and self-heal

#### Hands-on goals
- ArgoCD repo with an Application manifest that points to your deploy repo

#### Steps
1) Create repo: projectname-argocd
2) Add application.yaml:
   ```
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: projectname-app-dev
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: https://github.com/InHolland-Cloud-Minor-2526/projectname-deploy.git
       targetRevision: main
       path: manifests
     destination:
       server: https://kubernetes.default.svc
       namespace: projectname-dev
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
       syncOptions:
         - CreateNamespace=true
   ```
3) Apply it:
   - oc apply -f application.yaml -n argocd
   - Or add it via the ArgoCD UI
4) In ArgoCD, check the app turns Healthy/Synced
5) Visit the Route URL and see your app

Common gotchas:
- ArgoCD permissions to your target namespace
- Wrong repo URL or path
- Give it a minute to sync

---

### Part 5: Validate, fix issues, and stretch a bit

#### Quick checks
- oc get deploy,svc,route -n projectname-dev
- Hit the Route URL, see your app
- Change ConfigMap, commit, watch ArgoCD sync

#### Troubleshooting
- ImagePullBackOff (private image):
  - Create an imagePullSecret with GHCR credentials and reference it in the Deployment
- ArgoCD out of sync:
  - Check branch/path and manifest validity
- Route not reachable:
  - Check the router, TLS setting, and your network/VPN
- Permissions:
  - Verify role bindings from PaasOperator and ArgoCD access

#### Stretch goals
- Add kustomize overlays for dev/test/prod
- Use immutable image tags and an updater flow
- Add readiness/liveness probes
- Add resource requests/limits
- Add tests and basic security scans in CI
- Manage secrets properly (sealed-secrets or external secrets)

---

### Slide pointers for each part

- Part 0: What we’re building, the three repos, the pipeline diagram
- Part 1: PaaS.yaml anatomy and examples
- Part 2: Actions workflow layout and GHCR auth
- Part 3: What each manifest does and how they connect
- Part 4: ArgoCD Application fields and auto-sync
- Part 5: Common errors and fixes

---

### Repo summary
- projectname-app
  - App code
  - Dockerfile
  - .github/workflows/ci.yml
- projectname-deploy
  - manifests/
    - configmap.yaml
    - deployment.yaml
    - service.yaml
    - route.yaml
- projectname-argocd
  - application.yaml
- projectname-platform (optional)
  - PaaS.yaml

---

### Work with AI during the workshop
Ask AI to:
- Draft a PaaS.yaml for your team and capabilities
- Generate a minimal Java or .NET app + Dockerfile
- Write a GitHub Actions workflow with caching or multi-arch
- Create kustomize overlays
- Produce an ArgoCD Application YAML tailored to your repo names
- Troubleshoot specific error messages

Tip: Share exact repo names, namespaces, and error logs for better help.

---

### Done when
- PaaS.yaml applied and projects exist
- CI builds and pushes an image to GHCR
- Deploy repo has working manifests
- ArgoCD syncs and your app is reachable through a Route

If you tell me your project name and whether you want Java or .NET, I can generate ready-to-paste files for each repo.
