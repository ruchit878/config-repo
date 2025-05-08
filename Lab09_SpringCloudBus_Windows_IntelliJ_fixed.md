
# **Lab 9: Event‑Based Config Refresh with Spring Cloud Bus (Spring Boot 3.4.5, Windows + IntelliJ)**

## **Objective**
Demonstrate how multiple Spring Boot microservices **auto‑reload their configuration** without a restart.  
You will:

1. Run **Kafka + ZooKeeper** locally.  
2. Stand up a **Spring Cloud Config Server** that reads properties from **GitHub**.  
3. Build two client services that refresh through **Spring Cloud Bus**.

Everything is done on **Windows 10/11** with **PowerShell (Administrator)** and **IntelliJ IDEA**.

---

## **Prerequisites**

| Tool | Install / Link | Purpose |
|------|----------------|---------|
| Java 17 SDK | `winget install --id EclipseAdoptium.Temurin.17.JDK` | Run Spring Boot apps |
| Git | `winget install --id Git.Git` | Version control, push to GitHub |
| Maven 3.9+ | `winget install --id Apache.Maven` | Build/run projects |
| IntelliJ IDEA (Community) | <https://www.jetbrains.com/idea/download/> | IDE used in all steps |
| Kafka 3.9.0 ZIP | Provided in **Desktop\Spring Clouds** zip (extract later) | Message broker |
| GitHub account | <https://github.com/join> | Remote config repo |

---

## **GitHub Setup (once)**

1. **Find your username**  
   After signing‑in, click the avatar in the top‑right. The bold text beside it is your **GitHub username**. Copy this value for later.

2. **Create a repository**  
   1. Click **➕ New → New repository**.  
   2. **Repository name**: `config-repo`  
   3. **☑ Add a README** (check this).  
   4. Click **Create repository**.

3. **Add a properties file**  
   1. In the new repo page, click **Add file → Create new file**.  
   2. **File name**: `user-service.properties`  
   3. Paste:  
      ```properties
      # placeholder
      ```  
   4. **Commit new file**.

4. **Clone the repo locally (PowerShell Admin)**  
   Open *Windows Terminal (Admin)* and run:  
   ```powershell
   git clone https://github.com/<your-username>/config-repo.git C:\SpringCloud\others\config-repo
   ```
   Expected output ends with: `Checking connectivity... done.`  

> **Already have** `C:\SpringCloud\others\config-repo` with the files above? Skip to Part 1.

---

## **Part 1 – Kafka & ZooKeeper**

> Kafka carries Spring Cloud Bus events.

**1. Prepare folders**  
   * Create `C:\kafka`.  
   * Locate **Desktop\Spring Clouds\kafka_2.12‑3.9.0.zip**, right‑click → **Extract All…** → choose `C:\kafka`.

**2. Start services (each in its own _PowerShell (Admin)_ window)**  

| Window | Command | What you should see |
|--------|---------|---------------------|
| 1 – ZooKeeper | ```powershell
cd C:\kafka
.\kafka_2.12-3.9.0\bin\windows\zookeeper-server-start.bat ..\..\config\zookeeper.properties
``` | last line: `binding to port 0.0.0.0/0.0.0.0:2181` |
| 2 – Kafka | ```powershell
cd C:\kafka
.\kafka_2.12-3.9.0\bin\windows\kafka-server-start.bat ..\..\config\server.properties
``` | `[KafkaServer id=0] started` |
| 3 – List topics | ```powershell
cd C:\kafka
.\kafka_2.12-3.9.0\bin\windows\kafka-topics.bat --list --bootstrap-server localhost:9092
``` | blank line **or** `__consumer_offsets` |

> **Timeout fix** – If Window 1 shows a session timeout, re‑run ZooKeeper (Window 1) then restart Kafka (Window 2).

---

## **Part 2 – Config Server**

### 2.1  Generate project

| Setting | Value |
|---------|-------|
| Folder | `C:\Projects\Lab09\config-server` |
| Group / Artifact / Name | `com.microservices` / **config-server** / **config-server** |
| Dependencies | **Cloud Bus**, **Config Server**, **Spring Boot Actuator** |

Download the ZIP, then **Extract All…** to the folder above.

### 2.2  Open in IntelliJ  
*File → Open…* → choose `C:\Projects\Lab09\config-server`.

