# **Lab 11: Implement Routing Rules and Custom Filters Using Spring Cloud Gateway (Spring Boot 3.4.5)**

> **What Is Spring Cloud Gateway?**  
> **Spring Cloud Gateway** is a lightweight, reactive API-gateway framework. In this lab it fronts two tiny services, adds headers, logs requests, and exposes actuator metrics.

## Installing Prerequisites

| Tool | One-line install (Windows → winget · macOS → brew · Ubuntu → apt) | Purpose in this lab |
|------|-------------------------------------------------------------------|---------------------|
| **Java 17 JDK** | `winget install --id=EclipseAdoptium.Temurin.17.JDK`<br>`brew install openjdk@17`<br>`sudo apt-get install openjdk-17-jdk` | Runs all Spring Boot apps |
| **Maven 3.9+** | `winget install Apache.Maven`<br>`brew install maven`<br>`sudo apt-get install maven` | Builds & runs projects (use **mvnw** if already generated) |
| **Git** | `winget install Git.Git`<br>`brew install git`<br>`sudo apt-get install git` | Optional version control |
| **IDE** | IntelliJ IDEA Community or VS Code (Java Extension Pack) | Edit and run code |
| **curl or Postman** | Pre-installed on macOS/Linux · `winget install curl` | Test HTTP endpoints |

> **Troubleshooting – JAVA_HOME not found?**  
> Re-open the terminal or `echo %JAVA_HOME%` / `echo $JAVA_HOME` to confirm the path. Add it if missing.

---

## Objective
Configure **Spring Cloud Gateway** to route requests, add custom filters, and monitor everything with Actuator endpoints.

---

## Lab Steps

### Part 1: Setting Up the API Gateway

1. **Generate a new Spring Boot project for `ApiGateway`.**  
   Visit <https://start.spring.io/> and fill these fields: Spring Boot `3.4.5`, Group `com.microservices`, Artifact `api-gateway`, **Dependencies**: *Spring Cloud Gateway*, *Spring Boot Actuator*.  
   *Download the ZIP* and **unzip into** `~/labs/api-gateway`.

2. **Import the project into your IDE.**  
   *IntelliJ*: **File → Open…** select the folder.  
   *VS Code*: **File → Open Folder…** then `Ctrl+Shift+P › Java: Import Maven Projects`.

   **Dependency tidy-up (version hygiene).**  
   Open `pom.xml` and **ensure no manual `<version>` tags** for Spring Cloud libs; rely on the BOM.

3. **Replace the `<dependencies>` block.**  
   **Path:** `api-gateway/pom.xml`  
   ```xml
   <!-- ✅ keep existing top-level tags unchanged -->
   <dependencies>
       <!-- Spring Cloud Gateway (Reactive) -->
       <dependency>
           <groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-starter-gateway</artifactId>
       </dependency>

       <!-- WebFlux runtime -->
       <dependency>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-starter-webflux</artifactId>
       </dependency>

       <!-- Actuator endpoints -->
       <dependency>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-starter-actuator</artifactId>
       </dependency>

       <!-- Tests -->
       <dependency>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-starter-test</artifactId>
           <scope>test</scope>
       </dependency>
   </dependencies>
   ```

4. **Create the main application class.**  
   **Path:** `api-gateway/src/main/java/com/microservices/apigateway/ApiGatewayApplication.java`  
   ```java
   package com.microservices.apigateway;

   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.autoconfigure.SpringBootApplication;

   @SpringBootApplication
   public class ApiGatewayApplication {
       public static void main(String[] args) {
           SpringApplication.run(ApiGatewayApplication.class, args);
       }
   }
   ```

5. **Add route configuration.**  
   **Path:** `api-gateway/src/main/resources/application.properties`  
   ```properties
   # Service identity
   spring.application.name=api-gateway
   server.port=8080

   # Route 1 – user-service
   spring.cloud.gateway.routes[0].id=user-service
   spring.cloud.gateway.routes[0].uri=http://localhost:8081
   spring.cloud.gateway.routes[0].predicates[0]=Path=/users/**
   spring.cloud.gateway.routes[0].predicates[1]=Query=username
   spring.cloud.gateway.routes[0].filters[0]=AddRequestHeader=X-User-Header, UserServiceHeader

   # Route 2 – product-service
   spring.cloud.gateway.routes[1].id=product-service
   spring.cloud.gateway.routes[1].uri=http://localhost:8082
   spring.cloud.gateway.routes[1].predicates[0]=Path=/products/**
   spring.cloud.gateway.routes[1].filters[0]=AddResponseHeader=X-Product-Header, ProductServiceHeader

   # Actuator endpoints
   management.endpoints.web.exposure.include=*
   management.endpoint.gateway.enabled=true
   management.endpoints.web.base-path=/actuator
   ```

6. **Run the gateway.**  
   *IDE*: right-click `ApiGatewayApplication` → **Run**.  
   *CLI*:  
   ```bash
   ./mvnw spring-boot:run
   ```
   **Expected output**  
   ```text
   ...Started ApiGatewayApplication in 4.5 seconds (JVM running for 5.2)
   ```

