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
- GitHub org ready: [InHolland-Cloud-Minor-2526](https://github.com/InHolland-Cloud-Minor-2526)
- OpenShift cluster ready and PaasOperator installed: [opr-paas](https://github.com/belastingdienst/opr-paas)
- Students have OpenShift access and permissions
- ArgoCD installed and reachable
- Optional templates (nice to have):
  - App template (simple Java/.NET + Dockerfile)
  - projectname-deploy template (manifests + optional kustomize)
  - projectname-argocd template (Application YAML)
- Student laptops: Git, browser, optional Docker/Podman, Java/.NET SDK if building locally

---

### Schedule at a glance
- Part 0: Intro + architecture (20–30 min)
- Part 1: PaasOperator + PaaS.yaml (45–60 min)
- Part 2: Continuous Integration with GitHub Actions + GHCR (60 min)
- Part 3: Deployment repo with manifests (45–60 min)
- Part 4: ArgoCD for CD (45–60 min)
- Part 5: Validate + troubleshoot + stretch goals (30–45 min)
- Wrap-up (10–15 min)

Each part starts with a short mini-talk, then hands-on
