# **Lab 11: Implement Routing Rules and Custom Filters Using Spring Cloud Gateway (Spring Boot 3.4.5)**

## **Objective**
Learn to configure **Spring Cloud Gateway** (Spring Boot 3.4.5) to route and filter HTTP requests between microservices, log traffic, inject headers, and monitor the gateway with Actuator.

---

## **Date**
May 01, 2025

---

## **Project Structure**

- **ApiGateway** (Port: 8080)
- **UserService** (Port: 8081)
- **ProductService** (Port: 8082)

---

## **1. Generate Spring Boot Projects**

### **ApiGateway**
- Spring Boot 3.4.5
- Dependencies: `Spring Cloud Gateway`, `Spring Boot Actuator`

### **UserService**
- Spring Boot 3.4.5
- Dependency: `Spring Web`

### **ProductService**
- Spring Boot 3.4.5
- Dependency: `Spring Web`

---

## **2. Update `pom.xml` of ApiGateway**

Use the final cleaned `pom.xml` with:
- Spring Boot Starter Actuator
- Spring Cloud Gateway
- Spring Cloud BOM (2024.0.1)

---

## **3. Configure `application.properties` of ApiGateway**

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

management.endpoints.web.exposure.include=*
management.endpoint.gateway.enabled=true
management.endpoints.web.base-path=/actuator
```

---

## **4. Create LoggingFilter.java (Global Filter)**

```java
@Component
@Order(1)
public class LoggingFilter implements GlobalFilter {
    private static final Logger logger = LoggerFactory.getLogger(LoggingFilter.class);
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        logger.info("Incoming request: " + exchange.getRequest().getURI());
        return chain.filter(exchange).then(Mono.fromRunnable(() ->
            logger.info("Outgoing response: " + exchange.getResponse().getStatusCode())
        ));
    }
}
```

---

## **5. Create CustomResponseFilter.java (Response Header Filter)**

```java
@Component
public class CustomResponseFilter extends AbstractGatewayFilterFactory<Object> {
    @Override
    public GatewayFilter apply(Object config) {
        return (exchange, chain) -> chain.filter(exchange).then(Mono.fromRunnable(() ->
            exchange.getResponse().getHeaders().add("X-Response-Header", "ModifiedResponse")
        ));
    }
}
```

To enable it, add to `application.properties`:
```properties
spring.cloud.gateway.routes[1].filters[0]=CustomResponseFilter
```

---

## **6. UserService Sample Code**

**application.properties**
```properties
server.port=8081
```

**UserController.java**
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

---

## **7. ProductService Sample Code**

**application.properties**
```properties
server.port=8082
```

**ProductController.java**
```java
@RestController
public class ProductController {
    @GetMapping("/products")
    public String getProducts() {
        return "List of products from ProductService";
    }
}
```

---

## **8. Test Your Setup**

| URL | Description |
|-----|-------------|
| `http://localhost:8080/users?username=admin` | Routes to UserService + Injects `X-User-Header` |
| `http://localhost:8080/products` | Routes to ProductService + Appends `X-Response-Header` |
| `http://localhost:8080/actuator/gateway/routes` | Shows current routes |
| `http://localhost:8080/actuator/gateway/globalfilters` | Lists global filters |
| `http://localhost:8080/actuator/gateway/routefilters` | Lists route filters |

---

## **Conclusion**

This lab helps you:
- ✅ Route requests using Spring Cloud Gateway.
- ✅ Add custom headers and modify responses.
- ✅ Implement global and route-specific filters.
- ✅ Monitor using Spring Boot Actuator.

You're now ready to handle dynamic routing and filtering in a microservice environment.
