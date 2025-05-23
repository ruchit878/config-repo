
# **Lab 16 + 17: End‑to‑End Workflow — Spring Boot → Docker → Minikube → Kubernetes (v3.4.5)**

## Overview
You’ll take two microservices (**user‑service**, **order‑service**) from source code all the way to a running Kubernetes cluster on **Minikube**, using:

* Spring Boot 3.4.5
* Docker images with wildcard‑friendly Dockerfiles
* ConfigMaps for runtime configuration
* Deployments & Services for orchestration

---

### Table of Contents
1. Tooling Prereqs  
2. Generate Projects (Spring Initializr)  
3. Add Controllers & Placeholder Config  
4. Build JARs  
5. Dockerize & (optionally) Test Locally  
6. Load Images into Minikube  
7. Kubernetes Manifests (ConfigMaps, Deployments, Services)  
8. Apply, Test, Scale, Clean Up  
9. Optional Extras  

---

## 1  Tooling Prereqs

| Tool | Min Version | Verify |
|------|-------------|--------|
| Java (Temurin/OpenJDK) | 17 | `java -version` |
| Maven | 3.9 | `mvn -version` |
| Docker Desktop | 4.x | `docker --version` |
| Minikube | 1.33 | `minikube version` |
| kubectl | 1.30 | `kubectl version --client` |

---

## 2  Generate Projects

<details><summary>curl commands</summary>

```bash
# user-service
curl https://start.spring.io/starter.zip ^
 -d dependencies=web ^
 -d bootVersion=3.4.5 ^
 -d javaVersion=17 ^
 -d type=maven-project ^
 -d groupId=com.example ^
 -d artifactId=user-service ^
 -o user-service.zip && unzip user-service.zip -d user-service && del user-service.zip

# order-service
curl https://start.spring.io/starter.zip ^
 -d dependencies=web ^
 -d bootVersion=3.4.5 ^
 -d javaVersion=17 ^
 -d type=maven-project ^
 -d groupId=com.example ^
 -d artifactId=order-service ^
 -o order-service.zip && unzip order-service.zip -d order-service && del order-service.zip
```
</details>

Folder layout:

```
workspace/
├─ user-service/
└─ order-service/
```

---

## 3  Add Controllers & Config

### user‑service

*`src/main/java/com/example/user/controller/UserController.java`*

```java
package com.example.user.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.*;

@RestController
public class UserController {
    @Value("${app.name}") String appName;

    @GetMapping("/users")
    public String users() {
        return "Users from " + appName;
    }
}
```

*`src/main/resources/application.properties`*

```properties
server.port=${SERVER_PORT:8081}
app.name=${APP_NAME:user-service}
```

### order‑service

*`src/main/java/com/example/order/controller/OrderController.java`*

```java
package com.example.order.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.client.RestClient;

@RestController
public class OrderController {
    private final RestClient rest = RestClient.create();

    @Value("${app.name}")          String appName;
    @Value("${user.service.url}")  String userServiceUrl;

    @GetMapping("/orders")
    public String orders() {
        String users = rest.get().uri(userServiceUrl)
                           .retrieve().body(String.class);
        return "Orders from " + appName + " / " + users;
    }
}
```

*`application.properties`*

```properties
server.port=${SERVER_PORT:8082}
app.name=${APP_NAME:order-service}
user.service.url=${USER_SERVICE_URL:http://user-service:8081/users}
```

---

## 4  Build JARs

```bash
cd user-service   && mvn clean package
cd ../order-service && mvn clean package
```

---

## 5  Dockerize

*Dockerfile* (place next to each `pom.xml`):

```dockerfile
FROM eclipse-temurin:17-jre
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

`.dockerignore`:

```
target/
.idea/
*.iml
```

Build images:

```bash
cd workspace
docker build -t user-service:1.0 ./user-service
docker build -t order-service:1.0 ./order-service
```

Optional local test:

```bash
docker run -d --name user -p 8081:8081 user-service:1.0
docker run -d --name order -p 8082:8082 -e USER_SERVICE_URL=http://host.docker.internal:8081/users order-service:1.0
curl http://localhost:8082/orders
docker rm -f user order
```

---

## 6  Load Images into Minikube

```bash
minikube start --driver=docker
minikube image load user-service:1.0
minikube image load order-service:1.0
```

---

## 7  Kubernetes Manifests (`k8s/`)

**ConfigMaps**

```yaml
# user-service-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: user-service-config }
data:
  SERVER_PORT: "8081"
  APP_NAME: "user-service"
```

```yaml
# order-service-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: order-service-config }
data:
  SERVER_PORT: "8082"
  APP_NAME:  "order-service"
  USER_SERVICE_URL: "http://user-service:8081/users"
```

**Deployments & Services** (one pair per service; ports 8081/8082 accordingly, `envFrom` → ConfigMap).

---

## 8  Apply, Test, Scale, Clean Up

```bash
kubectl apply -f k8s
kubectl get pods
kubectl port-forward service/order-service 8082:8082
curl http://localhost:8082/orders          # expect composite response
kubectl scale deployment order-service --replicas=3
kubectl delete -f k8s
```

---

## 9  Optional Extras
* Push images to Docker Hub  
* Add Horizontal Pod Autoscaler  
* Wire into CI/CD  

---

🎉 **Done!** You’ve built, containerized, and orchestrated two Spring Boot microservices entirely on your laptop.
