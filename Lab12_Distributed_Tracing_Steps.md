
# **Lab 12: Implement Distributed Tracing with Spring Boot 3.4.1 Using Micrometer Tracing and Zipkin**

## **Objective**
Enable distributed tracing across multiple microservices using **Micrometer Tracing with Brave**, the replacement for Spring Cloud Sleuth in Spring Boot 3.4.1. You'll configure two microservices (`UserService` and `OrderService`) to propagate trace IDs and spans automatically and visualize them using **Zipkin**.

---

## **Step-by-Step Instructions**

### 🔧 Setup

1. **Ensure the following tools are installed and running**:
   - Java 17+
   - Maven
   - Docker Desktop (Running)
   - IDE (IntelliJ IDEA, Eclipse, or Visual Studio Code)

---

### 🛠️ UserService Setup

2. **Generate a Spring Boot project** at [start.spring.io](https://start.spring.io):
   - Group: `com.microservices`
   - Artifact: `user-service`
   - Spring Boot: `3.4.1`
   - Dependencies: Spring Web, Spring Boot Actuator

3. **Extract and import the project into your IDE**.

4. **Add dependencies in `pom.xml`**:
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

5. **Create `UserController.java`** in `src/main/java/com/microservices/userservice/`:
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

6. **Update `application.properties`**:
```properties
server.port=8081
spring.application.name=user-service
management.tracing.sampling.probability=1.0
management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
management.endpoint.env.show-values=always
logging.pattern.level=%5p [${spring.application.name:},traceId=%X{traceId},spanId=%X{spanId}]
```

7. **Run UserService**:
```bash
./mvnw spring-boot:run
```

8. **Test UserService endpoint**:
```
http://localhost:8081/users
```

---

### 🛠️ OrderService Setup

9. **Generate project at [start.spring.io](https://start.spring.io)**:
   - Group: `com.microservices`
   - Artifact: `order-service`
   - Dependencies: Spring Web, Spring Boot Actuator, Spring Reactive Web

10. **Import project into IDE**.

11. **Add dependencies to `pom.xml`** (same as Step 4).

12. **Create `OrderController.java`**:
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

13. **Add WebClient bean in `OrderServiceApplication.java`**:
```java
@Bean
public WebClient.Builder webClientBuilder() {
    return WebClient.builder();
}
```

14. **Update `application.properties`**:
```properties
server.port=8082
spring.application.name=order-service
management.tracing.sampling.probability=1.0
management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
management.endpoint.env.show-values=always
logging.pattern.level=%5p [${spring.application.name:},traceId=%X{traceId},spanId=%X{spanId}]
```

15. **Run OrderService**:
```bash
./mvnw spring-boot:run
```

16. **Test OrderService endpoint**:
```
http://localhost:8082/orders
```

---

### 📊 Zipkin Setup

17. **Start Zipkin using Docker**:
```bash
docker run -d -p 9411:9411 openzipkin/zipkin
```

18. **Open Zipkin UI**:
```
http://localhost:9411
```

19. **Trigger request to generate a trace**:
```
curl http://localhost:8082/orders
```

20. **View trace in Zipkin**: Click **Run Query** in the UI to see the trace flowing from OrderService to UserService.

---

### ✅ Optional Exercises

21. Add a third service (e.g., `ProductService`) and trace all 3 hops.

22. Simulate delay or failure in `UserService` and observe it in Zipkin.

23. Export traces to OpenTelemetry (advanced).

---

### ✅ Conclusion

By following all 23 steps, you now have a full understanding of distributed tracing using Micrometer Tracing + Brave and can visualize trace spans in Zipkin.

