
# **Lab 17: Deploy and Orchestrate Microservices on Kubernetes with ConfigMaps (Spring Boot 3.4.5)**

This single lab walks you from a blank folder to two fully orchestrated microservices (**user‑service** and **order‑service**) running on Minikube.

---

## 1  Tooling Prerequisites

| Tool | Min Version | Verify |
|------|-------------|--------|
| Java (Temurin / OpenJDK) | 17 | `java -version` |
| Maven | 3.9 | `mvn -version` |
| Docker Desktop | 4.x | `docker --version` |
| Minikube | 1.33 + | `minikube version` |
| kubectl | 1.30 + | `kubectl version --client` |

---

## 2  Generate Projects (Spring Initializr)

Open **PowerShell as Administrator**, `cd` to the folder you want to use as **lab17**, then run:

```powershell
# user‑service
curl.exe -d dependencies=web `
         -d bootVersion=3.4.5 `
         -d javaVersion=17 `
         -d type=maven-project `
         -d groupId=com.example `
         -d artifactId=user-service `
         -o user-service.zip `
         https://start.spring.io/starter.zip; `
Expand-Archive -LiteralPath user-service.zip -DestinationPath user-service; `
Remove-Item user-service.zip

# order‑service
curl.exe -d dependencies=web `
         -d bootVersion=3.4.5 `
         -d javaVersion=17 `
         -d type=maven-project `
         -d groupId=com.example `
         -d artifactId=order-service `
         -o order-service.zip `
         https://start.spring.io/starter.zip; `
Expand-Archive -LiteralPath order-service.zip -DestinationPath order-service; `
Remove-Item order-service.zip
```

Resulting layout:

```
lab17/
├── user-service/
└── order-service/
```

---

## 3  Add Controllers & Config

> **Tip:** open both projects in **IntelliJ IDEA** and let it auto‑import Maven.

### user‑service  
Location: `src/main/java/com/microservices/user_service/UserController.java`

```java
package com.microservices.user_service;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserController {

    @Value("${app.name}")
    private String appName;

    @GetMapping("/users")
    public String users() {
        return "Users from " + appName;
    }
}
```

`src/main/resources/application.properties`

```properties
server.port=${SERVER_PORT:8081}
app.name=${APP_NAME:user-service}
```

### order‑service  
Location: `src/main/java/com/microservices/order_service/OrderController.java`

```java
package com.microservices.order_service;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestClient;

@RestController
public class OrderController {

    private final RestClient rest = RestClient.create();

    @Value("${app.name}")         private String appName;
    @Value("${user.service.url}") private String userServiceUrl;

    @GetMapping("/orders")
    public String orders() {
        String users = rest.get()
                           .uri(userServiceUrl)
                           .retrieve()
                           .body(String.class);
        return "Orders from " + appName + " / " + users;
    }
}
```

`src/main/resources/application.properties`

```properties
server.port=${SERVER_PORT:8082}
app.name=${APP_NAME:order-service}
user.service.url=${USER_SERVICE_URL:http://user-service:8081/users}
```

---

## 4  Build JARs

In **each** service folder (or via IntelliJ terminal):

```bash
mvn clean package
```

---

## 5  Dockerize

Create a **Dockerfile** in each service root.

<details><summary>*user-service/Dockerfile*</summary>

```dockerfile
FROM eclipse-temurin:17-jre
ARG JAR_FILE=target/user-service-0.0.1-SNAPSHOT.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```
</details>

<details><summary>*order-service/Dockerfile*</summary>

```dockerfile
FROM eclipse-temurin:17-jre
ARG JAR_FILE=target/order-service-0.0.1-SNAPSHOT.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```
</details>

_No `.dockerignore` is needed now._

### Build images (Administrator PowerShell)

```powershell
cd lab17        # ensure you’re in the lab17 root

docker build -t user-service:1.0 ./user-service
docker build -t order-service:1.0 ./order-service
```

### Optional local test

```powershell
docker run -d --name user -p 8081:8081 user-service:1.0
docker run -d --name order -p 8082:8082 -e USER_SERVICE_URL=http://host.docker.internal:8081/users order-service:1.0
curl http://localhost:8082/orders
docker rm -f user order
```

---

## 6  Load Images into Minikube

```powershell
minikube start --driver=docker
minikube image load user-service:1.0
minikube image load order-service:1.0
```

---

## 7  Kubernetes Manifests (`k8s/`)

```
lab17/
└─ k8s/
   ├─ user-service-configmap.yaml
   ├─ order-service-configmap.yaml
   ├─ user-service-deployment.yaml
   ├─ user-service-service.yaml
   ├─ order-service-deployment.yaml
   └─ order-service-service.yaml
```

### ConfigMaps

```yaml
# k8s/user-service-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-service-config
data:
  SERVER_PORT: "8081"
  APP_NAME: "user-service"
```

```yaml
# k8s/order-service-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
data:
  SERVER_PORT: "8082"
  APP_NAME: "order-service"
  USER_SERVICE_URL: "http://user-service:8081/users"
```

### Deployments and Services

```yaml
# k8s/user-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: user-service
          image: user-service:1.0
          ports:
            - containerPort: 8081
          envFrom:
            - configMapRef:
                name: user-service-config
```

```yaml
# k8s/user-service-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
    - protocol: TCP
      port: 8081
      targetPort: 8081
  type: ClusterIP
```

```yaml
# k8s/order-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: order-service:1.0
          ports:
            - containerPort: 8082
          envFrom:
            - configMapRef:
                name: order-service-config
```

```yaml
# k8s/order-service-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
    - protocol: TCP
      port: 8082
      targetPort: 8082
  type: ClusterIP
```

---

## 8  Apply → Test → Scale → Clean Up

```powershell
kubectl apply -f k8s
kubectl get pods
kubectl port-forward service/order-service 8082:8082
curl http://localhost:8082/orders    # ➜ Orders from order-service / Users from user-service
kubectl scale deployment order-service --replicas=3
kubectl delete -f k8s
```

---

🎉 **Done!**  
You’ve built, containerized, and orchestrated two Spring Boot microservices from scratch—complete with externalized configuration and a clean folder structure.
