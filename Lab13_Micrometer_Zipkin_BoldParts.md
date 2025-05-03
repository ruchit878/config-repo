
# **Lab 13: Distributed Tracing on Spring Boot 3.4.5 with Micrometer Tracing (Brave) + Zipkin**

> **Why this lab was updated**  
> Spring Cloud Sleuth is no longer supported on Spring Boot 3.x.  
> Starting with Boot 3.0, use **Micrometer Tracing** (with the Brave bridge) to generate spans and export them to **Zipkin**.

---

## **Objective**

Configure two Spring Boot 3.4.5 microservices (`UserService` and `OrderService`) so that:

* Every incoming HTTP request and outgoing WebClient call is wrapped in a **span**.  
* Spans are sent to a locally running **Zipkin** collector (`localhost:9411`).  
* The Zipkin UI visualises complete, end‑to‑end traces.

---

## **Lab Steps**

### **Part 1 – Run Zipkin locally**

1. **Confirm Java 17 + is installed**

   ```bash
   java -version
   ```

2. **Download or run Zipkin**

   *Fastest:*  
   ```bash
   docker run -d -p 9411:9411 openzipkin/zipkin
   ```
   or download `zipkin-server-*-exec.jar` from the [Zipkin quick‑start page](https://zipkin.io/pages/quickstart) and run  
   ```bash
   java -jar zipkin-server-*-exec.jar
   ```

3. **Verify Zipkin UI**

   Open <http://localhost:9411>. You should see the search page.

---

### **Part 2 – Create the producer microservice (**UserService**)**

1. **Generate project (Spring Initializr)**

| Setting            | Value                          |
|--------------------|--------------------------------|
| Spring Boot        | 3.4.5                          |
| Group              | `com.microservices`            |
| Artifact / Name    | `user-service`                 |
| Dependencies       | *Spring Web*, *Spring Boot Actuator* |

2. **Add Micrometer‑Tracing + Zipkin dependencies**

`pom.xml` → inside `<dependencies>`:

```xml
<!-- Micrometer Tracing API bridged to Brave -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>

<!-- Reporter that ships spans to Zipkin -->
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

3. **Configure tracing & span‑aware logging**

`src/main/resources/application.properties`:

```properties
server.port=8081
spring.application.name=user-service

# Send every span to Zipkin
management.tracing.sampling.probability=1.0
management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans

# Include traceId / spanId in every log line
logging.pattern.level=%5p [${spring.application.name},traceId=%X{traceId},spanId=%X{spanId}]
```

4. **Create a simple REST endpoint**

`UserController.java`

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

5. **Run `UserService`**

```bash
./mvnw spring-boot:run
```

Check the console—each log line now prints `traceId` and `spanId`.

---

### **Part 3 – Create the consumer microservice (**OrderService**)**

1. **Generate project**

| Setting            | Value                                       |
|--------------------|---------------------------------------------|
| Artifact / Name    | `order-service`                             |
| Dependencies       | *Spring Web*, *Spring Boot Actuator*, *Spring Reactive Web* |

2. **Add Micrometer‑Tracing + Zipkin dependencies**

`pom.xml` (same two as UserService):

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

3. **Configure tracing & logging**

`src/main/resources/application.properties`:

```properties
server.port=8082
spring.application.name=order-service

management.tracing.sampling.probability=1.0
management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans

logging.pattern.level=%5p [${spring.application.name},traceId=%X{traceId},spanId=%X{spanId}]
```

4. **Register a WebClient bean**

`OrderServiceApplication.java`

```java
package com.microservices.orderservice;

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

5. **Create the REST controller**

`OrderController.java`

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
                .uri("http://localhost:8081/users")
                .retrieve()
                .bodyToMono(String.class)
                .block();

        return "Orders from OrderService and Users: " + users;
    }
}
```

6. **Run `OrderService`**

```bash
./mvnw spring-boot:run
```

Notice the same `traceId` appears in both `OrderService` and `UserService` logs.

---

### **Part 4 – Visualise traces in Zipkin**

1. **Generate traffic**

```bash
curl http://localhost:8082/orders
```

Run a few times.

2. **Open Zipkin UI** (<http://localhost:9411>)

3. **Click “Run Query / Find Traces”**

You should see traces that include:

* `order-service (SERVER)`  
* `order-service → user-service (CLIENT)`  
* `user-service (SERVER)`

4. **Inspect a trace**

The timeline view displays latency for each span and the relationship between services.

---

## **Optional Exercises**

* Add a third microservice (e.g., `ProductService`) to create a three‑hop trace.  
* Introduce a random delay in `UserService` and observe it immediately in Zipkin.  
* Experiment with lower sampling rates by reducing `management.tracing.sampling.probability`.

---
## **Troubleshooting**

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Zipkin UI is blank | Wrong port (Zipkin not on 9411) or services not sending data | Ensure Zipkin is running, verify `management.zipkin.tracing.endpoint`, watch service logs for “Sending spans…” |
| `Zipkin` port already in use | Another Zipkin instance running | Kill old process or start Zipkin on a different port (`--server.port=9412`) and update the endpoint property |
| Logs show no `traceId` / `spanId` | Sleuth still on classpath or missing Micrometer bridge | Remove `spring-cloud-sleuth-*` dependencies, keep only Micrometer Tracing + Brave |

---

With Micrometer Tracing in place, you now have production‑ready distributed tracing on Spring Boot 3.4.5, fully compatible with Zipkin.
