# Workshop CI/CD for OpenShift

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

Setup a pipeline that builds and deploys the application to OpenShift.
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

now remove the deploymentthe route and the service , or switch to another namespace. Using the oc command and the yaml files we just saved we can create application again by using the oc command.
```
oc apply -f deployment.yaml
oc apply -f service.yaml
oc apply -f route.yaml
```
Check the application in the web console.

## ArgoCD

Install kustomize on your laptop

Github accesstokens
finegrained access
point naar argocd repo

nieuwe 
    resource owner
    only select repositories
    add permissions


