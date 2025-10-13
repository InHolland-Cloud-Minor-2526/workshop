# Workshop CI/CD for OpenShift


by: wim-jan.hilgenbos@hcs-company.com

## Important links and conventions
- GitHub InHolland-Cloud-Monor-2526 organisation: https://github.com/InHolland-Cloud-Minor-2526
- OpenShift cluster: https://inholland-minor.openshift.eu/
- Bhr-paas project: https://github.com/InHolland-Cloud-Minor-2526/bhr-paas

## Check prerequistes
- are you in the GitHub organization?
  - did you accept the invitation?
  - if not, did you get an email? Check with the instructor.
- join your team in the GitHub organization
- Have docker or podman installed on your laptop
- Have an IDE
  - IntelliJ
  - VSCode
  - Eclipse
  - ...
- Have a terminal
  - Windows: PowerShell
  - Linux: Bash
  - MacOS: Terminal
- Have a browser
  - Chrome
  - Firefox


---
> Short tour of the GitHub organization
---

## GitHub 

checkout the GitHub organization: https://github.com/InHolland-Cloud-Minor-2526

### setup of your own environment
You will be sharing your work with your teammates on GitHub. You can do this by having individual repos or start a team organisation like we did voor this workshop.
- Make sure to invite the trainers/teachers to your team organisation (read-only) by the time you will present your work.
- there are some useful tools within GitHub, like  
  - "Projects" 
  - wikis
  - ...

---
> GitOps
---

---
> Overview of OpenShift
---

