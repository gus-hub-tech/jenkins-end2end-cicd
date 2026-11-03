# Jenkins End-to-End CI/CD

A working Jenkins pipeline for a Spring Boot Java application: Maven build, SonarQube analysis, Docker image push, Helm chart update, and Argo CD deployment to minikube.

This repository is a fork of [iam-veeramalla/Jenkins-Zero-To-Hero](https://github.com/iam-veeramalla/Jenkins-Zero-To-Hero). The active pipeline lives under `java-maven-sonar-argocd-helm-k8s/`.

## Architecture

```text
GitHub main
    |
    v
Jenkins (Docker agent)
    |-- mvn clean package
    |-- SonarQube analysis
    |-- docker build/push gustavdhub/ultimate-cicd:<BUILD_NUMBER>
    |-- commit updated Helm chart tag to main
    v
GitHub main (chart tag)
    |
    v
Argo CD (auto-sync) --> minikube --> Deployment/Service (spring-boot-app)
```

## Current demo deployment

| Component | URL | Notes |
| --- | --- | --- |
| Jenkins | http://15.240.98.85:8080 | Login required |
| Argo CD | https://15.240.98.85:8081 | Self-signed certificate |
| Spring Boot app | http://15.240.98.85:8082 | Port-forward to the service |
| SonarQube | http://15.240.98.85:9000 | Analysis server |

The public address is an Elastic IP. Substitute your own host/IP if you rebuild the environment. The app is also reachable on the host at `http://192.168.49.2:32208` through the minikube NodePort.

## Repository layout

Active pipeline:

- `java-maven-sonar-argocd-helm-k8s/spring-boot-app/` — Spring Boot app and `JenkinsFile`
- `java-maven-sonar-argocd-helm-k8s/spring-boot-app-chart/` — Helm chart (single source of truth for the deployment)
- `java-maven-sonar-argocd-helm-k8s/Argo CD/application.yaml` — Argo CD `Application`
- `deploy/systemd/` — systemd units for SonarQube, minikube, and the port-forwards
- `deploy/README.md` — host unit installation and verification

Examples from the upstream repository (not part of this pipeline):

- `my-first-pipeline/`, `multi-stage-multi-agent/`, `shared-libraries/`, `vars/`, `python-jenkins-argocd-k8s/`, `Interview_Questions.md`

## Pipeline stages

The Jenkins job is a **Pipeline script from SCM**:

- SCM: `https://github.com/gus-hub-tech/jenkins-end2end-cicd.git`
- Branch: `main`
- Script path: `java-maven-sonar-argocd-helm-k8s/spring-boot-app/JenkinsFile`

Stages:

1. **Build and Test** — `mvn clean package` in `spring-boot-app`.
2. **Static Code Analysis** — SonarQube Maven scanner against `SONAR_URL`.
3. **Build and Push Docker Image** — builds and pushes `gustavdhub/ultimate-cicd:${BUILD_NUMBER}`.
4. **Update Helm Chart** — updates `values.yaml` (`tag`) and `Chart.yaml` (`appVersion`), commits, and pushes to `main`.

Checkout is implicit: Jenkins checks out the configured branch before running the pipeline.

## Jenkins parameters

| Parameter | Default | Purpose |
| --- | --- | --- |
| `SONAR_URL` | `http://10.0.1.163:9000` | SonarQube URL reachable from the Docker agent |
| `DOCKER_UID` | `104` | UID for the Docker agent (host `jenkins` UID) |
| `DOCKER_GID` | `109` | GID for the Docker agent (host `jenkins` GID) |
| `DOCKER_GROUP_GID` | `110` | GID of the host `docker` group |

Use `id jenkins` and `getent group docker` to confirm the values on your host. The defaults match the demo host.

## Jenkins credentials

Create these in **Manage Jenkins > Credentials > System > Global credentials**:

| ID | Type | Used for |
| --- | --- | --- |
| `sonarqube` | Secret text | SonarQube user token (`SONAR_AUTH_TOKEN`) |
| `docker-cred` | Username with password | Docker Hub push |
| `github` | Secret text | GitHub PAT with `repo` scope (`GITHUB_TOKEN`) |

Do not commit tokens, passwords, or kubeconfigs.

## Prerequisites

- An Ubuntu EC2 instance with Docker and Jenkins.
- A SonarQube server (Java 17+; the demo uses Java 21) with `vm.max_map_count=262144`.
- minikube with the Docker driver (8 GiB / 3 CPU is enough for this demo).
- Argo CD installed in the `argocd` namespace from the official install manifests.
- Helm is not required on the host; Argo CD renders the chart.
- Security group ports: `22`, `8080`, `8081`, `8082`, `9000` (and `80`/`443` if needed).

## SonarQube

Install SonarQube under `/opt/sonarqube` and run it as the `sonarqube` user. The repository includes the systemd unit:

```bash
sudo cp deploy/systemd/sonarqube.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now sonarqube.service
```

Set the kernel parameter required by SonarQube's Elasticsearch:

```bash
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-sonarqube.conf
sudo sysctl --system
```

See `java-maven-sonar-argocd-helm-k8s/spring-boot-app/README.md` for the manual install steps.

## minikube and port-forwards

```bash
minikube start --driver=docker --memory=8192 --cpus=3
sudo cp deploy/systemd/minikube.service deploy/systemd/argocd-port-forward.service deploy/systemd/app-port-forward.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now minikube.service argocd-port-forward.service app-port-forward.service
```

The port-forwards expose:

- `8081` → `svc/argocd-server:443` in `argocd`
- `8082` → `svc/spring-boot-app-service:80` in `spring-boot-app`

## Argo CD

Apply the `Application` after Argo CD is installed:

```bash
kubectl apply -f "java-maven-sonar-argocd-helm-k8s/Argo CD/application.yaml"
```

The app uses automated sync with prune and self-heal, and creates the `spring-boot-app` namespace if needed.

Retrieve the initial Argo CD admin password with:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

## Operations

```bash
systemctl status docker jenkins sonarqube minikube argocd-port-forward app-port-forward
kubectl -n argocd get pods
kubectl -n argocd get application spring-boot-app
kubectl -n spring-boot-app get pods
```

After a host reboot the units start in this order: Docker → minikube → port-forwards; SonarQube starts after the network. Jenkins starts independently. The port-forward units use `Restart=always`.

## Notes and limitations

- The demo app's page content is static; a build changes the image tag (`:25`, `:26`, ...), not the rendered text. Edit `StartApplication.java` or `index.html` if you want a visible change.
- There are no unit tests in the app; `mvn clean package` builds the artifact and runs any tests that are added.
- The Helm chart is the single source of truth. The legacy raw manifests were removed so Jenkins and Argo CD cannot drift.
- The `SONAR_URL` default points at the demo host's private IP. Change the parameter when moving to another host.
- The deployment is intended for a demo; use a managed Kubernetes cluster, TLS, and restricted security groups for production.

## License and credits

See `LICENSE`. Based on [iam-veeramalla/Jenkins-Zero-To-Hero](https://github.com/iam-veeramalla/Jenkins-Zero-To-Hero).
