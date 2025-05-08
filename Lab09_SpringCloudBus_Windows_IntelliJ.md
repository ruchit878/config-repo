# **Lab 9: Event‑Based Config Refresh with Spring Cloud Bus (Spring Boot 3.4.5, Windows + IntelliJ)**

## **Objective**
Set up an end‑to‑end demo where **multiple Spring Boot microservices automatically reload configuration** without a restart.  
You will:  

1. Run **Kafka + ZooKeeper** locally (event backbone).  
2. Create a **Config Server** that reads properties from **GitHub**.  
3. Build two **client services** that fetch config and refresh via **Spring Cloud Bus**.  

Everything is done on **Windows 10/11**, using **PowerShell (Administrator)** and **IntelliJ IDEA**.

---

## GitHub Setup — do this once
> We need a Git repo that the Config Server can read.

| Step | What to do | Expected Result |
|------|------------|-----------------|
| 1 | Sign in to <https://github.com> (or create an account). Grab your **username** from the top-right avatar; you’ll replace `<your-username>` below. | — |
| 2 | Click **➕ New → New repository**.<br>• **Name**: `config-repo`<br>• **☑ Add a README** ✔<br>Click **Create repository**. | Repo page shows *README.md* |
| 3 | Still on GitHub, click **Add file → Create new file**.<br>• **Filename**: `user-service.properties`<br>• Put only a comment for now: `# placeholder`<br>Scroll down → **Commit new file**. | File appears in the repo |

---

## **Part 1 – Kafka & ZooKeeper**

> Kafka carries Spring Cloud Bus events.

| Action | Exact Command (run in **PowerShell Admin**) | Expected Output |
|--------|---------------------------------------------|-----------------|
| **A. Prepare** | 1. Create folder `C:\kafka`.<br>2. On your Desktop you’ll find **Spring Clouds\kafka_2.12‑3.9.0.zip** → *Right‑click → Extract All…* → choose **`C:\kafka`**. | Folder *C:\kafka\kafka_2.12‑3.9.0* |
| **B. Start ZooKeeper** *(Window 1)* | ```powershell
cd C:\kafka
.\kafka_2.12-3.9.0\bin\windows\zookeeper-server-start.bat ..\..\config\zookeeper.properties
``` | Last line shows `binding to port 0.0.0.0/0.0.0.0:2181` |
| **C. Start Kafka Broker** *(Window 2)* | ```powershell
cd C:\kafka
.\kafka_2.12-3.9.0\bin\windows\kafka-server-start.bat ..\..\config\server.properties
``` | Look for `[KafkaServer id=0] started` |
| **D. List Topics** *(Window 3)* | ```powershell
cd C:\kafka
.\kafka_2.12-3.9.0\bin\windows\kafka-topics.bat --list --bootstrap-server localhost:9092
``` | Either blank or `__consumer_offsets` |

> **Troubleshoot:** If ZooKeeper window shows a session timeout, rerun Step B, then Step C.

---

## **Part 2 – Config Server**

### 2.1  Create Project

| | Setting |
|-|---------|
| **Folder** | `C:\Projects\Lab09\config-server` |
| **Spring Initializr** | <https://start.spring.io> |
| **Group / Artifact / Name** | `com.microservices` / **config-server** / **config-server** |
| **Dependencies** | **Cloud Bus**, **Config Server**, **Spring Boot Actuator** |

Download the ZIP → **Extract All…** to **`C:\Projects\Lab09\config-server`**.

### 2.2  Open in IntelliJ

*File → Open…* → choose the **config-server** folder. IntelliJ detects Maven and indexes the project.

### 2.3  Enable Config Server Annotation

Open **`src/main/java/com/microservices/configserver/ConfigServerApplication.java`**  
Add one line just under `@SpringBootApplication`:

```java
@SpringBootApplication
@EnableConfigServer   // <‑‑ add this
public class ConfigServerApplication {
```

*(The class already exists; only the annotation is new.)*

### 2.4  Remove Version Tags (no BOM step!)

