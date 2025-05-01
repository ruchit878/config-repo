# **Lab 11: Implement Routing Rules and Custom Filters Using Spring Cloud Gateway (Spring Boot 3.4.5)**

## **Objective**
Learn how to configure **Spring Cloud Gateway** (on **Spring Boot 3.4.5**) to route requests to various microservices. Implement custom filters to intercept requests/responses, utilize route predicates for dynamic routing, and monitor the gateway with actuator endpoints.

---

## **Lab Steps**

### **Part 1: Setting Up the API Gateway**

1. **Generate a new Spring Boot project for `ApiGateway`.**
   - Visit [https://start.spring.io/](https://start.spring.io/)
   - Configure:
     - **Spring Boot Version**: **3.4.5**
     - **Group Id**: `com.microservices`
     - **Artifact Id**: `api-gateway`
     - **Name**: `ApiGateway`
     - **Dependencies**:
       - Spring Cloud Gateway
       - Spring Boot Actuator

2. **Import the project into your IDE.**

3. **Enable Spring Cloud Gateway.**
   - In `ApiGatewayApplication.java`:
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

4. **Configure default routes in `application.properties`.**
   ```properties
   server.port=8080

   spring.cloud.gateway.routes[0].id=user-service
   spring.cloud.gateway.routes[0].uri=http://localhost:8081
   spring.cloud.gateway.routes[0].predicates[0]=Path=/users/**
   spring.cloud.gateway.routes[0].predicates[1]=Query=username
   spring.cloud.gateway.routes[0].filters[0]=AddRequestHeader=X-User-Header, UserServiceHeader

   spring.cloud.gateway.routes[1].id=product-service
   spring.cloud.gateway.routes[1].uri=http://localhost:8082
   spring.cloud.gateway.routes[1].predicates[0]=Path=/products/**
   spring.cloud.gateway.routes[1].filters[0]=CustomResponseFilter

   management.endpoints.web.exposure.include=*
   management.endpoint.gateway.enabled=true
   management.endpoints.web.base-path=/actuator
   ```

5. **Run the `ApiGateway` application.**
   ```bash
   mvn spring-boot:run
   ```

6. **Test basic routing.**
   - Ensure `UserService` and `ProductService` are running.
   - Access:
     - `http://localhost:8080/users?username=admin` → Routed to UserService
     - `http://localhost:8080/products` → Routed to ProductService

---

### **Part 2: Setting Up Microservices**

7. **Create `UserService`.**
   - Spring Boot project with Spring Web dependency.
   - `application.properties`:
     ```properties
     server.port=8081
     ```
   - `UserController.java`:
     ```java
     package com.microservices.userservice;

     import org.springframework.web.bind.annotation.*;

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
   - Spring Boot project with Spring Web dependency.
   - `application.properties`:
     ```properties
     server.port=8082
     ```
   - `ProductController.java`:
     ```java
     package com.microservices.productservice;

     import org.springframework.web.bind.annotation.*;

     @RestController
     public class ProductController {
         @GetMapping("/products")
         public String getProducts() {
             return "List of products from ProductService";
         }
     }
     ```

---

### **Part 3: Adding Global Filters**

9. **Create a global logging filter.**
   - `LoggingFilter.java`:
     ```java
     package com.microservices.apigateway;

     import org.slf4j.Logger;
     import org.slf4j.LoggerFactory;
     import org.springframework.cloud.gateway.filter.GlobalFilter;
     import org.springframework.core.annotation.Order;
     import org.springframework.stereotype.Component;
     import reactor.core.publisher.Mono;

     @Component
     @Order(1)
     public class LoggingFilter implements GlobalFilter {
         private static final Logger logger = LoggerFactory.getLogger(LoggingFilter.class);

         @Override
         public Mono<Void> filter(org.springframework.web.server.ServerWebExchange exchange,
                                  org.springframework.cloud.gateway.filter.GatewayFilterChain chain) {
             logger.info("Incoming request: " + exchange.getRequest().getURI());
             return chain.filter(exchange).then(Mono.fromRunnable(() ->
                 logger.info("Outgoing response: " + exchange.getResponse().getStatusCode())));
         }
     }
     ```

---

### **Part 4: Custom Response Filter**

10. **Create a response filter for product-service.**
   - `CustomResponseFilter.java`:
     ```java
     package com.microservices.apigateway;

     import org.springframework.cloud.gateway.filter.GatewayFilter;
     import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
     import org.springframework.stereotype.Component;
     import reactor.core.publisher.Mono;

     @Component
     public class CustomResponseFilter extends AbstractGatewayFilterFactory<Object> {
         @Override
         public GatewayFilter apply(Object config) {
             return (exchange, chain) -> chain.filter(exchange).then(Mono.fromRunnable(() ->
                 exchange.getResponse().getHeaders().add("X-Response-Header", "ModifiedResponse")));
         }
     }
     ```

---

### **Part 5: Monitoring via Actuator**

11. **Access actuator endpoints.**
   - Routes:
     - [http://localhost:8080/actuator/gateway/routes](http://localhost:8080/actuator/gateway/routes)
     - [http://localhost:8080/actuator/gateway/globalfilters](http://localhost:8080/actuator/gateway/globalfilters)
     - [http://localhost:8080/actuator/gateway/routefilters](http://localhost:8080/actuator/gateway/routefilters)

---

## **Conclusion**

You’ve now implemented a fully working **Spring Cloud Gateway** system with:
- Custom headers
- Global and per-route filters
- Predicate-based routing
- Actuator-based monitoring

Use this gateway pattern to centralize routing and cross-cutting logic in your microservice architecture.
