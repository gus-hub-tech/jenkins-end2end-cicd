# Spring Boot based Java web application
 
This is a simple Sprint Boot based Java application that can be built using Maven. Sprint Boot dependencies are handled using the pom.xml 
at the root directory of the repository.

This is a MVC architecture based application where controller returns a page with title and message attributes to the view.

## Execute the application on remote server/instance and access it using your browser

Checkout the repo and move to the directory

```
git clone https://github.com/iam-veeramalla/Jenkins-Zero-To-Hero/java-maven-sonar-argocd-helm-k8s/sprint-boot-app
cd java-maven-sonar-argocd-helm-k8s/sprint-boot-app
```

Execute the Maven targets to generate the artifacts

```
mvn clean package
```

The above maven target stroes the artifacts to the `target` directory. You can either execute the artifact on your local machine
(or) run it as a Docker container.

** Note: To avoid issues with local setup, Java versions and other dependencies, I would recommend the docker way. **


### Execute Remote (Java 11 needed) and access the application on http://<ip-address>:9000


### The Docker way

Build the Docker Image

```
docker build -t ultimate-cicd-pipeline:v1 .
```

```
docker run -d -p 8010:8080 -t ultimate-cicd-pipeline:v1
```

Hurray !! Access the application on `http://<ip-address>:8010`


## Next Steps

### Configure a Sonar Server locally

```
System Requirements
Java 17+ (Oracle JDK, OpenJDK, or AdoptOpenJDK)
Hardware Recommendations:
   Minimum 2 GB RAM
   2 CPU cores
sudo apt update && sudo apt install unzip -y
sudo apt install -y openjdk-21-jdk unzip
cd /opt
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-26.1.0.118079.zip
sudo unzip sonarqube-26.1.0.118079.zip
sudo mv sonarqube-26.1.0.118079 sonarqube


sudo adduser --system --no-create-home --group --disabled-login sonarqube
sudo chown -R sonarqube:sonarqube /opt/sonarqube

sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee /etc/sysctl.d/99-sonarqube.conf

sudo -u sonarqube /opt/sonarqube/bin/linux-x86-64/sonar.sh start

```

Hurray !! Now you can access the `SonarQube Server` on `http://<ip-address>:9000` 


