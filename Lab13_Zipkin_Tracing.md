
# **Lab 13: Use Zipkin to Collect, Analyze, and Visualize Traces in Your Microservices Architecture (Spring Boot 3.4.5)**

## **Objective**
Learn how to integrate **Spring Cloud Sleuth** with **Zipkin** to collect and visualize distributed traces in a **Spring Boot 3.4.5** microservices architecture. You will configure two services (`UserService` and `OrderService`) to send trace data to a local Zipkin server and analyze end‑to‑end request flows via the Zipkin dashboard.

---

## **Lab Steps**

### **Part 1: Installing and Running Zipkin**

1. **Ensure Java is installed (for running Zipkin).**
   ```bash
   java -version
   ```
   If the command is not found, install **JDK 17 +** (e.g., from [Adoptium](https://adoptium.net/)).

2. **Download the Zipkin server.**
   * Visit the [Zipkin Quickstart](https://zipkin.io/pages/quickstart) page and download the latest `zipkin-server-*-exec.jar`.
   * Save it somewhere handy, e.g. `~/zipkin` (macOS / Linux) or `C:\Zipkin` (Windows).

3. **Run Zipkin locally.**
   ```bash
   java -jar zipkin-server-*-exec.jar
   ```
   Zipkin listens on **port 9411** by default.

4. **Verify Zipkin is up.**  
   Open <http://localhost:9411> — the Zipkin UI should load.

---

### **Part 2: Configuring the Producer Microservice (`UserService`)**

5. **Generate the project** on *Spring Initializr*:

| Option | Value |
|--------|-------|
| **Spring Boot** | `3.4.5` |
| **Group** | `com.microservices` |
| **Artifact** | `user-service` |
| **Name** | `UserService` |
| **Dependencies** | *Spring Web*, *Spring Boot Actuator*, *Spring Cloud Sleuth* |

Unzip the archive into a folder named **`UserService`**.

6. **Import `UserService` into your IDE.**

7. **➕ Add Sleuth / Zipkin dependencies.**  
   Open **`pom.xml`** and paste these inside `<dependencies>`:

   ```xml
   <!-- Spring Cloud Sleuth sends spans to Zipkin -->
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-sleuth-zipkin</artifactId>
       <version>3.1.11</version>
   </dependency>

   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-sleuth-core</artifactId>
       <version>2.2.8.RELEASE</version>
   </dependency>
   ```

8. **Configure tracing, sampling, and span logging.**

   `src/main/resources/application.properties`:
   ```properties
   server.port=8081
   spring.application.name=user-service

   # — Send all spans to the local Zipkin collector
   spring.zipkin.base-url=http://localhost:9411
   spring.sleuth.sampler.probability=1.0   # 100 % sampling

   # — Add traceId / spanId to every log line (same pattern as Lab 12)
   logging.pattern.level=%5p [${spring.application.name},traceId=%X{traceId},spanId=%X{spanId}]
   ```

9. **Create a simple REST controller.**

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

10. **Run `UserService`.**
    ```bash
    ./mvnw spring-boot:run
    ```
    *Observe the console* — every log line now prints its `traceId` and `spanId`.

11. **Smoke‑test `/users`.**  
    <http://localhost:8081/users> should return the user list string.

---

### **Part 3: Configuring the Consumer Microservice (`OrderService`)**

12. **Generate the project**:

| Option | Value |
|--------|-------|
| **Artifact** | `order-service` |
| **Dependencies** | *Spring Web*, *Spring Boot Actuator*, *Spring Cloud Sleuth*, *Spring WebClient* |

Unzip into **`OrderService`**.

13. **Import `OrderService` into your IDE.**

14. **➕ Add Sleuth / Zipkin / WebFlux dependencies.**

   Edit **`pom.xml`**:

   ```xml
   <!-- Spring Cloud Sleuth → Zipkin -->
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-sleuth-zipkin</artifactId>
       <version>3.1.11</version>
   </dependency>

   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-sleuth-core</artifactId>
       <version>2.2.8.RELEASE</version>
   </dependency>

   <!-- WebClient lives in WebFlux -->
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-webflux</artifactId>
   </dependency>
   ```

15. **Configure tracing, sampling, and span logging.**

   `src/main/resources/application.properties`:
   ```properties
   server.port=8082
   spring.application.name=order-service

   # — Zipkin collector
   spring.zipkin.base-url=http://localhost:9411
   spring.sleuth.sampler.probability=1.0

   # — Print trace identifiers in logs
   logging.pattern.level=%5p [${spring.application.name},traceId=%X{traceId},spanId=%X{spanId}]
   ```

16. **Create the REST controller.**

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

17. **Register a `WebClient.Builder` bean.**

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

18. **Run `OrderService`.**
    ```bash
    ./mvnw spring-boot:run
    ```
    *Observe the logs* — the same `traceId` that starts in `OrderService` propagates to `UserService`, each step with its own `spanId`.

19. **Test `/orders`.**  
    <http://localhost:8082/orders> should echo user data from the upstream call.

---

### **Part 4: Visualizing Traces in Zipkin**

20. **Generate some traffic.**  
    Hit `/orders` a few times (browser or `curl`).

21. **Open the Zipkin UI.**  
    <http://localhost:9411>

22. **Find your traces.**
    * Click **Find Traces**.  
    * Look for rows containing `order-service` and `user-service`.

23. **Inspect a trace.**
    * Click a row to expand details.  
    * Verify the span tree:  
      `OrderService (SERVER)` → `UserService (CLIENT)` → `UserService (SERVER)`.

---

## **Optional Exercises**

1. **Add a third service** (e.g. `ProductService`) to create a three‑hop trace.
2. **Introduce latency or exceptions** in `UserService` and watch the impact in Zipkin.
3. **Experiment with sampling** by lowering `spring.sleuth.sampler.probability`.

---

## **Conclusion**

You have:

* Installed **Zipkin** and verified it runs on <http://localhost:9411>.
* Wired two Spring Boot 3.4.5 services with **Spring Cloud Sleuth**, so every request is assigned a `traceId` and one or more `spanId`s.
* Added a logging pattern so the **console output** is trace‑aware, making it easy to correlate logs with the Zipkin UI.
* Visualized and analyzed distributed traces, gaining insight into request flow and latency across your microservices.
