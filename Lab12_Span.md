
# **Lab 12: Implement Distributed Tracing with Spring Boot 3.4.5 Using Micrometer Tracing (Brave) and Zipkin**

> **What’s new in this revision?**  
> * Added Actuator + Micrometer‑Brave + Zipkin reporter dependencies explicitly to both services  
> * Clarified how *spans* appear (logs & Zipkin) and how to create a **custom span**  
> * Tweaked logging pattern so every log line prints `traceId` and `spanId`

---

## **Objective**

Enable **distributed tracing** across multiple microservices with **Micrometer Tracing (Brave bridge)**—the successor to Spring Cloud Sleuth in Spring Boot 3.4.x.  
You will:

* Instrument two microservices (`UserService` and `OrderService`) so every incoming request, WebClient call, and actuator operation is wrapped in a *span*.  
* Propagate the **trace ID** automatically across service boundaries.  
* Ship all finished spans to **Zipkin** and view end‑to‑end traces in its UI.  
* (Optional) create a **custom span** around any block of code.

---

## **Prerequisites**

| Tool            | Version / Notes |
|-----------------|-----------------|
| Java            | 17 or newer |
| Maven           | Latest 3.x |
| Docker Desktop  | Running (needed for Zipkin) |
| IDE             | IntelliJ IDEA, Eclipse, or VS Code |

---

## **Part 1 – Create & Configure *UserService***

### 1. Generate the project

1. Go to **<https://start.spring.io>**  
2. Select  
   * **Group** `com.microservices`  
   * **Artifact** `user-service`  
   * **Spring Boot** `3.4.5`  
   * **Dependencies**   
     * *Spring Web*  
     * *Spring Boot Actuator*  
3. Click **Generate** and unzip the archive.

### 2. Import into your IDE
Open or import the Maven project (`user-service`).

### 3. Add tracing dependencies

Edit `pom.xml` and **add these three** (if not already present):

```xml
<!-- Micrometer Tracing API → Brave implementation -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>

<!-- Ships spans to Zipkin -->
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

*(Spring Boot Actuator was selected at generation time.)*

### 4. Create the REST controller

`src/main/java/com/microservices/userservice/UserController.java`:

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

### 5. Configure tracing & logging

`src/main/resources/application.properties`:

```properties
server.port=8081
spring.application.name=user-service

# — Trace everything
management.tracing.sampling.probability=1.0
# — Send spans to the local Zipkin collector
management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans

# — Expose all actuator endpoints (optional but handy)
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
management.endpoint.env.show-values=always

# — Print traceId + spanId with every log line
logging.pattern.level=%5p [${spring.application.name},traceId=%X{traceId},spanId=%X{spanId}]
```

### 6. Run *UserService*

```bash
mvn spring-boot:run
```

Verify:  
<http://localhost:8081/users> → console logs should now include `traceId` and `spanId`.

---

## **Part 2 – Create & Configure *OrderService***

### 7. Generate the project

Repeat **Spring Initializr**:

| Setting  | Value |
|----------|-------|
| Group    | `com.microservices` |
| Artifact | `order-service` |
| Spring Boot | `3.4.5` |
| Dependencies | *Spring Web*, *Spring Reactive Web*, *Spring Boot Actuator* |

### 8. Import into IDE

Open/import the `order-service` project.

### 9. Add tracing dependencies

Same three as in *UserService*:

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

### 10. Add WebClient bean

`src/main/java/com/microservices/orderservice/OrderServiceApplication.java`:

```java
@Bean
public WebClient.Builder webClientBuilder() {
    return WebClient.builder();
}
```

### 11. Create the REST controller

`src/main/java/com/microservices/orderservice/OrderController.java`:

```java
package com.microservices.orderservice;

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
            .uri("http://localhost:8081/users") // call UserService
            .retrieve()
            .bodyToMono(String.class)
            .block();

        return "Orders from OrderService and Users: " + users;
    }
}
```

### 12. Configure tracing & logging

`src/main/resources/application.properties`:

```properties
server.port=8082
spring.application.name=order-service
management.tracing.sampling.probability=1.0
management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans
management.endpoints.web.exposure.include=*
logging.pattern.level=%5p [${spring.application.name},traceId=%X{traceId},spanId=%X{spanId}]
```

### 13. Run *OrderService*

```bash
mvn spring-boot:run
```

---

## **Part 3 – Launch Zipkin & Verify Traces**

### 14. Start Zipkin (Docker)

```bash
docker run -d -p 9411:9411 openzipkin/zipkin
```

### 15. Generate a trace

```bash
curl http://localhost:8082/orders
```

Check each service’s console: you should see the **same `traceId`** but different **`spanId`s**.

### 16. View in Zipkin UI

1. Open <http://localhost:9411>  
2. Click **Run Query** → a row appears.  
3. Click it to expand and view the timeline (`OrderService ➜ UserService`).

---

## **Optional – Creating a Custom Span**

```java
import io.micrometer.observation.Observation;
import io.micrometer.observation.ObservationRegistry;
import org.springframework.stereotype.Service;

@Service
public class ReportGenerator {

    private final ObservationRegistry registry;

    public ReportGenerator(ObservationRegistry registry) {
        this.registry = registry;
    }

    public void generate() {
        Observation.createNotStarted("report-generation", registry)
                   .lowCardinalityKeyValue("reportType", "monthly")
                   .observe(() -> {
                       // your long‑running logic …
                   });
    }
}
```

The new span (`report-generation`) will appear nested under the current trace in Zipkin and in your logs with its own `spanId`.

---

## **Recap**

* Micrometer‑Brave auto‑creates **server** and **client** spans for HTTP & WebClient calls.  
* Adding the Zipkin reporter ships those spans to Zipkin.  
* Custom log pattern prints `traceId` + `spanId` for fast console filtering.  
* Optional `Observation` API lets you wrap any business logic in your own span.

You now have full, end‑to‑end observability across your Spring Boot microservices!