Maven **already** contains a `<dependencyManagement>` block.  
If any Spring‑Cloud starters show a `<version>` tag, delete it:

```xml
<!-- Before -->
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-bus-kafka</artifactId>
  <version>4.2.1</version>   <!-- remove -->
</dependency>

<!-- After -->
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-bus-kafka</artifactId>
</dependency>
```

### 2.5  `application.properties`

Create/overwrite **`src/main/resources/application.properties`**:

```properties
server.port=8888
spring.application.name=config-server

spring.cloud.config.server.git.uri=https://github.com/<your-username>/config-repo
spring.cloud.bus.enabled=true
spring.cloud.stream.kafka.binder.brokers=localhost:9092

management.endpoints.web.exposure.include=health,info,refresh,busrefresh
```

### 2.6  Run in IntelliJ

*Right‑click* **ConfigServerApplication.java → Run**, **or** open IntelliJ Terminal and:

```powershell
mvn spring-boot:run
```

Expected console tail:

```
Started ConfigServerApplication in 6 s
Tomcat started on port(s): 8888
```

---

## **Part 3 – Client Service (`user-service`)**

### 3.1  Create Project

| | Setting |
|-|---------|
| **Folder** | `C:\Projects\Lab09\user-service` |
| **Spring Initializr** | <https://start.spring.io> |
| **Group / Artifact / Name** | `com.microservices` / **user-service** / **user-service** |
| **Dependencies** | **Cloud Bus**, **Config Client**, **Spring Boot Actuator**, **Spring Web** |

Download ZIP → extract to the folder above.

### 3.2  Open in IntelliJ  
(File → Open… → choose **user-service**).

### 3.3  Remove Version Tags  
Delete any `<version>` elements on Spring‑Cloud starters just like Part 2 §2.4.

### 3.4  `application.properties`

```properties
server.port=8081
spring.application.name=user-service

spring.cloud.config.uri=http://localhost:8888
spring.cloud.bus.enabled=true
spring.cloud.stream.kafka.binder.brokers=localhost:9092

management.endpoints.web.exposure.include=health,info,refresh
```

### 3.5  Add REST Controller

File **`src/main/java/com/microservices/userservice/ConfigController.java`**:

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

### 3.6  Run in IntelliJ

Run **UserServiceApplication** or use:

```powershell
mvn spring-boot:run
```

Expected tail:

```
Started UserServiceApplication in 5 s
```

### 3.7  Test endpoint *(new PowerShell Admin window)*

```powershell
curl http://localhost:8081/message
```

Output:

```
Default message
```

---

## **Part 4 – Broadcast Config Changes**

1. **Edit property file in GitHub Web UI**  
   Open `config-repo/user-service.properties` online → click ✏️ **Edit** → replace content with:  
   ```properties
   message=Hello from Config Server!
   ```  
   → **Commit changes**.

2. **Trigger bus refresh** *(new PowerShell Admin window)*  
   ```powershell
   curl -X POST http://localhost:8888/actuator/busrefresh
   ```
   Expected output headers (no body):

   ```
   HTTP/1.1 204 No Content
   Date: Thu, 08 May 2025 17:42:00 GMT
   ```

3. **Verify**  
   ```powershell
   curl http://localhost:8081/message
   ```
   Returns:

   ```
   Hello from Config Server!
   ```

---

## **Part 5 – Second Client (`product-service`)**

Repeat **all steps of Part 3** but with:

| Setting | Value |
|---------|-------|
| **Artifact / Name** | `product-service` |
| **Port** | `8082` |
| **File in GitHub** | `product-service.properties` |

After running, execute the same **busrefresh** command; both:

```
curl http://localhost:8081/message
curl http://localhost:8082/message
```

should show the updated message.

---

## **Conclusion**

You now have:

* Kafka + ZooKeeper running on Windows.  
* A Config Server pulling from GitHub.  
* Two services that **auto‑refresh** configuration via Spring Cloud Bus.  

A single HTTP `POST /actuator/busrefresh` updates every service instantly. Enjoy exploring more dynamic config patterns!
