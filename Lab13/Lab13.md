# **Lab 13: Distributed Tracing on Spring Boot 3.4.5 with Micrometer Tracing (Brave) + Zipkin**

> **Why this lab was updated**  
> Spring Cloud Sleuth (used in Boot 3.4.1) is no longer supported on Spring Boot 3.x.  
> From Boot 3.0 onward, use **Micrometer Tracing** (with the Brave bridge) to generate spans and export them to **Zipkin**.

---

## 🛠️ Installing Prerequisites (complete once)

| Tool | One-line install (Windows ▶ macOS ▶ Ubuntu) | Purpose |
|------|---------------------------------------------|---------|
| **Java 17 +** | `winget install --id EclipseAdoptium.Temurin.17.JDK`| Run Spring Boot apps |
| **Maven 3.9 +** | `winget install Apache.Maven`| Build / run projects (`mvnw` wrapper works too) |
| **Docker Desktop** | `winget install Docker.DockerDesktop` | Fastest way to launch Zipkin |
| **curl** | pre-installed | Trigger HTTP calls |
| **IDE (IntelliJ Community / VS Code)** | `winget install JetBrains.IntelliJIDEA.Community`| Code editing & run |

> **If `java -version` fails:** install JDK 17 using a command above or from <https://adoptium.net/>.

---

## **Objective**

Configure two Spring Boot 3.4.5 microservices (`UserService` and `OrderService`) so that:

* Each HTTP request and WebClient call is wrapped in a **span**  
* Spans are sent to a local **Zipkin** collector (`localhost:9411`)  
* The Zipkin UI visualises complete, end-to-end traces

---

## **Tech Explainers (read once)**

**Spring Boot** – rapid framework for production-ready Java services.  
**Micrometer Tracing** – captures **spans** and propagates them across threads/services.  
**Brave bridge** – lets Micrometer speak classic Brave/Zipkin format.  
**Zipkin** – server/UI that stores and renders traces.

---

## **Lab Steps**

### **Part 1 – Run Zipkin locally (run on Powershell as Administrator)**

1. **Verify Java 17 +**

   ```bash
   java -version
   ```
   **Expected**

   ```text
   openjdk 17.0.x …
   ```

2. **Start Zipkin**

   ```bash
   docker run -d -p 9411:9411 openzipkin/zipkin
   ```
   **Expected**

   ```text
   8b1f3a1e2c5d  # container ID
   ```

3. **Open the UI**

   Browse to <http://localhost:9411> → “Zipkin Search” page should load.

---

### **Part 2 – Producer microservice (*UserService*)**

| # | Action | Details / Commands |
|---|--------|-------------------|
| **1** | **Generate project on <https://start.spring.io>** | **Group** `com.microservices` • **Artifact** `user-service` • **Boot** `3.4.5` |
| | **Select dependencies** | **Spring Web**, **Spring Boot Actuator**, **Micrometer Tracing (Brave)**, **Zipkin Reporter (Brave)** |
| **2** | **Unzip & import into IDE** | IntelliJ → *File ▸ Open…* → select folder |
| **3** | **Configure tracing** | `src/main/resources/application.properties` → see below |
| **4** | **Add controller** | Simple `/users` endpoint |
| **5** | **Run service** | `mvn spring-boot:run` or IDE **Run** |
| **6** | **Smoke-test** | `curl http://localhost:8081/users` |

> 🔧 **After importing**, manually add the following dependencies to your `pom.xml`:

```xml
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
  <groupId>io.zipkin.reporter2</groupId>
  <artifactId>zipkin-reporter-brave</artifactId>
</dependency>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>2023.0.2</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

**application.properties**

```properties
server.port=8081
spring.application.name=user-service

management.tracing.sampling.probability=1.0   # trace every request
management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans
# 1.0 = trace every request; reduce in prod

logging.pattern.level=%5p [${spring.application.name},traceId=%X{traceId},spanId=%X{spanId}]
```

**UserController.java**

```java
package com.microservices.user_service;

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

---

### **Part 3 – Consumer microservice (*OrderService*)**

| # | Action | Details / Commands |
|---|--------|-------------------|
| **1** | **Generate project** | **Artifact** `order-service` • same Boot version |
| | **Select dependencies** | **Spring Web**, **Spring Reactive Web**, **Spring Boot Actuator**, **Micrometer Tracing (Brave)**, **Zipkin Reporter (Brave)** |
| **2** | **Import into IDE** | as above |
| **3** | **Configure tracing** | `src/main/resources/application.properties` below |
| **4** | **Add WebClient bean** | see code snippet |
| **5** | **Add controller** | `/orders` endpoint calls `UserService` |
| **6** | **Run service** | `mvn spring-boot:run` |
| **7** | **Smoke-test** | `curl http://localhost:8082/orders` |

> 🔧 **After importing**, manually add the following dependencies to your `pom.xml`:

```xml
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
  <groupId>io.zipkin.reporter2</groupId>
  <artifactId>zipkin-reporter-brave</artifactId>
</dependency>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>2023.0.2</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

**application.properties**

```properties
server.port=8082
spring.application.name=order-service

management.tracing.sampling.probability=1.0
management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans

logging.pattern.level=%5p [${spring.application.name},traceId=%X{traceId},spanId=%X{spanId}]
```

**OrderServiceApplication.java**

```java
package com.microservices.order_service;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.web.reactive.function.client.WebClient;

@SpringBootApplication
public class OrderServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }

    @Bean
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }
}
```

**OrderController.java**

```java
package com.microservices.order_service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.reactive.function.client.WebClient;

@RestController
public class OrderController {

    @Autowired
    private WebClient.Builder webClientBuilder;

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
}
```

---

### **Part 4 – Visualise traces in Zipkin (run on Powershell as Administrator)**

1. **Generate traffic**

   ```bash
   curl http://localhost:8082/orders
   ```

2. **Open Zipkin** → click **Find Traces** → expect spans for `order-service` and `user-service`.

   It is reached by running http://localhost:9411

---

## **Troubleshooting Quick-ref**

| Symptom | Cause | Fix |
|---------|-------|-----|
| Zipkin UI blank | Zipkin not running | `docker ps` → restart container |
| Port clash | Other app on 8081/8082/9411 | Change `server.port` / use diff `-p` |
| No `traceId` in logs | Sleuth jars included | Remove any `spring-cloud-sleuth-*` deps |

---

## **Conclusion 🎉**

You built two Spring Boot 3.4.5 services with **Micrometer Tracing (Brave)**, pushed spans to **Zipkin**, and visualised end-to-end traces—no manual Maven edits required.
