# **Lab 9: Set Up Event-Based Messaging Between Microservices Using Spring Cloud Bus (Spring Boot 3.4.5, Windows Edition)**

## **Objective**
Learn how to use **Spring Cloud Bus** with **Spring Boot 3.4.5** to propagate configuration changes (and other events) among microservices in real time — on **Windows 10/11** with **PowerShell**. You’ll integrate **Spring Cloud Config Server**, **Kafka**, and **Spring Cloud Bus** so that multiple microservices automatically refresh updates without manual restarts.

---

## **Prerequisites & One‑Time Setup**

| Tool | Install Command / Link | Purpose in This Lab |
|------|------------------------|---------------------|
| **Java 17 SDK** | `winget install --id EclipseAdoptium.Temurin.17.JDK` | Runs all Spring Boot apps |
| **Git** | `winget install --id Git.Git` | Version‑control and push code to GitHub |
| **Maven 3.9+** | `winget install --id Apache.Maven` | Builds & runs Spring projects |
| **Visual Studio Code** | `winget install --id Microsoft.VisualStudioCode` | Code editor & integrated terminal |
| **Docker Desktop** <br>(*optional – replaces ZIP Kafka install*) | <https://www.docker.com/products/docker-desktop/> | Simplifies running Kafka/ZooKeeper |
| **Kafka 3.x** (ZIP) | Download → <https://kafka.apache.org/downloads> and extract to `C:\kafka` | Message broker used by Spring Cloud Bus |
| **GitHub account** | <https://github.com/join> | Hosts your configuration repo |

> **What is Spring Boot?** A framework that lets you package & run a microservice with a single command.  
> **What is Maven?** A build tool that downloads Java libraries and starts Spring Boot.  
> **What is Spring Cloud Bus?** It broadcasts events (via Kafka) so all services auto‑refresh config.  
> **What is Kafka?** A high‑throughput message broker carrying Bus events.  
> **Why GitHub?** Config Server reads property files from a Git repo you control.

### GitHub Setup (once)

1. **Create an account** → <https://github.com/join>.
2. **Create a repository**  
   - Click **➕ New** → **New repository**.  
   - *Repository name*: **config‑repo**.  
   - Leave **README** unchecked → **Create repository**.
3. **Clone locally**  
   ```powershell
   git clone https://github.com/<your‑username>/config-repo.git
   ```
   *Expected output*  
   ```text
   Cloning into 'config-repo'...
   ```
4. **Add placeholder file**  
   ```powershell
   cd config-repo
   echo # placeholder > application.properties
   git add .
   git commit -m "Initial commit"
   git push origin main
   ```
   *Expected output*  
   ```text
   [main abc1234] Initial commit
   ```

---

## **Lab Steps**

### **Part 1 – Setting Up the Configuration Server**

1. **Generate project (`ConfigServer`)**  
   - Open <https://start.spring.io>  
   - **Project** Maven **Language** Java **Spring Boot** 3.4.5  
   - **Group** `com.microservices` **Artifact** `config-server` **Name** `ConfigServer`  
   - **Dependencies**: Spring Cloud Config Server, Spring Boot Actuator, Spring Cloud Bus Kafka, Spring Cloud Stream Kafka  
   - Click **Generate** → unzip to `C:\Projects\ConfigServer`.

2. **Import into VS Code**  
   - *File → Open Folder* → choose `C:\Projects\ConfigServer`.  
   - If prompted, install the *Java Extension Pack*.  
   - *Expected*: Status bar shows “Java Language Server ready”.

3. **Add Spring Cloud BOM**  
   Edit `C:\Projects\ConfigServer\pom.xml` and insert before `<dependencies>`:  
   ```xml
   <dependencyManagement>
     <dependencies>
       <dependency>
         <groupId>org.springframework.cloud</groupId>
         <artifactId>spring-cloud-dependencies</artifactId>
         <version>2024.0.1</version>
         <type>pom</type>
         <scope>import</scope>
       </dependency>
     </dependencies>
   </dependencyManagement>
   ```
   Then **remove** any `<version>` tags from Spring Cloud starter dependencies.

4. **Enable Config Server** – file `ConfigServer/src/main/java/com/microservices/configserver/ConfigServerApplication.java`
   ```java
   package com.microservices.configserver;

   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.autoconfigure.SpringBootApplication;
   import org.springframework.cloud.config.server.EnableConfigServer;

   @SpringBootApplication
   @EnableConfigServer
   public class ConfigServerApplication {
       public static void main(String[] args) {
           SpringApplication.run(ConfigServerApplication.class, args);
       }
   }
   ```

5. **Create properties file** – `ConfigServer/src/main/resources/application.properties`
   ```properties
   server.port=8888
   spring.application.name=config-server

   spring.cloud.config.server.git.uri=https://github.com/<your-username>/config-repo
   spring.cloud.bus.enabled=true
   spring.cloud.stream.kafka.binder.brokers=localhost:9092

   management.endpoints.web.exposure.include=health,info,refresh,busrefresh
   ```

6. **Run Config Server**
   ```powershell
   cd C:\Projects\ConfigServer
   mvn spring-boot:run
   ```
   *Expected output*  
   ```text
   Started ConfigServerApplication in 6.2 seconds
   Tomcat started on port(s): 8888
   ```

