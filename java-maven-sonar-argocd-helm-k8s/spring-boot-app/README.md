# Spring Boot demo application

A small Spring Boot MVC application used by the CI/CD pipeline. It targets Java 11 and serves a static page on port `8080`.

## Build

From this directory:

```bash
mvn clean package
```

The artifact is written to `target/spring-boot-web.jar`. The demo contains no unit tests, so Maven runs the build and any tests added later.

## Run locally

```bash
java -jar target/spring-boot-web.jar
```

Open `http://localhost:8080`.

## Docker

```bash
docker build -t gustavdhub/ultimate-cicd:local .
docker run --rm -p 8080:8080 gustavdhub/ultimate-cicd:local
```

The Jenkins pipeline builds and pushes `gustavdhub/ultimate-cicd:${BUILD_NUMBER}` instead of the `:local` tag.

## Deployment

The application is deployed from the Helm chart in `../spring-boot-app-chart/` by Argo CD. The chart is the single source of truth for replicas, image tag, and the `NodePort` service.

```bash
kubectl apply -f "../Argo CD/application.yaml"
```

See the root `README.md` and `deploy/README.md` for the full environment setup.

## SonarQube server

Install Java 21 and the SonarQube distribution under `/opt/sonarqube`:

```bash
sudo apt update
sudo apt install -y openjdk-21-jdk unzip
cd /opt
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-26.1.0.118079.zip
sudo unzip sonarqube-26.1.0.118079.zip
sudo mv sonarqube-26.1.0.118079 sonarqube

sudo adduser --system --no-create-home --group --disabled-login sonarqube
sudo chown -R sonarqube:sonarqube /opt/sonarqube

echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-sonarqube.conf
sudo sysctl --system
```

Run it with the systemd unit from this repository:

```bash
sudo cp ../../deploy/systemd/sonarqube.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now sonarqube.service
```

The manual equivalent is:

```bash
sudo -u sonarqube /opt/sonarqube/bin/linux-x86-64/sonar.sh start
```

The server listens on `http://<host>:9000`. The Jenkins pipeline authenticates with the `sonarqube` credential and the `SONAR_URL` parameter.

## Notes

- The rendered page is static (`StartApplication.java` and `index.html`); a pipeline build changes the image tag, not the page text.
- The Dockerfile uses a Java 11 base image; update it if you change the application's Java version.
