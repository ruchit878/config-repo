# **Lab 12: Implement Distributed Tracing with Spring Boot 3.4.5 (Micrometer + Brave + Zipkin)**

---

## Installing Prerequisites 🚀
| Tool | One‑line install | Purpose |
|------|-----------------|---------|
| **Java 17 +** | `winget install --silent EclipseAdoptium.Temurin.17.JDK` | Runs Spring Boot 3.4.5 & Zipkin |
| **Maven 3.9 +** | `winget install Apache.Maven` | Builds & runs projects |
| **IDE (IntelliJ / VS Code)** | Download from vendor | Edit & run code |
| **Zipkin executable JAR** | Download `zipkin-server-*‑exec.jar` from Maven Central or <https://zipkin.io/pages/quickstart> | Collects & displays traces |

> **Why Micrometer instead of Sleuth?**  
> Spring Cloud Sleuth was removed in Spring Boot 3.x. Micrometer Tracing (with a Brave bridge) is the official successor. Properties moved from `spring.sleuth.*` ➜ `management.tracing.*`, and Zipkin is now the preferred exporter.

---

## Quick‑Start Topology
```
curl ─▶ OrderService (8082) ─┬─▶ UserService (8081)
                              └─▶ Zipkin (9411 local JAR)
```

---

## Part 1 – Create **UserService**

| Field | Value |
|-------|-------|
| **Project** | **UserService** |
| Group ID | `com.microservices` |
| Artifact ID | `user-service` |
| Spring Boot | `3.4.5` |
| Java | 17 |
| **Dependencies** | *Spring Web*, *Spring Boot Actuator*, *Zipkin* |

> Download the ZIP and extract to `C:\Projects\Lab12\user-service`.

> 🔧 **After importing**, add Micrometer Tracing Brave to `pom.xml`:
```xml
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>2023.0.2</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

1. **Import into IDE** (*File → Open →* project).  
2. **Create REST controller**  
3. **Add application.properties**  
4. **Run service** (`mvn spring-boot:run`).

---

## Part 2 – Create **OrderService**

| Field | Value |
|-------|-------|
| **Project** | **OrderService** |
| Dependencies | *Spring Web*, *Spring Reactive Web*, *Spring Boot Actuator*, *Zipkin* |

> Repeat the same post‑import step to add `micrometer-tracing-bridge-brave`.

Create controller ➜ configure properties ➜ run service.

---

## Part 3 – Run Zipkin locally (Windows)

1. Place `zipkin-server-*-exec.jar` in `C:\zipkin\zipkin.jar`.
2. **Start Zipkin**
```powershell
cd C:\zipkin
java -jar zipkin.jar
```
3. Visit <http://localhost:9411>.  
4. Call:
```powershell
curl http://localhost:8082/orders
```
5. Click **Run Query** in Zipkin UI to view the trace.

---

## Conclusion 🎉
You have full **distributed tracing** on Spring Boot 3.4.5 with a **local Zipkin JAR**—no Docker required. 
