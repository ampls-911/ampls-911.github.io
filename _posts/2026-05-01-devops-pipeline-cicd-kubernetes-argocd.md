---
title: "Building an End-to-End DevOps Pipeline: CI/CD to Kubernetes with GitOps"
date: 2026-05-01 14:00:00 +0100
categories: [Projects, DevOps]
tags: [devops, cicd, github-actions, docker, kubernetes, argocd, gitops, prometheus, trivy, sonarcloud]
image:
  path: https://github.com/user-attachments/assets/df0255da-1141-4c9e-882c-aa208268d221

---

# Summary

For my DevOps course I built a small web app and wrapped it in a full pipeline, from a `git push` all the way to running pods in Kubernetes, with security gates and monitoring along the way. The app itself is deliberately boring: the point was the pipeline, not the product.

The stack:

- **Frontend** — a React landing page served by Nginx
- **Backend** — a Node.js / Express REST API that also exposes Prometheus metrics
- **Pipeline** — [GitHub Actions](https://github.com/features/actions) → [DockerHub](https://hub.docker.com/) → [Kubernetes](https://kubernetes.io/) (Minikube), reconciled by [ArgoCD](https://argo-cd.readthedocs.io/)
- **Security** — [Trivy](https://github.com/aquasecurity/trivy) image scans and `npm audit`, gated on CRITICAL
- **Quality** — [SonarCloud](https://www.sonarsource.com/products/sonarcloud/)
- **Monitoring** — [Prometheus](https://prometheus.io/) scraping `/metrics`, visualized in [Grafana](https://grafana.com/)

All of it is on [GitHub](https://github.com/ampls-911/devops-project).

# The app, briefly

The backend is a few Express routes: a welcome message, `/api/health` (status + uptime), `/api/info`, and `/metrics`. That last one is the interesting one. Using `prom-client`, the API registers a `http_requests_total` counter and increments it on every finished request, labelled by method, route and status code, on top of the default Node process metrics. That endpoint is what makes the whole thing observable later.

Four Jest tests cover the routes, including one asserting that `/metrics` actually returns `http_requests_total`. Small, but enough to give the pipeline something real to run and to fail on.

Both services ship as **multi-stage Docker images**. The backend installs production dependencies in a first stage and copies only `node_modules` and `src/` into a clean `node:20-alpine` final image, running as the non-root `node` user. The frontend builds the React app in a Node stage, then copies the static build into an `nginx:alpine` image. Smaller images, smaller attack surface.

# The pipeline

The whole thing lives in one GitHub Actions workflow that runs on every push to `main` or `dev`. It's five jobs with real dependencies between them, not a flat script:

1. **Backend CI** — install, ESLint, Jest with coverage, then upload the coverage report as an artifact.
2. **Frontend CI** — install, lint, test, build.
3. **SonarCloud** — downloads the backend coverage artifact and runs static analysis. Depends on backend CI.
4. **DevSecOps** — `npm audit --audit-level=critical` on both apps, then builds each image and runs a **Trivy scan with `exit-code: 1` on CRITICAL**. Depends on both CI jobs.
5. **Docker build & push** — only on `main`, and only after SonarCloud *and* security pass. Builds both images, tags them, and pushes to DockerHub.

The dependency graph matters: nothing gets built into an image until the tests, the quality scan and the security scan have all gone green. The security job is a real gate, not a report. A CRITICAL vulnerability in a base image or a dependency fails the build and nothing ships.

# The part I found most interesting: the GitOps loop

The naive way to deploy from CI is to have the pipeline run `kubectl apply` against the cluster. This project doesn't do that, and the reason is the whole point of GitOps.

Instead, the push job tags each image with the **7-character commit SHA**, then rewrites the image tag inside `k8s/backend.yaml` and `k8s/frontend.yaml` and commits that change back to the repo:

```
sed -i "s|ampls9/devops-backend:.*|ampls9/devops-backend:${TAG}|" k8s/backend.yaml
git commit -m "ci: update images to ${TAG}" && git push
```

Meanwhile **ArgoCD** watches the `k8s/` folder of the same repo. Its Application manifest has:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

So the moment the manifest changes, ArgoCD notices the drift between "what git says" and "what's running" and reconciles the cluster to match. `selfHeal` means if someone hand-edits the live cluster, ArgoCD reverts it back to the repo. `prune` means resources deleted from git get deleted from the cluster.

The shift is subtle but important: **the git repo is the source of truth, not the CI runner**. CI never touches the cluster. It only changes a file in git, and the cluster converges to git on its own. That's more auditable (every deploy is a commit), easier to roll back (revert the commit), and it means the cluster can't drift silently.

One neat detail that makes this safe: the manifest commit is pushed with the default `GITHUB_TOKEN`, and GitHub deliberately does *not* trigger new workflow runs from pushes made with that token. So the "commit back" step doesn't set off an infinite build loop.

# Kubernetes

The manifests run on Minikube in a dedicated `devops-project` namespace. A few things I made sure to include because they're what separates a `kubectl run` demo from something that behaves like a deployment:

- **Two backend replicas** behind a ClusterIP service.
- **Liveness and readiness probes** both hitting `/api/health`, so Kubernetes restarts dead pods and only routes traffic to ready ones.
- **Resource requests and limits** (64–128Mi memory, 100–250m CPU), so a pod can't starve the node.

# Monitoring

Prometheus scrapes the backend's `/metrics` endpoint every 15 seconds via the Kubernetes service DNS name, and Grafana reads from Prometheus to chart request rate, response times and pod uptime. Because the metrics are exposed by the app itself rather than inferred from outside, the monitoring reflects what the application actually did, not just whether the port was open.

# The DevOps lifecycle, mapped

The project was really an excuse to touch every phase of the DevOps loop with a real tool at each step:

| Phase | Tool |
|---|---|
| Plan | GitHub Projects |
| Code | Git + GitHub |
| Build | npm, Docker |
| Test | Jest, ESLint |
| Release | DockerHub |
| Deploy | Kubernetes + ArgoCD |
| Operate | Kubernetes |
| Monitor | Prometheus + Grafana |
| Improve | SonarCloud |

# What I'd change for anything real

It's a school project and runs on a local Minikube, so a few things are deliberately simplified:

- **Secrets.** The DockerHub login uses a `DOCKER_PASSWORD` secret. In anything real that should be a scoped DockerHub access token, not the account password, and cluster secrets would go through something like Sealed Secrets or an external secrets manager rather than plain Kubernetes Secrets.
- **One cluster, one environment.** There's no staging/prod split. A real GitOps setup would have separate ArgoCD applications or branches per environment.
- **Trivy scans images, not everything.** Adding SAST and a dependency-license check would round out the "shift-left" security story.
- **Minikube is single-node.** Real high availability needs a multi-node cluster, and the replica count would be driven by a HorizontalPodAutoscaler rather than hard-coded to two.

None of that takes away from what the project set out to show: a single `git push` that flows through quality checks, security gates, an image registry and a GitOps controller, and ends as running, monitored pods, with git as the one source of truth the whole way.

*DevOps module (Pratique DevOps, Chaînes d'outils et Automatisation), IT Business School, 2026.*
