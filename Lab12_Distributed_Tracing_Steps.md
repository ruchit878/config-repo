# **Lab 12: Implement Distributed Tracing with Spring Boot 3.4.5 Using Micrometer Tracing and Zipkin**

## **Objective**
Enable distributed tracing across multiple microservices using **Micrometer Tracing with Brave**, the replacement for Spring Cloud Sleuth in Spring Boot 3.4.5. You'll configure two microservices (`UserService` and `OrderService`) to propagate trace IDs and spans automatically and visualize them using **Zipkin**.

---

## **Prerequisites**

Ensure the following tools are installed and running:
- **Java 17+**
- **Maven**
- **Docker Desktop** (Running)
- **IDE**: IntelliJ IDEA, Eclipse, or Visual Studio Code

---

## **Part 1: Create and Configure UserService**

### Step 1: Generate Project
- Visit [https://start.spring.io](https://start.spring.io)
- Select:
  - Group: `com.microservices`
  - Artifact: `user-service`
  - Spring Boot: `3.4.5`
  - Dependencies:
    - Spring Web
    - Spring Boot Actuator
- Click **Generate** and extract the ZIP.

### Step 2: Import into IDE
- Open your IDE
- Choose "Open Project" or "Import Maven Project"
- Navigate to and select the extracted `user-service` folder

### Step 3: Add Tracing Dependencies in `pom.xml`
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

### Step 4: Create a REST Controller
Create file `src/main/java/com/microservices/userservice/UserController.java`:
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

### Step 5: Configure Application Properties
Edit `src/main/resources/application.properties`:
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

### Step 6: Run UserService
Open terminal in the project folder and run:
```bash
mvn spring-boot:run
```

### Step 7: Test Endpoint
In browser or Postman:
```
http://localhost:8081/users
```

Check the console log for traceId and spanId.

---

## **Part 2: Create and Configure OrderService**

### Step 8: Generate Project
- Visit [https://start.spring.io](https://start.spring.io)
- Select:
  - Group: `com.microservices`
  - Artifact: `order-service`
  - Spring Boot: `3.4.5`
  - Dependencies:
    - Spring Web
    - Spring Boot Actuator
    - Spring Reactive Web
- Click **Generate** and extract the ZIP.

### Step 9: Import into IDE
Same as UserService: File > Open or Import Project and select `order-service`.

### Step 10: Add Tracing Dependencies
Same two dependencies as `UserService` in the `pom.xml`

### Step 11: Create Controller
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
            .uri("http://localhost:8081/users")
            .retrieve()
            .bodyToMono(String.class)
            .block();

        return "Orders from OrderService and Users: " + users;
    }
}
```

### Step 12: Add WebClient Bean
In `OrderServiceApplication.java`:
```java
@Bean
public WebClient.Builder webClientBuilder() {
    return WebClient.builder();
}
```

### Step 13: Configure Application Properties
Edit `src/main/resources/application.properties`:
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

### Step 14: Run OrderService
```bash
mvn spring-boot:run
```

### Step 15: Test Endpoint
```bash
curl http://localhost:8082/orders
```
or visit it in browser.

Check both services' logs for matching traceId.

---

## **Part 3: Run and View Zipkin UI**

### Step 16: Start Zipkin using Docker (Make sure to run Docker Desktop first)
```bash
docker run -d -p 9411:9411 openzipkin/zipkin
```

### Step 17: Access Zipkin UI
Go to:
```
http://localhost:9411
```

### Step 18: Trigger Trace and View
- Visit: `http://localhost:8082/orders`
- In Zipkin UI, click **Run Query**
- You should see trace from `OrderService → UserService`

---

## **Optional Exercises**

✅ Add a third service (e.g., ProductService) and trace all 3 hops.  
✅ Simulate a delay or error in UserService and observe in Zipkin.  
✅ Export traces to OpenTelemetry (advanced).

---

## ✅ **Conclusion**
- You now have full distributed tracing setup using **Micrometer Tracing with Brave**
- Traces appear in logs and visually in **Zipkin UI**
- This lab replaces older Sleuth-based solutions for Spring Boot 3.x

Students should now understand how to implement tracing across microservices for real-world debugging and monitoring.

