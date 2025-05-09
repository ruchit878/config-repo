# **Lab 11: Implement Routing Rules and Custom Filters Using Spring Cloud Gateway (Spring Boot 3.4.5)**

> **What Is Spring Cloud Gateway?**  
> **Spring Cloud Gateway** is a lightweight, reactive API‑gateway framework. In this lab it fronts two tiny services, adds headers, logs requests, and exposes actuator metrics.

## Installing Prerequisites

| Tool | One‑line install (Windows → winget · macOS → brew · Ubuntu → apt) | Purpose in this lab |
|------|-------------------------------------------------------------------|---------------------|
| **Java 17 JDK** | `winget install --id=EclipseAdoptium.Temurin.17.JDK` | Runs all Spring Boot apps |
| **Maven 3.9+** | `winget install Apache.Maven` | Builds & runs projects (use **mvnw** if generated) |
| **Git** | `winget install Git.Git` | Optional version control |
| **IDE** | IntelliJ IDEA Community or VS Code (Java Extension Pack) | Edit and run code |
| **curl / Postman** | `winget install curl` | Test HTTP endpoints |

> **Troubleshooting – `JAVA_HOME` not found?**  
> Re‑open the terminal or `echo %JAVA_HOME%` / `echo $JAVA_HOME` to confirm. Add it if missing.

---

## Objective
Configure **Spring Cloud Gateway** to route requests, add custom filters, and monitor everything with Actuator endpoints.

---

## Lab Steps

### Part 1: Setting Up the **ApiGateway**

#### 1. Create Project

| Setting | Value |
|---------|-------|
| **Folder** | `C:\Projects\Lab11\api-gateway` |
| **Spring Initializr** | <https://start.spring.io> |
| **Group / Artifact / Name** | `com.microservices` / `api-gateway` / `api-gateway` |
| **Dependencies** | **Spring Cloud Gateway**, **Spring Boot Actuator** |

*Download* the generated ZIP and unzip it into the folder shown above.

#### 2. Import into IDE  
*IntelliJ*: **File → Open…** select the folder.  

> **No manual POM edits needed!** Both dependencies come pre‑configured, and `spring-boot-starter-webflux` plus `spring-boot-starter-test` are added transitively.

#### 3. Create the main application class  
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

#### 4. Configure default routes  
**Path:** `api-gateway/src/main/resources/application.properties`  
```properties
spring.application.name=api-gateway
server.port=8080

# Route 1 – user-service
spring.cloud.gateway.routes[0].id=user-service
spring.cloud.gateway.routes[0].uri=http://localhost:8081
spring.cloud.gateway.routes[0].predicates[0]=Path=/users/**
spring.cloud.gateway.routes[0].predicates[1]=Query=username
spring.cloud.gateway.routes[0].filters[0]=AddRequestHeader=X-User-Header, UserServiceHeader

# Route 2 – product-service (time‑based)
spring.cloud.gateway.routes[1].id=product-service
spring.cloud.gateway.routes[1].uri=http://localhost:8082
spring.cloud.gateway.routes[1].predicates[0]=Path=/products/**
spring.cloud.gateway.routes[1].predicates[1]=After=2024-01-01T00:00:00Z
spring.cloud.gateway.routes[1].filters[0]=AddResponseHeader=X-Product-Header, ProductServiceHeader

# Actuator
management.endpoints.web.exposure.include=routes,filters
```

#### 5. Run the gateway  
*IDE*: right‑click `ApiGatewayApplication` → **Run**.  
*CLI*:
```bash
mvn spring-boot:run
```
Expected:
```text
...Started ApiGatewayApplication...
```

#### 6. Smoke‑test routing (run PowerShell as Administrator)  
```bash
# Call the user route with required query parameter
curl "http://localhost:8080/users?username=admin"
# Call the product route
curl "http://localhost:8080/products"
```
Expected:
```text
Users fetched by: admin | Header: UserServiceHeader
List of products from ProductService
```

---

### Part 2: Setting Up **UserService**

#### 7. Create Project

| Setting | Value |
|---------|-------|
| **Folder** | `C:\Projects\Lab11\user-service` |
| **Spring Initializr** | <https://start.spring.io> |
| **Group / Artifact / Name** | `com.microservices` / `user-service` / `user-service` |
| **Dependencies** | **Spring Web**, **Spring Boot Actuator** |

#### 8. Set port & controller  
`src/main/resources/application.properties`
```properties
server.port=8081
```
`src/main/java/.../UserController.java`
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

#### 9. Run UserService  
```bash
mvn spring-boot:run
```

---

### Part 3: Setting Up **ProductService**

#### 10. Create Project

| Setting | Value |
|---------|-------|
| **Folder** | `C:\Projects\Lab11\product-service` |
| **Spring Initializr** | <https://start.spring.io> |
| **Group / Artifact / Name** | `com.microservices` / `product-service` / `product-service` |
| **Dependencies** | **Spring Web**, **Spring Boot Actuator** |

#### 11. Set port & controller  
`src/main/resources/application.properties`
```properties
server.port=8082
```
`src/main/java/.../ProductController.java`
```java
@RestController
public class ProductController {
    @GetMapping("/products")
    public String getProducts() {
        return "List of products from ProductService";
    }
}
```

#### 12. Run ProductService  
```bash
mvn spring-boot:run
```

---

### Part 4: Global & Custom Filters (Gateway)

13. **Global logging filter**  
`api-gateway/src/main/java/.../LoggingFilter.java`
```java
@Component
@Order(1)
public class LoggingFilter implements GlobalFilter {
    private static final Logger log = LoggerFactory.getLogger(LoggingFilter.class);
    public Mono<Void> filter(ServerWebExchange ex, GatewayFilterChain chain) {
        log.info("Incoming: {}", ex.getRequest().getURI());
        return chain.filter(ex).then(Mono.fromRunnable(() ->
            log.info("Outgoing: {}", ex.getResponse().getStatusCode())));
    }
}
```
Restart the gateway, then run:
```bash
curl "http://localhost:8080/users?username=test"
```
You should see two log lines (`Incoming:` and `Outgoing:`) in the gateway console.

14. **Custom response filter**  
`api-gateway/src/main/java/.../CustomResponseFilter.java`
```java
@Component
public class CustomResponseFilter
        extends AbstractGatewayFilterFactory<Object> {
    @Override
    public GatewayFilter apply(Object cfg) {
        return (ex, chain) -> chain.filter(ex)
            .then(Mono.fromRunnable(() ->
                ex.getResponse().getHeaders()
                  .add("X-Response-Header","ModifiedResponse")));
    }
}
```
Refer to it in the `product-service` route **or** keep `AddResponseHeader`.

15. Call `/products` and check header via:
```bash
curl -I "http://localhost:8080/products"
```

---

### Part 5: Predicate Rules Recap

* `/users` requires `?username=` query param.  
* `/products` route activates only **after** 2024‑01‑01T00:00:00Z.

---

### Part 6: Monitoring with Actuator (run PowerShell as Administrator)

```bash
curl "http://localhost:8080/actuator/routes"
curl "http://localhost:8080/actuator/filters"
```
You should see JSON for each route and filter.

---

## Conclusion 🎉
You now have:

* A reactive **API Gateway** (Spring Boot 3.4.5)  
* Query‑ and time‑based predicates  
* Global & custom filters  
* Actuator visibility  

Next step: connect Eureka for dynamic service registration and see routes appear automatically!
