
# **Lab 13: Distributed Tracing with Micrometer Tracing (Brave) + Zipkin (Spring Boot 3.4.5, Windows + IntelliJ)**

## **Objective**
Set up two Spring Boot 3.4.5 microservices that **emit distributed traces** end-to-end.

1. Run a local **Zipkin** server (trace collector + UI).  
2. Build a **producer service** (`UserService`) that returns users.  
3. Build a **consumer service** (`OrderService`) that calls `UserService` with **WebClient**.  
4. Verify that **every request shares the same `traceId`** and appears in the Zipkin UI.

All steps use **Windows 10/11**, **PowerShell (Admin)**, and **IntelliJ IDEA**.

---

## **Installing Prerequisites**

| Tool | Install Command / Download | Purpose in this Lab |
|------|----------------------------|---------------------|
| **Java 17 SDK** | `winget install --id EclipseAdoptium.Temurin.17.JDK` | Runtime for Spring Boot 3.x |
| **Git** | `winget install --id Git.Git` | Optional cloning / version control |
| **Maven 3.9+** | `winget install --id Apache.Maven`  *(skip if you’ll use the generated **mvnw**)* | Builds the projects |
| **IntelliJ IDEA Community** | `winget install --id JetBrains.IntelliJIDEA.Community` | IDE to edit and run code |
| **Docker Desktop** | <https://www.docker.com/products/docker-desktop/> | One-liner to run Zipkin |
| **curl** | Built-in on Win 10/11 (`winget install Curl.Curl` if missing) | Quick HTTP tests |

> **After installing**, close and re-open **PowerShell (Admin)** so the new tools are on your `PATH`.

---

## **Tech Explainer — Micrometer Tracing & Zipkin**
**Micrometer Tracing** (with the Brave bridge) generates trace **spans** inside Spring Boot 3.x apps and propagates context between services. **Zipkin** is a lightweight backend that stores those spans and offers a UI to inspect timing and causal relationships.

---

## **Part 1 – Run Zipkin**

| Step | Window | Command (PowerShell Admin) | Expected Output |
|------|--------|----------------------------|-----------------|
| **A – Verify Java** | — | `java -version` | Line containing `openjdk 17` |
| **B – Start Zipkin (Docker)** | 1 | `docker run -d -p 9411:9411 openzipkin/zipkin` | Container ID hash |
| *Alt B′ – Run Zipkin JAR* | 1 | `java -jar zipkin-server-*-exec.jar` | `Started ZipkinServer` |
| **C – Check UI** | Browser | Go to <http://localhost:9411> | Zipkin search page loads |

> **Troubleshoot:** Page blank? Ensure Docker Desktop is running and port **9411** is free.

---

## **Part 2 – Producer Service (`UserService`)**

> **Tech Explainer — Producer vs. Consumer**  
> `UserService` replies to `/users`. Each request is wrapped in a **SERVER** span and exported to Zipkin.

### 2.1 Create Project

| Setting | Value |
|---------|-------|
| **Folder** | `C:\\Projects\\Lab13\\user-service` |
| **Spring Initializr** | <https://start.spring.io> |
| **Group / Artifact / Name** | `com.microservices` / `user-service` / `user-service` |
| **Dependencies** | **Spring Web**, **Spring Boot Actuator** |

Download ZIP → **Extract All…** → open folder in IntelliJ (*File → Open…*).

### 2.2 Add Tracing Dependencies  
`pom.xml` → inside `<dependencies>` (no version tags):

```xml
<!-- Micrometer Tracing API bridged to Brave -->
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>

<!-- Sends spans to Zipkin -->
<dependency>
  <groupId>io.zipkin.reporter2</groupId>
  <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

### 2.3 `application.properties`  
`src/main/resources/application.properties`

```properties
server.port=8081
spring.application.name=user-service

management.tracing.sampling.probability=1.0
management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans

logging.pattern.level=%5p [${spring.application.name},traceId=%X{traceId},spanId=%X{spanId}]
```

### 2.4 Add REST Controller  
`src/main/java/com/microservices/userservice/UserController.java`

```java
package com.microservices.userservice;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserController {

    @GetMapping("/users")
    public String getUsers() {
        return "List of users from UserService";
    }
}
```

### 2.5 Run `UserService`

```powershell
cd C:\Projects\Lab13\user-service
mvn spring-boot:run
```

Tail of console:

```
... Started UserServiceApplication ...
INFO  ... traceId=abc123 spanId=def456
```

---

## **Part 3 – Consumer Service (`OrderService`)**

> **Tech Explainer — Client & Server Spans**  
> `OrderService` creates a **CLIENT** span when it calls `/users`, then a **SERVER** span for its own `/orders` endpoint.

### 3.1 Create Project

| Setting | Value |
|---------|-------|
| **Folder** | `C:\\Projects\\Lab13\\order-service` |
| **Group / Artifact / Name** | `com.microservices` / `order-service` / `order-service` |
| **Dependencies** | **Spring Web**, **Spring Reactive Web**, **Spring Boot Actuator** |

### 3.2 Add the Same Tracing Deps  
Paste the two `<dependency>` blocks into `order-service/pom.xml`.

### 3.3 `application.properties`  
`src/main/resources/application.properties`

```properties
server.port=8082
spring.application.name=order-service

management.tracing.sampling.probability=1.0
management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans

logging.pattern.level=%5p [${spring.application.name},traceId=%X{traceId},spanId=%X{spanId}]
```

### 3.4 Register WebClient Bean  
`OrderServiceApplication.java`

```java
@Bean
public WebClient.Builder webClientBuilder() {
    return WebClient.builder();
}
```

### 3.5 Add REST Controller  
`OrderController.java`

```java
@GetMapping("/orders")
public String getOrders() {
    String users = webClientBuilder.build()
            .get()
            .uri("http://localhost:8081/users")
            .retrieve()
            .bodyToMono(String.class)
            .block();
    return "Orders from OrderService and Users: " + users;
}
```

### 3.6 Run `OrderService`

```powershell
cd C:\Projects\Lab13\order-service
mvn spring-boot:run
```

Logs show the **same `traceId`** as `UserService` for each request.

---

## **Part 4 – Visualise Traces in Zipkin**

| Step | Window | Command / Action | Expected Result |
|------|--------|------------------|-----------------|
| **A – Generate traffic** | New PowerShell Admin | `curl http://localhost:8082/orders` | Response string |
| **B – Open Zipkin UI** | Browser | <http://localhost:9411> | Search page |
| **C – Click “Run Query”** | — | — | List of traces appears |
| **D – Inspect a trace** | — | Click any trace row | Timeline shows:<br>• `order-service (SERVER)`<br>• `order-service → user-service (CLIENT)`<br>• `user-service (SERVER)` |

---

## **Optional Exercises**

* Add a `ProductService` for a three-hop trace.  
* Drop `management.tracing.sampling.probability` to 0.5 and watch the sample rate.  
* Insert `Thread.sleep()` in `UserService` and watch latency in Zipkin.

---

## **Troubleshooting**

| Symptom | Likely Cause | Quick Fix |
|---------|--------------|-----------|
| Zipkin UI blank | Zipkin not running | `docker ps`; restart container |
| Port 9411 busy | Another Zipkin instance | Stop old container or run `-p 9412:9411` and update endpoint |
| No `traceId` in logs | Wrong dependencies (Sleuth on classpath) | Remove `spring-cloud-sleuth-*`; keep Micrometer deps |
| `/orders` hangs | `UserService` not up | Start `UserService` first |

---

## **Conclusion**
You now have **distributed tracing** on Spring Boot 3.4.5 via **Micrometer Tracing + Brave**, with spans flowing to **Zipkin**. Each request travels across services under a single `traceId`, and you can inspect every hop, latency, and relationship in the Zipkin UI. 🎉
