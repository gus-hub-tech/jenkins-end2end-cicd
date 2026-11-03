# Host deployment units

These systemd units run the demo stack on the Ubuntu EC2 host. They assume:

- User `ubuntu` owns the minikube/kubeconfig setup.
- `minikube` and `kubectl` are installed in `/usr/local/bin`.
- SonarQube is installed at `/opt/sonarqube` and runs as the `sonarqube` user.
- Argo CD is installed in the `argocd` namespace.
- The Spring Boot app is deployed by Argo CD in the `spring-boot-app` namespace.

## Install

```bash
sudo cp deploy/systemd/*.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now sonarqube.service minikube.service
sudo systemctl enable --now argocd-port-forward.service app-port-forward.service
```

If SonarQube was already started manually, stop it before starting the unit:

```bash
sudo -u sonarqube /opt/sonarqube/bin/linux-x86-64/sonar.sh stop
sudo systemctl start sonarqube.service
```

## Units

| Unit | Purpose | Port |
| --- | --- | --- |
| `sonarqube.service` | SonarQube server | `9000` |
| `minikube.service` | minikube cluster (docker driver, 8 GiB / 3 CPU) | - |
| `argocd-port-forward.service` | Argo CD UI | `8081` |
| `app-port-forward.service` | Spring Boot app | `8082` |

Jenkins runs as the distro-provided `jenkins.service` on port `8080`.

## Verify

```bash
systemctl status sonarqube minikube argocd-port-forward app-port-forward
curl -s http://127.0.0.1:9000/api/system/status
curl -k -s -o /dev/null -w '%{http_code}\n' https://127.0.0.1:8081/
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8082/
kubectl -n argocd get application spring-boot-app
```

## Notes

- Boot order is handled by `After=`/`Requires=`: Docker starts first, then minikube, then the port-forwards. SonarQube starts after the network is online.
- The port-forward units use `Restart=always`, so they recover if the Argo CD server or app pod is replaced.
- Adjust the user, paths, service namespace, and resources if your host differs.
- Open only the ports you need in the EC2 security group: `22`, `8080`, `8081`, `8082`, and `9000` (plus `80`/`443` if you terminate HTTP(S) separately).
