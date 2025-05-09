# **Lab 12: Implement Distributed Tracing with Spring Boot 3.4.5 (Micrometer + Brave + Zipkin)**

---

## Installing Prerequisites 🚀
| Tool | One‑line install | Purpose |
|------|-----------------|---------|
| **Java 17 +** | `winget install --silent EclipseAdoptium.Temurin.17.JDK` | Runs Spring Boot 3.4.5 |
| **Maven 3.9 +** | `winget install Apache.Maven` | Builds & runs projects |
| **Docker Desktop** | <https://www.docker.com/products/docker-desktop/> | Hosts Zipkin container |
| **IDE (IntelliJ / VS Code)** | Download from vendor | Edit & run code |

> **Why Micrometer instead of Sleuth?**  
> Spring Cloud Sleuth was removed in Spring Boot 3.4.x. Micrometer Tracing (with a Brave bridge) is the official successor. The property namespace changed from `spring.sleuth.*` ➜ `management.tracing.*`, and Zipkin became the default exporter.

---

## Quick‑Start Topology
```
curl ─▶ OrderService (8082) ─┬─▶ UserService (8081)
                              └─▶ Zipkin (9411 docker)
```

---

## Part 1 – Create **UserService**

| Field | Value |
|-------|-------|
| **Project** | **UserService** |
| Group ID | `com.microservices` |
| Artifact ID | `user-service` |
| Spring Boot | `3.4.5` |
| Packaging | JAR (default) |
| Java | 17 |
| **Dependencies (add on start.spring.io)** | *Spring Web*, *Spring Boot Actuator* |

> Download the generated ZIP and extract to `C:\Projects\Lab12\user-service`.

1. **Import into IDE**  
   *IntelliJ*: **File → Open →** `user-service`  

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

2. **Create REST controller** – `src/main/java/com/microservices/userservice/UserController.java`
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

3. **Configure tracing** – `src/main/resources/application.properties`
```properties
server.port=8081
spring.application.name=user-service

management.tracing.sampling.probability=1.0
management.tracing.brave.trace-id128=true
management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans

management.endpoints.web.exposure.include=*
logging.pattern.level=%5p [${spring.application.name},traceId=%X{traceId},spanId=%X{spanId}]
```

4. **Run service**
```bash
cd user-service
mvn spring-boot:run
```
**Expected**
```text
...Started UserServiceApplication...
INFO  [user-service,traceId=...,spanId=...]
```

5. **Verify endpoint** (run Powershell as administrator)
```bash
curl http://localhost:8081/users
```
**Expected**
```text
List of users from UserService
```
---

## Part 2 – Create **OrderService**

| Field | Value |
|-------|-------|
| **Project** | **OrderService** |
| Group ID | `com.microservices` |
| Artifact ID | `order-service` |
| Spring Boot | `3.4.5` |
| Packaging | JAR (default) |
| Java | 17 |
| **Dependencies (add on start.spring.io)** | *Spring Web*, *Spring Reactive Web*, *Spring Boot Actuator* |

> Extract ZIP to `C:\Projects\Lab12\order-service` and open in your IDE.

1. **Import into IDE**  
   *IntelliJ*: **File → Open →** `order-service`  

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

2. **Create controller** – `src/main/java/com/microservices/orderservice/OrderController.java`
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
        .get().uri("http://localhost:8081/users")
        .retrieve().bodyToMono(String.class).block();
    return "Orders from OrderService and Users: " + users;
  }
}
```

3. **Register WebClient bean** – `OrderServiceApplication.java`
```java
@Bean
public WebClient.Builder webClientBuilder() {
  return WebClient.builder();
}
```

4. **Configure tracing** – `src/main/resources/application.properties`
```properties
server.port=8082
spring.application.name=order-service

management.tracing.sampling.probability=1.0
management.tracing.brave.trace-id128=true
management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans

management.endpoints.web.exposure.include=*
logging.pattern.level=%5p [${spring.application.name},traceId=%X{traceId},spanId=%X{spanId}]
```

5. **Run service**
```bash
cd order-service
mvn spring-boot:run
```

6. **End‑to‑end test** (run Powershell as administrator)
```bash
curl http://localhost:8082/orders
```
**Expected**
```text
Orders from OrderService and Users: List of users from UserService
```
---

## Part 3 – Zipkin UI (run Powershell as administrator)

1. **Start Zipkin**
```bash
docker run -d -p 9411:9411 openzipkin/zipkin
```

2. **Open** <http://localhost:9411> → click **Run Query** after calling `/orders` again.  
   You’ll see **order-service → user-service** trace.
---

## Conclusion 🎉
You now have **128‑bit distributed tracing** with Spring Boot 3.4.5 using **Micrometer + Zipkin**, with logs and UI verification. Just remember to manually add the required tracing dependencies after generation.
