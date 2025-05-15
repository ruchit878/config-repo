
# **Lab 11 — Routing & Custom Filters with Spring Cloud Gateway (Spring Boot 3.4.5)**

> **Spring Cloud Gateway** is the reactive API‑gateway module in Spring Cloud.  
> **Use only:**  
> ```xml
> <dependency>
>   <groupId>org.springframework.cloud</groupId>
>   <artifactId>spring-cloud-starter-gateway</artifactId>
> </dependency>
> ```  
> Do **NOT** use the deprecated *spring-cloud‑starter‑gateway‑mvc* or other variants.

---

## 0 · Prerequisites

| Tool | One‑line install (Windows → winget · macOS → brew · Ubuntu → apt) | Purpose |
|------|-------------------------------------------------------------------|---------|
| Java 17 JDK | `winget install --id=EclipseAdoptium.Temurin.17.JDK` | Run Spring Boot 3.4.x apps |
| Maven 3.9+ | `winget install Apache.Maven` | Build & run (`mvn spring-boot:run`) |
| Git | `winget install Git.Git` | (Optional) version control |
| IDE | IntelliJ IDEA Community / VS Code | Edit & debug |
| curl | `winget install curl` | Test HTTP endpoints |

---

## 1 · Create & Run **UserService**

| Setting | Value |
|---------|-------|
| Folder | `C:\Projects\Lab11\user-service` |
| Spring Initializr | <https://start.spring.io> |
| Group / Artifact / Name | `com.microservices` / `user-service` / `user-service` |
| Dependencies | **Spring Web**, **Spring Boot Actuator** |

`src/main/resources/application.properties`
```properties
server.port=8081
```

`UserController.java`
```java
@RestController
public class UserController {
  @GetMapping("/users")
  public String getUsers(@RequestParam String username,
                         @RequestHeader(value="X-User-Header", required=false) String hdr) {
    System.out.println("Received X-User-Header: " + hdr);
    return "Users fetched by: " + username + " | Header: " + hdr;
  }
}
```

Run:
```powershell
cd C:\Projects\Lab11\user-service
mvn spring-boot:run
```

---

## 2 · Create & Run **ProductService**

| Setting | Value |
|---------|-------|
| Folder | `C:\Projects\Lab11\product-service` |
| Spring Initializr | <https://start.spring.io> |
| Group / Artifact / Name | `com.microservices` / `product-service` / `product-service` |
| Dependencies | **Spring Web**, **Spring Boot Actuator** |

`application.properties`
```properties
server.port=8082
```

`ProductController.java`
```java
@RestController
public class ProductController {
  @GetMapping("/products")
  public String list() {
    return "List of products from ProductService";
  }
}
```

Run:
```powershell
cd C:\Projects\Lab11\product-service
mvn spring-boot:run
```

---

## 3 · Create **ApiGateway**

| Setting | Value |
|---------|-------|
| Folder | `C:\Projects\Lab11\api-gateway` |
| Spring Initializr | <https://start.spring.io> |
| Group / Artifact / Name | `com.microservices` / `api-gateway` / `api-gateway` |
| Dependencies | **Spring Cloud Gateway**, **Spring Boot Actuator** |

### 3.1 · Gateway main class  
`ApiGatewayApplication.java`
```java
@SpringBootApplication
public class ApiGatewayApplication {
  public static void main(String[] args) {
    SpringApplication.run(ApiGatewayApplication.class, args);
  }
}
```

### 3.2 · Route configuration  
`application.properties`
```properties
spring.application.name=api-gateway
server.port=8080

# /users route – requires ?username=
spring.cloud.gateway.routes[0].id=user-service
spring.cloud.gateway.routes[0].uri=http://localhost:8081
spring.cloud.gateway.routes[0].predicates[0]=Path=/users/**
spring.cloud.gateway.routes[0].predicates[1]=Query=username
spring.cloud.gateway.routes[0].filters[0]=AddRequestHeader=X-User-Header, UserServiceHeader

# /products route – active after 2024‑01‑01
spring.cloud.gateway.routes[1].id=product-service
spring.cloud.gateway.routes[1].uri=http://localhost:8082
spring.cloud.gateway.routes[1].predicates[0]=Path=/products/**
spring.cloud.gateway.routes[1].predicates[1]=After=2024-01-01T00:00:00Z
spring.cloud.gateway.routes[1].filters[0]=AddResponseHeader=X-Product-Header, ProductServiceHeader

# Actuator
management.endpoints.web.exposure.include=gateway
```

### 3.3 · Global logging filter *(optional)*
`LoggingFilter.java`
```java
@Component @Order(1)
public class LoggingFilter implements GlobalFilter {
  private static final Logger log = LoggerFactory.getLogger(LoggingFilter.class);
  public Mono<Void> filter(ServerWebExchange ex, GatewayFilterChain chain) {
    log.info("▶ Incoming {}", ex.getRequest().getURI());
    return chain.filter(ex).then(Mono.fromRunnable(() ->
      log.info("◀ Outgoing {}", ex.getResponse().getStatusCode())));
  }
}
```

Run:
```powershell
cd C:\Projects\Lab11\api-gateway
mvn spring-boot:run
```

---

## 4 · Smoke Test

```powershell
# Check headers (bypass PowerShell alias)
curl.exe --head http://localhost:8080/products

curl.exe http://localhost:8080/users?username=alice
```

Expected:
```
X-Product-Header: ProductServiceHeader
Users fetched by: alice | Header: UserServiceHeader
```

---

## 5 · Monitor with Actuator

```powershell
curl.exe http://localhost:8080/actuator/gateway/routes
curl.exe http://localhost:8080/actuator/gateway/globalfilters
```

---

## 6 · Troubleshooting

| Issue | Fix |
|-------|-----|
| `Uri:` prompt | Use `curl.exe` or `curl --head` |
| 404 on `/products` | Remove or adjust the `After=` predicate |
| Port conflict | Change `server.port` in properties |

---

🎉 **Done!**
