# **Lab 13: Use Zipkin to Collect, Analyze, and Visualize Traces in Your Microservices Architecture (Spring Boot 3.4.5)**

## **Objective**
Learn how to integrate **Spring Cloud Sleuth** with **Zipkin** to collect and visualize distributed traces in a **Spring Boot 3.4.5** microservices architecture. You will configure two services (`UserService` and `OrderService`) to send trace data to a local Zipkin server and analyze end-to-end request flows via the Zipkin dashboard.

---

## **Lab Steps**

### **Part 1: Installing and Running Zipkin**

1. **Ensure Java is installed (for running Zipkin).**
   - Check Java:
     ```bash
     java -version
     ```
   - If not installed, install **JDK 17** (e.g., from [AdoptOpenJDK](https://adoptium.net/)).

2. **Download the Zipkin Server.**
   - Visit [Zipkin Quickstart](https://zipkin.io/pages/quickstart) and download the latest Zipkin JAR file (e.g., `zipkin-server-2.x.x-exec.jar`).
   - Save it in a folder like `~/zipkin` or `C:\Zipkin`.

3. **Run the Zipkin server locally.**
   - From the directory with the Zipkin JAR:
     ```bash
     java -jar zipkin-server-2.x.x-exec.jar
     ```
   - Zipkin listens on **port 9411** by default.

4. **Verify Zipkin is running.**
   - Open:
     ```
     http://localhost:9411
     ```
   - The Zipkin dashboard should appear.

---

### **Part 2: Configuring the Producer Microservice (`UserService`)**

5. **Generate a new Spring Boot project for `UserService`.**
   - **Spring Boot Version**: **3.4.5**
   - **Group Id**: `com.microservices`
   - **Artifact Id**: `user-service`
   - **Name**: `UserService`
   - **Dependencies**:
     - Spring Web
     - Spring Boot Actuator
     - Spring Cloud Sleuth
   - Extract into a folder named `UserService`.

6. **Import `UserService`** into your IDE.

7. **Configure `UserService` to send traces to Zipkin.**
   - In `src/main/resources/application.properties`:
     ```properties
     server.port=8081

     spring.application.name=user-service
     spring.zipkin.base-url=http://localhost:9411
     spring.sleuth.sampler.probability=1.0
     ```
   - `sampler.probability=1.0` means all requests are traced.

8. **Create a REST controller in `UserService`.**
   - `UserController.java`:
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

9. **Run `UserService`.**
   - From the `UserService` directory:
     ```bash
     ./mvnw spring-boot:run
     ```
   - It listens on **8081**.

10. **Test the `/users` endpoint.**
    - Access:
      ```
      http://localhost:8081/users
      ```
    - Confirm it returns `"List of users from UserService"`.

---

### **Part 3: Configuring the Consumer Microservice (`OrderService`)**

11. **Generate a new Spring Boot project for `OrderService`.**
    - **Artifact Id**: `order-service`
    - **Dependencies**:
      - Spring Web
      - Spring Boot Actuator
      - Spring Cloud Sleuth
      - Spring WebClient
    - Extract into a folder named `OrderService`.

12. **Import `OrderService`** into your IDE.

13. **Configure `OrderService` to send traces to Zipkin.**
    - In `src/main/resources/application.properties`:
      ```properties
      server.port=8082

      spring.application.name=order-service
      spring.zipkin.base-url=http://localhost:9411
      spring.sleuth.sampler.probability=1.0
      ```

14. **Create a REST controller in `OrderService`.**
    - `OrderController.java`:
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

15. **Add WebClient configuration.**
    - In `OrderServiceApplication.java`:
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

16. **Run `OrderService`.**
    - From `OrderService`:
      ```bash
      ./mvnw spring-boot:run
      ```
    - It listens on **8082**.

17. **Test the `/orders` endpoint.**
    - Access:
      ```
      http://localhost:8082/orders
      ```
    - It retrieves data from `UserService` on `8081`.

---

### **Part 4: Visualizing Traces in Zipkin**

18. **Generate traffic between the services.**
    - Hit `http://localhost:8082/orders` multiple times.

19. **Access the Zipkin dashboard.**
    - Go to:
      ```
      http://localhost:9411
      ```
    - You should see the **Zipkin UI**.

20. **Search for traces.**
    - Click **Find Traces** on the Zipkin dashboard.
    - Look for spans involving `order-service` and `user-service`.

21. **Analyze a specific trace.**
    - Click a trace to see details, including:
      - **Spans** (segments of the request)
      - **Timing** (latency)
      - **Service dependencies**.

---

## **Optional Exercises**

1. **Add a third microservice.**
   - E.g., `ProductService` calls `UserService`. See how Zipkin links all three in a single trace.

2. **Simulate service failure or slowdown.**
   - Make `UserService` throw an exception or introduce a delay and observe the trace in Zipkin.

3. **Experiment with sampling rates.**
   - Adjust `spring.sleuth.sampler.probability` to reduce overhead in production scenarios.

---

## **Conclusion**
By finishing this lab, you have:
- **Installed and ran Zipkin** locally on **port 9411**.
- **Configured two microservices** (`UserService`, `OrderService`) with **Spring Cloud Sleuth** to send trace data to Zipkin.
- **Observed** trace data (service calls, latencies, errors) in the **Zipkin dashboard**.
- **Validated** how distributed tracing simplifies root cause analysis and performance tuning across microservices.
