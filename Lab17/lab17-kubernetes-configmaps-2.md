
# **Lab 17: Deploy and Orchestrate Microservices on Kubernetes with ConfigMaps (Spring Boot 3.4.5)**

## **Objective**
Deploy two independent Spring Boot 3.4.5 microservices—`user-service` and `order-service`—onto a local **Kubernetes** cluster.  
You will:

1. **Generate** starter projects with **Spring Initializr**.
2. **Containerize** each service with Docker.
3. **Load** the images into **Minikube’s** registry.
4. Externalize all runtime settings in `application.properties` using **placeholders** (`${ENV_VAR:default}`).
5. Inject real values at deployment time using **ConfigMaps**.
6. Verify, scale, and clean up the cluster.

---

## **Part 1 – Install & Verify Kubernetes Tooling**

```bash
# 1. Install Minikube (see https://minikube.sigs.k8s.io/docs/start/)
# 2. Install kubectl  (see https://kubernetes.io/docs/tasks/tools/)
minikube version            # 3. Verify installation
minikube start --driver=docker
kubectl cluster-info        # 4. Verify cluster
minikube status
```

---

## **Part 2 – Create the Microservices**

### 2‑A  Generate skeletons with Spring Initializr

You can use either the **web UI** or the **curl API**.

| Setting               | Value                     |
|-----------------------|---------------------------|
| **Project**           | Maven                     |
| **Language**          | Java 17                  |
| **Spring Boot**       | 3.4.5                    |
| **Packaging**         | Jar                      |
| **Dependencies**      | Spring Web               |
| **Group**             | `com.example`            |
| **Artifact**          | `user-service` / `order-service` |

<details>
<summary>Using the web UI</summary>

1. Browse to **https://start.spring.io/**.  
2. Fill in the table values above for **user‑service**.  
3. Click **Generate** → unzip to `user-service/`.  
4. Repeat for **order‑service** (artifact = `order-service`).  
</details>

<details>
<summary>Using the curl API</summary>

```bash
curl https://start.spring.io/starter.zip      -d dependencies=web      -d bootVersion=3.4.5      -d javaVersion=17      -d type=maven-project      -d groupId=com.example      -d artifactId=user-service      -o user-service.zip

unzip user-service.zip -d user-service
rm user-service.zip

curl https://start.spring.io/starter.zip      -d dependencies=web      -d bootVersion=3.4.5      -d javaVersion=17      -d type=maven-project      -d groupId=com.example      -d artifactId=order-service      -o order-service.zip

unzip order-service.zip -d order-service
rm order-service.zip
```
</details>

> **Why generate first?**  
> Spring Initializr builds a ready‑to‑run Maven structure (`src/main/java`, `src/test/java`, `application.properties`, etc.) so all you have to do is add controllers and a Dockerfile.

---

### 2‑B  Add business code & placeholders

> The sections below show only the files you must **add or modify** after generation.

#### user‑service

<details>
<summary>src/main/java/com/example/user/UserServiceApplication.java</summary>

```java
package com.example.user;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class UserServiceApplication {
  public static void main(String[] args) {
    SpringApplication.run(UserServiceApplication.class, args);
  }
}
```
</details>

<details>
<summary>src/main/java/com/example/user/controller/UserController.java</summary>

```java
package com.example.user.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserController {

  @Value("${app.name}")
  private String appName;

  @GetMapping("/users")
  public String getUsers() {
    return "Users from " + appName;
  }
}
```
</details>

<details>
<summary>src/main/resources/application.properties</summary>

```properties
server.port=${SERVER_PORT:8081}
app.name=${APP_NAME:user-service}
database.url=${DATABASE_URL:jdbc:mysql://localhost:3306/userdb}
```
</details>

<details>
<summary>Dockerfile</summary>

```dockerfile
FROM eclipse-temurin:17-jre
ARG JAR_FILE=target/user-service-1.0.0.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```
</details>

#### order‑service

<details>
<summary>Order code & config</summary>

```java
// OrderServiceApplication.java
package com.example.order;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class OrderServiceApplication {
  public static void main(String[] args) {
    SpringApplication.run(OrderServiceApplication.class, args);
  }
}
```

```java
// OrderController.java
package com.example.order.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.client.RestClient;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class OrderController {

  @Value("${app.name}")
  private String appName;

  @Value("${user.service.url}")
  private String userServiceUrl;

  private final RestClient rest = RestClient.create();

  @GetMapping("/orders")
  public String getOrders() {
    String users = rest.get().uri(userServiceUrl).retrieve().body(String.class);
    return "Orders from " + appName + " / " + users;
  }
}
```

```properties
# src/main/resources/application.properties
server.port=${SERVER_PORT:8082}
app.name=${APP_NAME:order-service}
user.service.url=${USER_SERVICE_URL:http://user-service:8081/users}
```

```dockerfile
# Dockerfile
FROM eclipse-temurin:17-jre
ARG JAR_FILE=target/order-service-1.0.0.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```
</details>

---

## **Part 3 – Build Images & Load into Minikube**

```bash
mvn -pl user-service clean package
docker build -t user-service:1.0 ./user-service

mvn -pl order-service clean package
docker build -t order-service:1.0 ./order-service

minikube image load user-service:1.0
minikube image load order-service:1.0
```

---

## **Part 4 – ConfigMaps, Deployments & Services**

> These YAMLs are unchanged from the earlier version—see previous sections if you need a reference.

1. Create `user-service-configmap.yaml` and `order-service-configmap.yaml`.
2. Apply them: `kubectl apply -f …`.
3. Create Deployment and Service YAMLs for each service.
4. Apply them with `kubectl apply -f …`.

---

## **Part 5 – Test, Scale, Clean Up**

```bash
kubectl port-forward service/order-service 8082:8082 &
curl http://localhost:8082/orders

kubectl scale deployment order-service --replicas=3

kubectl delete deployment,user-service,order-service
kubectl delete service user-service order-service
kubectl delete configmap user-service-config order-service-config
```

---

## **Why Placeholders Matter**

* **Inside the image**: properties are `${ENV_VAR:default}`—generic, portable, rebuild‑free.  
* **Inside Docker/K8s**: real values come from **environment variables** via ConfigMaps or `docker run -e`.  
* **Result**: the same image runs **anywhere** with different configs simply by swapping a ConfigMap.

Happy coding & containerizing!