## Check your OpenShift cluster
- login to the OpenShift cluster (https://inholland-minor.openshift.eu/)
- browse the OpenShift web console
- check your cluster version
- Check the number of nodes in the cluster
- Check the CPU and memory per node
- Check the storage per node
- install the OpenShift CLI
  - Under the "?" menu, click on "Command Line Tools" and install the "oc" tool
  - Check the version of the oc tool
- login to the OpenShift cluster from the command line
  - go to your name in the top right corner
  - click on "Copy Login Command"
  - paste the command in a terminal
  - login to the cluster
  - check the version of the oc tool
- check the status of the cluster
  - oc get nodes
  - oc get pods -A
  - oc get projects
  - oc get quota
  - oc get storageclass
  - oc get pvc -A
  - oc get pv -A   

---
> Short tour of the application we are going to deploy
---

## Application
- FotoShow https://github.com/InHolland-Cloud-Minor-2526/fotoshow
- create a fork of the FotoShow repository to your GitHub account
- clone the repository to your laptop
- Have a look at the README file
  - build the application locally
  - run the application locally
    - Note that that the port is 81818 not 8080
  - Play around with the environment variables
  - Check out the health endpoints
  - Create a Docker image locally
    - run it with docker run or podman run

---
> Introduction to the PaaS operator
---

## Create a new project in OpenShift with the PaaS operator
- fork the bhr-paas repo and clone it to your laptop (https://github.com/InHolland-Cloud-Minor-2526/bhr-paas)
- copy an existing PaaS.yaml file 
- edit the PaaS.yaml file to change/create a group and added your team members
- define a couple of namespaces/projects in OpenShift with the PaaS operator
- define your basic quota
  - CPU request: 3 core
  - Mem Request: 5 Gi
  - Mem Limit: 10 Gi
  - Storage: 20 Gi

Your PaaS should look something like this:
```
apiVersion: cpet.belastingdienst.nl/v1alpha2
kind: Paas
metadata:
  name: sam-sam
spec:
  requestor: group-t
  groups:
    admin:
      query: CN=Groep 7 - Enbition
      roles:
        - admin
    group-t-members:
      users:
        - mirkarlar
      roles:
        - view
  namespaces:
    showcase: {}
    test: {}
  quota:
    limits.memory: 4Gi
    requests.cpu: '1'
    requests.memory: 2Gi
    requests.storage: 2Gi
```
- Create a pull request to the bhr-paas repo
- After the pull request is merged, the PaaS operator will create the namespaces and the quota for you.
- login to the OpenShift web console and check the namespaces and quota. (Home->Projects)
- login to the OpenShift CLI and check the namespaces and quota. (oc get projects, oc get quota)

---
> Setup CI/CD
---

## Create a new pipeline in GitHub
Pipelines in github are called "Actions" or workflows.
An action is a set of jobs that are executed when a certain event occurs.
Each job is a set of steps that are executed in sequence.

Our pipeline only does CI, so it will only create and store and image in the registry. Deployments are handled by ArgoCD.

Setup a pipeline that builds an application and its container image and pushes the image to the github registry.
- You can choose to use a separate repo for you for you pipeline or use the same repo as you used for the application.
  - a separate repo is more purist because the lifecycle of the pipeline is separated from the lifecycle of the application.
  - a separate repo can also be more practical because you can use the same pipeline for multiple applications. 
- Within the pipeline distinguish between the build of the application, the build of the Docker image and the push to the registry.

If you use a separate repo for your pipeline, remember to create a new GitHub secret for the pipeline to access the application repo. 
And possibly a new GitHub secret for the pipeline to access the registry.

I prefer separate jobs in the pipeline. But since every job is executed in a new virtual machine, you must consider passing data between jobs.

Schematicly the pipeline should look like this:
```
name: Build and Push fotoshow Image

on:
  workflow_dispatch:

permissions:
  contents: read
  packages: write

env:
  IMAGE_NAME: ghcr.io/${{ github.repository_owner }}/fotoshow

jobs:
  build:
    name: Build (compile and test)
    runs-on: ubuntu-latest
    steps:
      - name: Checkout pipeline repo
        uses: actions/checkout@v4
      - name: Checkout mirkarlar/fotoshow source
        uses: actions/checkout@v4
        with:
          repository: mirkarlar/fotoshow
          path: fotoshow
          token: ${{ secrets.CHECKOUT_PAT }}
          fetch-depth: 1

<< build steps >>
      - name: Upload fotoshow workspace
        uses: actions/upload-artifact@v4
        with:
          name: fotoshow-workspace
          path: fotoshow
          
  create-image:
    name: Create image (build only)
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Download fotoshow workspace
        uses: actions/download-artifact@v4
        with:
          name: fotoshow-workspace
          path: fotoshow
          
          
    << create image steps >>
    
    
      - name: Upload built image
        uses: actions/upload-artifact@v4
        with:
          name: built-image
          path: /tmp/image.tar
          
  push:
    name: Push image to GHCR
    runs-on: ubuntu-latest
    needs: create-image
    steps:
      << push steps >>          
          
```
The jobs can be extended by statically checking the code, running tests, scanning for fulnerabilities,  etc.


### Triggers
For now we wont focus on the triggers. But we want to run our pipeline when a pull request is created or when a commit is pushed to the main branch. 

In the end the pipeline should be triggered by 
a pull request is created or when a commit is pushed to the main branch.
- triggered by a tag.
- by a release.
You can find more information about triggers here: https://docs.github.com/en/actions/reference/events-that-trigger-workflows


## Manual deployment

First we deploy the application manually to OpenShift.
- Login to the OpenShift web console

We use one of the namespaces that we created in the PaaS operator.

If we had a private registry, we would have to create a new secret with the credentials.
- Create a new secret with the registry credentials
- Create a new deployment config
- Create a new service
- Create a new route.

We can now save the artifacts to our deployment repo

When Openshift creates a new deployments and such the yaml will containt many fields we dont need. They will also contain lifecycle and status data.

Strip your yaml file of these fields.

The results should look like:

```
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: fotoshow
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fotoshow
  template:
    metadata:
      labels:
        app: fotoshow
    spec:
      containers:
        - name: fotoshow
          image: 'image-registry.openshift-image-registry.svc:5000/sam-sam-showcase/fotoshow@sha256:3353cb84e8a956a5c53f88da2e45153d465b4b7115ff28e80a4c4c4053305c89'
          ports:
            - containerPort: 18181
              protocol: TCP
          env:
            - name: SLIDESHOW_EXTERNAL_DIR
              value: /data
          volumeMounts:
            - name: foto-data-volume
              mountPath: /data
      volumes:
        - name: foto-data-volume
          persistentVolumeClaim:
            claimName: pvc-foto-show
---

kind: Service
apiVersion: v1
metadata:
  name: fotoshow
  namespace: sam-sam-showcase
  labels:
    app: fotoshow
    app.kubernetes.io/component: fotoshow
    app.kubernetes.io/instance: fotoshow
    app.kubernetes.io/name: fotoshow
spec:
  ports:
    - name: 18181-tcp
      protocol: TCP
      port: 18181
      targetPort: 18181
  type: ClusterIP
  selector:
    app: fotoshow

---

kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: fotoshow
  namespace: sam-sam-showcase
  labels:
    app: fotoshow
    app.kubernetes.io/component: fotoshow
    app.kubernetes.io/instance: fotoshow
    app.kubernetes.io/name: fotoshow
  annotations:
spec:
  host: fotoshow-sam-sam-showcase.apps.inholland-minor.openshift.eu
  to:
    kind: Service
    name: fotoshow
    weight: 100
  port:
    targetPort: 18181-tcp
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
  wildcardPolicy: None



```
 

now remove the deploymentthe route and the service , or switch to another namespace. Using the oc command and the yaml files we just saved we can create application again by using the oc command.
```
oc apply -f deployment.yaml
oc apply -f service.yaml
oc apply -f route.yaml
```
Check the application in the web console.

### organizing your deployment repo
In your deployment repo you can create a folder for each environment. To avoid duplication ArgoCD uses kustomize to handle the differences between environments.

You will setup a base folder with the the common yaml files. 
You will then create overlays for each environment. In theses overlays you can add or remove or mutate yaml files to match the environment.

In our example we will have a base folder with the deployment.yaml, service.yaml and route.yaml files.
then create overlays for the test environments.

In the test case we will only deploy the application to the test namespace.

Each directory you point ArgoCD to will need a kustomization.yaml file. For our regular deployment we will point ArgoCD to the base folder.

Then we are going to create another ArgoCD application for the test environment. We will point ArgoCD to the overlay/test folder. In that folder we will add a kustomization.yaml file that points to the base folder. 
There will also be a rule in the kustomization.yaml file that will apply the deployment to the test namespace.

### kustomize

Kustomize tool is a tool to manipulate yaml files. The most basic usage is to list files and directories to be included in the output.
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment-fotoshow.yaml
  - route-fotoshow.yaml
  - service-fotoshow.yaml
  - pvc-fotoshow.yaml

```

But with kustomize you can also manipulate the yaml files. In our overlays/test folder we include the following kustomization.yaml

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization


namespace: sam-sam-test

resources:
  - ../../base

```

which includes the base folder and sets the namespace to sam-sam-test.


Kustomize can also be used to mutate the yaml files. In our overlays/test folder we could add the following to kustomization.yaml

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

patches:
  - target:
      kind: Deployment
      name: my-app
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
```

This will replace the replicas in the deployment.yaml file with 5.

## ArgoCD

Install kustomize on your laptop

In ArgoCD you will need to configure the connection to the ArgoCD repo and the deployment (or applicaton) repo.
For this you will need Github accesstokens (see below)


Add the ArgoCD capabilities to your project in the PaaS.yaml (make sure the indentation is correct)



```
capabilities:
    argocd:
      custom_fields:
        git_url: 'https://github.com/Myorg/myown-argocd.git'
```



In ArgoCD go to Settings -> Applications -> Connections
add your argocd repo and/or your deployment repo depending on the visibility of the repos. (public/private)

Create a new (ArgoCD) application 

```

---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: simple-slideshow-test
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: sam-sam-test
  project: default
  source:
    path: simpleSlideShow/overlays/test/
    repoURL: https://github.com/InHolland-Cloud-Minor-2526/simpleSlideshow-deploy.git
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

```

Make sure to replace the repoURL with the url of your deployment repo. Also the path with the path to the overlay/test folder.



Later on we can add more applications. For instance the test application we prepared for in the deployment repo.

In this case the repoUrl will be the same but the path will point to the overlay/test folder.



---
## Creating github accesstokens
In your github account go to settings -> developer settings -> personal access tokens
Choose:
   finegrained access
   If in an organisation select resource owner --> organisation
   choose for : only select repositories
   point to the repo you want to connect to
   Click the add permissions button
     select Contents: read
or
     select Packages: read 
  Click Create token

Remember to save the token somewhere save(!) It will not be shown again!
   