7. **Test routing.**  
   ```bash
   curl "http://localhost:8080/users?username=admin"
   curl "http://localhost:8080/products"
   ```
   **Expected (truncated)**
   ```text
   Users fetched by: admin | Header: UserServiceHeader
   List of products from ProductService
   ```

> **Troubleshooting – Port 8080 already in use?**  
> Stop the old process (`Ctrl+C`) or change `server.port`.

---

### Part 2: Setting Up Microservices

8. **Create `UserService`.**  
   Generate a Spring Boot project with *Spring Web*, unzip to `~/labs/user-service`, and set `server.port=8081` in `application.properties`.

   **Controller – Path:** `user-service/src/main/java/.../UserController.java`  
   ```java
   @RestController
   public class UserController {
       @GetMapping("/users")
       public String getUsers(@RequestParam String username,
                              @RequestHeader(value="X-User-Header",required=false) String userHeader) {
           System.out.println("Received X-User-Header: " + userHeader);
           return "Users fetched by: " + username + " | Header: " + userHeader;
       }
   }
   ```

9. **Create `ProductService`.**  
   Repeat the wizard; unzip to `~/labs/product-service`, set `server.port=8082`.

   **Controller – Path:** `product-service/src/main/java/.../ProductController.java`  
   ```java
   @RestController
   public class ProductController {
       @GetMapping("/products")
       public String getProducts() {
           return "List of products from ProductService";
       }
   }
   ```

10. **Run both services.**  
    *IDE*: Run each main class.  
    *CLI* (in each folder):  
    ```bash
    ./mvnw spring-boot:run
    ```
    **Expected**  
    `UserService` listening on **8081**; `ProductService` on **8082**.

> **Troubleshooting – “Address already in use”**  
> Each service must have a unique port value.

---

### Part 3: Creating and Testing Filters

11. **Create a global logging filter.**  
    **Path:** `api-gateway/src/main/java/.../LoggingFilter.java`  
    ```java
    @Component
    @Order(1)
    public class LoggingFilter implements GlobalFilter {
        private static final Logger log = LoggerFactory.getLogger(LoggingFilter.class);
        public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
            log.info("Incoming: {}", exchange.getRequest().getURI());
            return chain.filter(exchange)
                        .then(Mono.fromRunnable(() ->
                            log.info("Outgoing: {}", exchange.getResponse().getStatusCode())));
        }
    }
    ```

12. **Restart the gateway and hit `/users`.**  
    Observe two log lines (request + response) in the console.

13. **Verify header propagation.**  
    The gateway adds `X-User-Header`; `UserService` prints it.  
    **Expected console line (UserService):**  
    ```text
    Received X-User-Header: UserServiceHeader
    ```

14. **(Optional) Create a custom response filter.**  
    **Path:** `api-gateway/src/main/java/.../CustomResponseFilter.java`  
    ```java
    @Component
    public class CustomResponseFilter
            extends AbstractGatewayFilterFactory<Object> {
        @Override
        public GatewayFilter apply(Object config) {
            return (exchange, chain) -> chain.filter(exchange)
                .then(Mono.fromRunnable(() ->
                    exchange.getResponse().getHeaders()
                            .add("X-Response-Header", "ModifiedResponse")));
        }
    }
    ```

15. **Bind the custom filter** by referencing it in the `product-service` route or keep `AddResponseHeader`.

16. **Call `/products`** and check the extra header with `curl -I`.

---

### Part 4: Advanced Routing with Predicates

17. **Call `/users?username=test`** – should succeed (HTTP 200).  
    **Expected body:** `Users fetched by: test | Header: UserServiceHeader`.

18. **Call `/users` without the query param** – should fail (HTTP 404).

19. **(Optional) Add a time-based route.**  
    Append to `application.properties` in `api-gateway`.  
    ```properties
    spring.cloud.gateway.routes[2].id=time-test
    spring.cloud.gateway.routes[2].uri=http://localhost:8083
    spring.cloud.gateway.routes[2].predicates[0]=After=2024-01-01T00:00:00Z
    spring.cloud.gateway.routes[2].predicates[1]=Path=/time/**
    ```

20. **Spin up a dummy service on port 8083** or watch the gateway return `503` if the predicate matches but no service exists.

---

### Part 5: Monitoring the Gateway with Actuator

21. **Actuator dependency is already present.**

22. **Exposed endpoints configured in `application.properties`.**

23. **Hit the endpoints.**  
    ```bash
    curl "http://localhost:8080/actuator/gateway/routes"
    curl "http://localhost:8080/actuator/gateway/globalfilters"
    ```
    **Expected JSON snippet**  
    ```json
    [{"id":"user-service","uri":"http://localhost:8081", ...}]
    ```

24. **Verify all routes and filters appear** and match earlier definitions.

---

## Scaling Example (Same Pattern, New Port)

Need another microservice? Duplicate `UserService`, change `artifactId`, and set `server.port=8084`. Adjust gateway predicates exactly as you did before.

---

## Conclusion 🎉
You:

* Built an **API Gateway** with reactive routing.  
* Added global and route-level filters plus custom headers.  
* Observed logs and live config via **Actuator**.  

**Next experiment:** add **rate-limiting** (`RequestRateLimiter` filter) to protect your services.