---

### **Part 2 – Starting Kafka & ZooKeeper**

1. **Start ZooKeeper**
   ```powershell
   cd C:\kafka
   .\bin\windows\zookeeper-server-start.bat .\config\zookeeper.properties
   ```
   *Expected* `binding to port 2181`.

2. **Start Kafka (new window)**
   ```powershell
   cd C:\kafka
   .\bin\windows\kafka-server-start.bat .\config\server.properties
   ```
   *Expected* `[KafkaServer id=0] started`.

3. **List topics**
   ```powershell
   cd C:\kafka
   .\bin\windows\kafka-topics.bat --list --bootstrap-server localhost:9092
   ```
   *Expected output*  
   ```text
   __consumer_offsets
   ```

---

### **Part 3 – Creating the Client Service (`UserService`)**

1. **Generate project (`UserService`)**  
   - Open <https://start.spring.io>  
   - **Project** Maven **Language** Java **Spring Boot** 3.4.5  
   - **Group** `com.microservices` **Artifact** `user-service` **Name** `UserService`  
   - **Dependencies**: Spring Web, Spring Cloud Config Client, Spring Cloud Bus Kafka, Spring Cloud Stream Kafka  
   - Click **Generate** → unzip to `C:\Projects\UserService`.

2. **Import into VS Code**  
   - *File → Open Folder* → choose `C:\Projects\UserService`.  
   - Ensure Java extensions are enabled (one‑time).

3. **Add Spring Cloud BOM**  
   Open `C:\Projects\UserService\pom.xml`, insert the **same `<dependencyManagement>` block** as in Part 1 Step 3, then remove `<version>` tags from Cloud starters.

4. **Create properties file** – `UserService/src/main/resources/application.properties`
   ```properties
   server.port=8081
   spring.application.name=user-service

   spring.cloud.config.uri=http://localhost:8888
   spring.cloud.bus.enabled=true
   spring.cloud.stream.kafka.binder.brokers=localhost:9092

   management.endpoints.web.exposure.include=health,info,refresh
   ```

5. **Add REST controller** – `UserService/src/main/java/com/microservices/userservice/ConfigController.java`
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

6. **Run `UserService`**
   ```powershell
   cd C:\Projects\UserService
   mvn spring-boot:run
   ```
   *Expected* `Started UserServiceApplication` on port 8081.

7. **Call the endpoint**
   ```powershell
   curl http://localhost:8081/message
   ```
   *Expected output* `Default message`.

---

### **Part 4 – Broadcasting Config Changes**

1. **Add property to GitHub repo** – edit `config-repo/user-service.properties`
   ```properties
   message=Hello from Config Server!
   ```
   Then:
   ```powershell
   cd config-repo
   git add .
   git commit -m "Add message property"
   git push origin main
   ```
   *Expected* commit hash displayed.

2. **Trigger Bus refresh**
   ```powershell
   curl -X POST http://localhost:8888/actuator/busrefresh
   ```
   *Expected output*  
   ```json
   {"status":"OK", ...}
   ```

3. **Verify update**
   ```powershell
   curl http://localhost:8081/message
   ```
   *Expected output* `Hello from Config Server!`

---

### **Part 5 – Adding a Second Client (`ProductService`)**

1. **Generate project (`ProductService`)**  
   - Open <https://start.spring.io>  
   - **Project** Maven **Language** Java **Spring Boot** 3.4.5  
   - **Group** `com.microservices` **Artifact** `product-service` **Name** `ProductService`  
   - **Dependencies**: Spring Web, Spring Cloud Config Client, Spring Cloud Bus Kafka, Spring Cloud Stream Kafka  
   - Click **Generate** → unzip to `C:\Projects\ProductService`.

2. **Import into VS Code**  
   - *File → Open Folder* → choose `C:\Projects\ProductService`.

3. **Add Spring Cloud BOM**  
   Edit `C:\Projects\ProductService\pom.xml`, insert the `<dependencyManagement>` block and remove explicit versions.

4. **Create properties file** – `ProductService/src/main/resources/application.properties`
   ```properties
   server.port=8082
   spring.application.name=product-service

   spring.cloud.config.uri=http://localhost:8888
   spring.cloud.bus.enabled=true
   spring.cloud.stream.kafka.binder.brokers=localhost:9092

   management.endpoints.web.exposure.include=health,info,refresh
   ```

5. **Add same REST controller** – `ProductService/src/main/java/com/microservices/productservice/ConfigController.java`
   ```java
   package com.microservices.productservice;

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

6. **Run `ProductService`**
   ```powershell
   cd C:\Projects\ProductService
   mvn spring-boot:run
   ```
   *Expected* `Started ProductServiceApplication` on port 8082.

7. **Global refresh test**
   ```powershell
   curl -X POST http://localhost:8888/actuator/busrefresh
   ```
   Call both endpoints:
   ```powershell
   curl http://localhost:8081/message   # UserService
   curl http://localhost:8082/message   # ProductService
   ```
   *Expected output* both show the latest value from GitHub.

---

## **Conclusion**
You installed all tools, set up GitHub, and built three Spring Boot services that share configuration via **Spring Cloud Bus over Kafka**. A single `/actuator/busrefresh` now updates every running instance in real time.

