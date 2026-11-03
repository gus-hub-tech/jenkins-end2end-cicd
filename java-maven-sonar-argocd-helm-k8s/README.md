# Java pipeline: Maven, SonarQube, Docker, Helm, and Argo CD

This directory contains the active end-to-end pipeline for the Spring Boot demo application.

## Contents

- `spring-boot-app/` — Spring Boot app, `JenkinsFile`, and Dockerfile.
- `spring-boot-app-chart/` — Helm chart rendered by Argo CD.
- `Argo CD/application.yaml` — Argo CD `Application` for the chart.

The Helm chart is the single source of truth for the Kubernetes deployment. Argo CD watches `main`, renders the chart, and syncs the `spring-boot-app` namespace. Jenkins does not deploy to Kubernetes directly and does not call the Argo CD API.

## Pipeline stages

The Jenkins job uses `spring-boot-app/JenkinsFile` as a **Pipeline script from SCM**:

1. **Build and Test** — `mvn clean package` in `spring-boot-app`.
2. **Static Code Analysis** — SonarQube Maven scanner against the `SONAR_URL` parameter.
3. **Build and Push Docker Image** — builds and pushes `gustavdhub/ultimate-cicd:${BUILD_NUMBER}`.
4. **Update Helm Chart** — updates the chart image tag and `appVersion`, commits, and pushes to `main`.

The SCM checkout is implicit. There is no separate unit-test, Helm-deploy, Selenium, or Argo-API stage in this pipeline.

## Jenkins parameters and credentials

Parameters:

- `SONAR_URL` — SonarQube base URL reachable from the Docker agent.
- `DOCKER_UID`, `DOCKER_GID`, `DOCKER_GROUP_GID` — IDs used by the Docker agents.

Credentials:

- `sonarqube` — Secret text containing a SonarQube token.
- `docker-cred` — Docker Hub username/password.
- `github` — Secret text containing a GitHub PAT with `repo` scope.

See the root `README.md` for the full job configuration and bootstrap steps.

## Deployment flow

1. Jenkins pushes a new image tag to `main`.
2. Argo CD detects the chart change.
3. Argo CD renders `spring-boot-app-chart/` and applies it to the `spring-boot-app` namespace.
4. The `Service` exposes port `80` (NodePort `32208`); the host port-forward exposes it on `8082`.

Apply the `Application` after Argo CD is installed:

```bash
kubectl apply -f "java-maven-sonar-argocd-helm-k8s/Argo CD/application.yaml"
```

## Related documentation

- Root `README.md` — architecture, host bootstrap, credentials, and operations.
- `deploy/README.md` — systemd units for SonarQube, minikube, and the port-forwards.
- `spring-boot-app/README.md` — build and run the application.

## Notes

- The app currently has no unit tests; `mvn clean package` builds the artifact and runs any tests that are added.
- The rendered page is static, so a build changes the deployed image tag rather than the page text.
- The `SONAR_URL` default points at the demo host's private IP; change it for other environments.
