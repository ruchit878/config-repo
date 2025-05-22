
# **Lab 17: Deploy and Orchestrate Microservices on Kubernetes with ConfigMaps (Spring Boot 3.4.5)**

## **Objective**
Deploy two independent Spring Boot 3.4.5 microservices—`user-service` and `order-service`—onto a local **Kubernetes** cluster.  
You will:

1. Containerize each service with Docker.
2. Load the images into **Minikube’s** registry.
3. Externalize all runtime settings in `application.properties` using **placeholders** (`${ENV_VAR:default}`).
4. Inject real values at deployment time using **ConfigMaps**.
5. Verify, scale, and clean up the cluster.

---

## **Part 1 – Install & Verify Kubernetes Tooling**

```bash
# 1. Install Minikube (see https://minikube.sigs.k8s.io/docs/start/)
# 2. Install kubectl  (see https://kubernetes.io/docs/tasks/tools/)
minikube version            # 3. Verify installation
minikube start --driver=docker
kubectl cluster-info        # 4. Verify cluster
minikube status
```

---

## **Part 2 – Create the Microservices**

> The two services live in sibling Maven projects: `user-service/` and `order-service/`.

### 2‑A  `user-service`

<details>
<summary>pom.xml</summary>

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" …>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>user-service</artifactId>
  <version>1.0.0</version>

  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.4.5</version>
  </parent>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
  </dependencies>
  <properties><java.version>17</java.version></properties>
</project>
```
</details>

<details>
<summary>Application & Controller</summary>

```java
// UserServiceApplication.java
@SpringBootApplication
public class UserServiceApplication {
  public static void main(String[] args) {
    SpringApplication.run(UserServiceApplication.class, args);
  }
}

// UserController.java
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
<summary>application.properties (placeholders!)</summary>

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

---

### 2‑B  `order-service`

The structure is identical; differences are highlighted.

<details>
<summary>Main files</summary>

```java
// OrderServiceApplication.java
@SpringBootApplication
public class OrderServiceApplication {
  public static void main(String[] args) {
    SpringApplication.run(OrderServiceApplication.class, args);
  }
}

// OrderController.java
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
</details>

<details>
<summary>application.properties</summary>

```properties
server.port=${SERVER_PORT:8082}
app.name=${APP_NAME:order-service}
user.service.url=${USER_SERVICE_URL:http://user-service:8081/users}
```
</details>

<details>
<summary>Dockerfile</summary>

```dockerfile
FROM eclipse-temurin:17-jre
ARG JAR_FILE=target/order-service-1.0.0.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```
</details>

---

## **Part 3 – Build Images & Load into Minikube**

```bash
# From repo root
mvn -pl user-service clean package
docker build -t user-service:1.0 ./user-service

mvn -pl order-service clean package
docker build -t order-service:1.0 ./order-service

minikube image load user-service:1.0
minikube image load order-service:1.0
```

---

## **Part 4 – ConfigMaps (external config)**

```yaml
# user-service-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-service-config
data:
  SERVER_PORT: "8081"
  APP_NAME: "user-service"
  DATABASE_URL: "jdbc:mysql://user-database:3306/userdb"
```

```yaml
# order-service-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
data:
  SERVER_PORT: "8082"
  APP_NAME: "order-service"
  USER_SERVICE_URL: "http://user-service:8081/users"
```

Apply them:

```bash
kubectl apply -f user-service-configmap.yaml
kubectl apply -f order-service-configmap.yaml
```

---

## **Part 5 – Deployments & Services**

<details><summary>user-service-deployment.yaml</summary>

```yaml
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
</details>

<details><summary>user-service-service.yaml</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
    - port: 8081
      targetPort: 8081
  type: ClusterIP
```
</details>

<details><summary>order-service-deployment.yaml</summary>

```yaml
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
</details>

<details><summary>order-service-service.yaml</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
    - port: 8082
      targetPort: 8082
  type: ClusterIP
```
</details>

Apply:

```bash
kubectl apply -f user-service-deployment.yaml -f user-service-service.yaml
kubectl apply -f order-service-deployment.yaml -f order-service-service.yaml
```

---

## **Part 6 – Test, Scale, Clean Up**

```bash
kubectl get pods
kubectl port-forward service/order-service 8082:8082 &
curl http://localhost:8082/orders                 # should call user‑service

# Scale order-service to 3 replicas
kubectl scale deployment order-service --replicas=3
kubectl get pods -l app=order-service

# Clean up
kubectl delete deployment user-service order-service
kubectl delete service user-service order-service
kubectl delete configmap user-service-config order-service-config
```

---

## **Why Placeholders Matter**

* In the **image**: properties are `${ENV_VAR:default}`—generic, portable, rebuild‑free.  
* In **Docker/Kubernetes**: we provide real values as **environment variables** (`envFrom:` ConfigMap).  
* Result: the same image runs **anywhere** with different configs simply by swapping a ConfigMap or `docker run -e`.

Enjoy orchestrating your Spring Boot microservices with declarative, externalized configuration!
