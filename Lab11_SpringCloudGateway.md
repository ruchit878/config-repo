# **Lab 11: Implement Routing Rules and Custom Filters Using Spring Cloud Gateway (Spring Boot 3.4.5)**

## **Objective**
Learn how to configure **Spring Cloud Gateway** (on **Spring Boot 3.4.5**) to route requests to various micro-services. Implement custom filters to intercept requests/responses, utilize route predicates for dynamic routing, and monitor the gateway with actuator endpoints.

---

## **Lab Steps**

### **Part 1: Setting Up the API Gateway**

1. **Generate a new Spring Boot project for `ApiGateway`.**  
   - Go to <https://start.spring.io/>  
   - Use the following configuration:  
     - Spring Boot Version: `3.4.5`  
     - Group Id: `com.microservices`  
     - Artifact Id: `api-gateway`  
     - **Dependencies**: Spring Cloud Gateway, Spring Boot Actuator  

2. **Import the project into your IDE** (e.g., IntelliJ IDEA, VS Code).

   **Sub-step – Update Maven dependencies (replace the old MVC starter):**  
   In `pom.xml`, delete any existing `<dependency>` that references `spring-cloud-starter-gateway-mvc` and replace the whole `<dependencies>` block with:

   ```xml
   <dependencies>
       <!-- ✅ Spring Cloud Gateway (Reactive) -->
       <dependency>
           <groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-starter-gateway</artifactId>
       </dependency>

       <!-- ✅ Explicit WebFlux runtime -->
       <dependency>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-starter-webflux</artifactId>
       </dependency>

       <!-- Optional: Actuator endpoints -->
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

3. **Create the main application class.**  
   File: `src/main/java/com/microservices/apigateway/ApiGatewayApplication.java`

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

4. **Add route configuration in `application.properties`.**  
   File: `src/main/resources/application.properties`

   ```properties
   # ----------------------------------------------------------------------
   # Basic service settings
   # ----------------------------------------------------------------------
   spring.application.name=api-gateway
   server.port=8080

   # ----------------------------------------------------------------------
   # Route 1 – user-service (unchanged)
   # ----------------------------------------------------------------------
   spring.cloud.gateway.routes[0].id=user-service
   spring.cloud.gateway.routes[0].uri=http://localhost:8081
   spring.cloud.gateway.routes[0].predicates[0]=Path=/users/**
   spring.cloud.gateway.routes[0].predicates[1]=Query=username
   spring.cloud.gateway.routes[0].filters[0]=AddRequestHeader=X-User-Header, UserServiceHeader

   # ----------------------------------------------------------------------
   # Route 2 – product-service (filter fixed)
   # ----------------------------------------------------------------------
   spring.cloud.gateway.routes[1].id=product-service
   spring.cloud.gateway.routes[1].uri=http://localhost:8082
   spring.cloud.gateway.routes[1].predicates[0]=Path=/products/**
   # Built-in filter: add a response header instead of a custom filter
   spring.cloud.gateway.routes[1].filters[0]=AddResponseHeader=X-Product-Header, ProductServiceHeader

   # ----------------------------------------------------------------------
   # Actuator / management endpoints
   # ----------------------------------------------------------------------
   management.endpoints.web.exposure.include=*
   management.endpoint.gateway.enabled=true
   management.endpoints.web.base-path=/actuator
   ```

5. **Run the `ApiGateway` application.**

   ```bash
   mvn spring-boot:run
   ```

6. **Test routing.**
   - `http://localhost:8080/users?username=admin`  
   - `http://localhost:8080/products`

---

### **Part 2: Setting Up Microservices**

7. **Create `UserService`.**
   - Spring Boot project with `Spring Web` dependency.  
   - Set port to 8081 in `application.properties`:
     ```properties
     server.port=8081
     ```
   - Create a controller:
     ```java
     @RestController
     public class UserController {
         @GetMapping("/users")
         public String getUsers(@RequestParam String username,
                                @RequestHeader(value = "X-User-Header", required = false) String userHeader) {
             System.out.println("Received X-User-Header: " + userHeader);
             return "Users fetched by: " + username + " | Header: " + userHeader;
         }
     }
     ```

8. **Create `ProductService`.**
   - Spring Boot project with `Spring Web` dependency.  
   - Set port to 8082 in `application.properties`:
     ```properties
     server.port=8082
     ```
   - Create a controller:
     ```java
     @RestController
     public class ProductController {
         @GetMapping("/products")
         public String getProducts() {
             return "List of products from ProductService";
         }
     }
     ```

9. **Run both services.**  
   Ensure `UserService` on port `8081` and `ProductService` on port `8082`.

---

### **Part 3: Creating and Testing Filters**

10. **Create a global logging filter.**  
    File: `LoggingFilter.java`
    ```java
    @Component
    @Order(1)
    public class LoggingFilter implements GlobalFilter {
        private static final Logger logger = LoggerFactory.getLogger(LoggingFilter.class);

        public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
            logger.info("Incoming request: " + exchange.getRequest().getURI());
            return chain.filter(exchange).then(Mono.fromRunnable(() ->
                logger.info("Outgoing response: " + exchange.getResponse().getStatusCode())));
        }
    }
    ```

11. **Run the gateway and inspect the logs.**

12. **Verify that `X-User-Header` is added.**

13. **Confirm that `UserService` prints the header.**

14. **Create a custom response filter (optional).**  
    If you prefer a code-based filter instead of the built-in `AddResponseHeader`, create:

    ```java
    @Component
    public class CustomResponseFilter extends AbstractGatewayFilterFactory<Object> {
        @Override
        public GatewayFilter apply(Object config) {
            return (exchange, chain) -> chain.filter(exchange).then(Mono.fromRunnable(() ->
                exchange.getResponse().getHeaders().add("X-Response-Header", "ModifiedResponse")));
        }
    }
    ```

15. **Bind the response filter to the `product-service` route** (or keep the built-in filter).

16. **Verify that `/products` responses include the new header.**

---

### **Part 4: Advanced Routing with Predicates**

17. **Test `/users?username=test` — it should succeed.**

18. **Accessing `/users` without the query parameter should return 404.**

19. **Add a time-based route (optional).**
    ```properties
    spring.cloud.gateway.routes[2].id=time-test
    spring.cloud.gateway.routes[2].uri=http://localhost:8083
    spring.cloud.gateway.routes[2].predicates[0]=After=2024-01-01T00:00:00Z
    spring.cloud.gateway.routes[2].predicates[1]=Path=/time/**
    ```
20. **Test the time-based routing.**

---

### **Part 5: Monitoring the Gateway with Actuator**

23. **Access these endpoints:**
   - `/actuator/gateway/routes`
   - `/actuator/gateway/globalfilters`
   - `/actuator/gateway/routefilters`

24. **Verify all routes and filters appear as expected.**

---

## ✅ Conclusion
You now have:

- A fully working **API Gateway**  
- Global and route-specific filters  
- Predicate-based routing  
- Actuator-powered visibility  

You’re ready to apply gateway patterns in real-world micro-service systems!
