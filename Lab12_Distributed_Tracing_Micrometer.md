
# **Lab 12: Implement Distributed Tracing with Spring Boot 3.4.1 Using Micrometer Tracing and Zipkin**

## **Objective**

This lab guides you to enable **distributed tracing** across multiple microservices using **Micrometer Tracing with Brave**, the replacement for Spring Cloud Sleuth in Spring Boot 3.4.1. You'll configure two microservices (`UserService` and `OrderService`) that will propagate trace IDs and spans automatically. You'll also visualize traces using **Zipkin**.

---

## **Lab Steps**

### **🧰 Part 1: Set Up Prerequisites**

- **Java 17+**
- **Maven**
- **IDE** (e.g., IntelliJ, VS Code)
- **Docker Desktop (for Zipkin UI)**

---

### **🛠️ Part 2: Create UserService**

1. **Generate the project at [start.spring.io](https://start.spring.io):**
   - Group: `com.microservices`
   - Artifact: `user-service`
   - Spring Boot: `3.4.1`
   - Dependencies:
     - Spring Web
     - Spring Boot Actuator

2. **Add these dependencies to `pom.xml`:**
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

3. **Create a controller:**
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

4. **Add `application.properties`:**
   ```properties
   server.port=8081
   spring.application.name=user-service
   management.tracing.sampling.probability=1.0
   management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans
   management.endpoints.web.exposure.include=*
   logging.pattern.level=%5p [${spring.application.name:},traceId=%X{traceId},spanId=%X{spanId}]
   ```

5. **Run UserService:**
   ```bash
   ./mvnw spring-boot:run
   ```

---

### **🛠️ Part 3: Create OrderService**

1. **Generate the project at [start.spring.io](https://start.spring.io):**
   - Group: `com.microservices`
   - Artifact: `order-service`
   - Dependencies:
     - Spring Web
     - Spring Boot Actuator
     - Spring Reactive Web

2. **Add to `pom.xml`:**
   Same as UserService:
   - `micrometer-tracing-bridge-brave`
   - `zipkin-reporter-brave`

3. **Create the controller:**
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

4. **Add WebClient bean to `OrderServiceApplication.java`:**
   ```java
   @Bean
   public WebClient.Builder webClientBuilder() {
       return WebClient.builder();
   }
   ```

5. **Add `application.properties`:**
   ```properties
   server.port=8082
   spring.application.name=order-service
   management.tracing.sampling.probability=1.0
   management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans
   management.endpoints.web.exposure.include=*
   logging.pattern.level=%5p [${spring.application.name:},traceId=%X{traceId},spanId=%X{spanId}]
   ```

6. **Run OrderService:**
   ```bash
   ./mvnw spring-boot:run
   ```

---

### **📊 Part 4: Visualize with Zipkin**

1. **Run Zipkin using Docker:**
   ```bash
   docker run -d -p 9411:9411 openzipkin/zipkin
   ```

2. **Visit**: [http://localhost:9411](http://localhost:9411)

3. **Trigger request:**
   ```bash
   curl http://localhost:8082/orders
   ```

4. **Go to Zipkin UI**, click “Run Query”, and see the trace flow from `OrderService → UserService`.

---

### ✅ Conclusion

You have successfully:
- Created 2 microservices with tracing.
- Used Micrometer Tracing (Brave backend).
- Visualized traces via Zipkin.

This replaces the older Sleuth approach and works seamlessly in Spring Boot 3.4.1.