### 2.3  Enable Config Server  
Open `src/main/java/com/microservices/configserver/ConfigServerApplication.java` and add:

```java
@SpringBootApplication
@EnableConfigServer          // add this line
public class ConfigServerApplication {
```

*(No need to add a new class; Initializr already generated it.)*

### 2.4  Clean up dependency versions  
If any Spring‑Cloud starters in **`pom.xml`** contain a `<version>` tag, delete that tag.  
Example:

```xml
<!-- Before -->
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-bus-kafka</artifactId>
  <version>4.2.1</version>
</dependency>

<!-- After -->
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-bus-kafka</artifactId>
</dependency>
```

### 2.5  `application.properties`  

```properties
server.port=8888
spring.application.name=config-server

spring.cloud.config.server.git.uri=https://github.com/<your-username>/config-repo
spring.cloud.bus.enabled=true
spring.cloud.stream.kafka.binder.brokers=localhost:9092

management.endpoints.web.exposure.include=health,info,refresh,busrefresh
```

### 2.6  Run Config Server  
In IntelliJ, *right‑click* **ConfigServerApplication.java → Run**, or open the *IDE terminal* and execute:

```powershell
mvn spring-boot:run
```

Expected tail:

```
Started ConfigServerApplication in X s
Tomcat started on port(s): 8888
```

---

## **Part 3 – Client Service (`user-service`)**

### 3.1  Generate project

| Setting | Value |
|---------|-------|
| Folder | `C:\Projects\Lab09\user-service` |
| Group / Artifact / Name | `com.microservices` / **user-service** / **user-service** |
| Dependencies | **Cloud Bus**, **Config Client**, **Spring Boot Actuator**, **Spring Web** |

Extract ZIP into the folder shown.

### 3.2  Open in IntelliJ  
*File → Open…* → `C:\Projects\Lab09\user-service`.

### 3.3  Remove version tags  
Delete any `<version>` elements on Spring‑Cloud starters as in Part 2 §2.4.

### 3.4  `application.properties`

```properties
server.port=8081
spring.application.name=user-service

spring.cloud.config.uri=http://localhost:8888
spring.cloud.bus.enabled=true
spring.cloud.stream.kafka.binder.brokers=localhost:9092

management.endpoints.web.exposure.include=health,info,refresh
```

### 3.5  Add REST controller  

`src/main/java/com/microservices/userservice/ConfigController.java`

```java
package com.microservices.userservice;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RefreshScope
public class ConfigController {

    @Value("${message:Default message}")
    private String message;

    @GetMapping("/message")
    public String getMessage() {
        return message;
    }
}
```

### 3.6  Run user‑service  
*Right‑click* **UserServiceApplication.java → Run** or in IntelliJ terminal:

```powershell
mvn spring-boot:run
```

Console ends with:

```
Started UserServiceApplication on port 8081
```

### 3.7  Test endpoint *(new PowerShell Admin window)*  

```powershell
curl http://localhost:8081/message
```

Expected:

```
Default message
```

---

## **Part 4 – Broadcast Config Changes**

1. **Edit GitHub file**  
   On GitHub, open `user-service.properties` → **Edit** → replace with:  
   ```properties
   message=Hello from Config Server!
   ```  
   Click **Commit changes**.

2. **Trigger refresh** *(new PowerShell Admin window)*  
   ```powershell
   curl -X POST http://localhost:8888/actuator/busrefresh
   ```
   Headers show:

   ```
   HTTP/1.1 204 No Content
   Date: <timestamp>
   ```

3. **Verify update**  
   ```powershell
   curl http://localhost:8081/message
   ```
   Returns `Hello from Config Server!`.

---

## **Part 5 – Second Client (`product-service`)**

Repeat Part 3 with the following adjustments:

| Setting | Value |
|---------|-------|
| Folder | `C:\Projects\Lab09\product-service` |
| Group / Artifact / Name | `com.microservices` / **product-service** / **product-service** |
| Port | `8082` |
| GitHub file | `product-service.properties` (same property content) |

After running, execute the `busrefresh` command again, then:

```powershell
curl http://localhost:8081/message   # user-service
curl http://localhost:8082/message   # product-service
```

Both should display the latest message.

---

## **Conclusion**

You now have Kafka, a Config Server, and two Spring Boot services that **instantly reload configuration** via Spring Cloud Bus. Experiment by changing properties in GitHub and watching all services update without a restart!
