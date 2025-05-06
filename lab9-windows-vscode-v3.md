# **Lab 9: Set Up Event‑Based Messaging Between Microservices Using Spring Cloud Bus (Spring Boot 3.4.5, Windows Edition)**

## **Objective**
Learn how to use **Spring Cloud Bus** with **Spring Boot 3.4.5** to propagate configuration changes (and other events) among microservices in real time on **Windows 10/11** using **PowerShell (Admin)** and **VS Code**. You’ll integrate **Spring Cloud Config Server**, **Kafka**, and **Spring Cloud Bus** so that every microservice automatically reloads configuration—no restarts needed.

---

## **Prerequisites & One‑Time Setup**

| Tool | Install Command / Link | Why You Need It |
|------|------------------------|-----------------|
| **Java 17 SDK** | `winget install --id EclipseAdoptium.Temurin.17.JDK` | Runs every Spring Boot service |
| **Git** | `winget install --id Git.Git` | Version‑control & push code to GitHub |
| **Maven 3.9+** | `winget install --id Apache.Maven` | Builds & starts Spring projects |
| **Visual Studio Code** | `winget install --id Microsoft.VisualStudioCode` | Code editor & integrated terminal |
| **Docker Desktop** (optional) | <https://www.docker.com/products/docker-desktop/> | Simplifies running Kafka/ZooKeeper |
| **Kafka 3.x ZIP** | Download → <https://kafka.apache.org/downloads>, extract to **`C:\kafka`** | Message broker for Spring Cloud Bus |
| **GitHub account** | <https://github.com/join> | Hosts your configuration repo |

> **Spring Boot** packages Tomcat, dependencies, and configuration so a microservice runs with one command.  
> **Maven** downloads libraries, compiles Java, and launches Spring Boot via `mvn spring-boot:run`.  
> **Spring Cloud Bus** broadcasts “refresh” events over **Kafka**, so all services reload updated config instantly.  
> **Kafka** is the event backbone that carries those Bus messages.  
> **GitHub** stores your property files so the Config Server can fetch them.

---

### GitHub Setup (once)

> **💡 Tip – Grab your username first:** After signing in, look at the top‑right avatar → the name beside it (or under **Settings → Profile**) is your **GitHub username**. You’ll replace `<your-username>` with this value in every URL.

1. **Create an account** → <https://github.com/join>.

2. **Create a repository**  
   - Click **➕ New** → **New repository**.  
   - *Repository name*: **config-repo** (lower‑case).  
   - Leave **README** unchecked.  
   - Click **Create repository**.

3. **Clone the repo (PowerShell Admin)**  
   Open **PowerShell as Administrator** (`Win + X` → *Windows Terminal (Admin)*):  
   ```powershell
   git clone https://github.com/<your-username>/config-repo.git
   ```
   *Expected output*  
   ```text
   Cloning into 'config-repo'...
   remote: Enumerating objects: ...
   ```

4. **Add a placeholder file**  
   ```powershell
   cd config-repo
   echo # placeholder > application.properties
   git add .
   git commit -m "Initial commit"
   git push origin main
   ```
   *Expected output*  
   ```text
   [main 1a2b3c4] Initial commit
   1 file changed, 1 insertion(+)
   ```

---

## **Lab Steps**

### **Part 1 – Setting Up the Configuration Server**

1. **Generate project (`ConfigServer`)**  
   - Go to <https://start.spring.io>  
   - **Project** Maven **Language** Java **Spring Boot** 3.4.5  
   - **Group** `com.microservices` **Artifact** `config-server` **Name** `ConfigServer`  
   - **Dependencies**:  
     - Spring Cloud Config Server  
     - Spring Boot Actuator  
     - Spring Cloud Bus Kafka  
     - Spring Cloud Stream Kafka  
   - Click **Generate** → unzip to **`C:\Projects\ConfigServer`**.

2. **Open folder in VS Code**  
   - Launch VS Code → **File › Open Folder…** → select **`C:\Projects\ConfigServer`**.  
   - **Enable Java extensions once:** click the square‑icon **Extensions** sidebar, search **“Extension Pack for Java”**, click **Install**. VS Code reloads and shows “Java Language Server ready” in the status bar.

3. **Add Spring Cloud BOM**  
   In **`pom.xml`** (root of the project) insert **above** `<dependencies>`:  
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
   Then remove any `<version>` tags from Spring Cloud starters, e.g.:

   **Before**
   ```xml
   <dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-bus-kafka</artifactId>
     <version>4.2.1</version>
   </dependency>
   ```

   **After**
   ```xml
   <dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-bus-kafka</artifactId>
   </dependency>
   ```

4. **Create main class** – file **`src/main/java/com/microservices/configserver/ConfigServerApplication.java`**
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

5. **Add properties** – file **`src/main/resources/application.properties`**
   ```properties
   server.port=8888
   spring.application.name=config-server

   spring.cloud.config.server.git.uri=https://github.com/<your-username>/config-repo
   spring.cloud.bus.enabled=true
   spring.cloud.stream.kafka.binder.brokers=localhost:9092

   management.endpoints.web.exposure.include=health,info,refresh,busrefresh
   ```

6. **Run Config Server in VS Code terminal**  
   - In VS Code, press **Ctrl + `** (back‑tick) to open the integrated terminal.  
   - Make sure the terminal is **PowerShell (Admin)** (dropdown top‑right → PowerShell → “Run as administrator”).  
   - Execute:  
     ```powershell
     mvn spring-boot:run
     ```  
   *Expected output*  
   ```text
   Started ConfigServerApplication in 6.2 s
   Tomcat started on port(s): 8888
   ```

---

### **Part 2 – Starting Kafka & ZooKeeper**

> **Run every command in a separate *PowerShell (Admin)* window.**

1. **Start ZooKeeper** (Window #1)  
   ```powershell
   cd C:\kafka
   .\bin\windows\zookeeper-server-start.bat .\config\zookeeper.properties
   ```
   *Expected* text ends with `binding to port 0.0.0.0/0.0.0.0:2181`.

2. **Start Kafka broker** (Window #2)  
   ```powershell
   cd C:\kafka
   .\bin\windows\kafka-server-start.bat .\config\server.properties
   ```
   *Expected* `[KafkaServer id=0] started`.

3. **Verify broker running** (Window #3)  
   ```powershell
   cd C:\kafka
   .\bin\windows\kafka-topics.bat --list --bootstrap-server localhost:9092
   ```
   *Expected output*  
   ```text
   __consumer_offsets
   ```

---

### **Part 3 – Creating the Client Service (`UserService`)**

1. **Generate project (`UserService`)**  
   - Go to <https://start.spring.io>  
   - **Project** Maven **Language** Java **Spring Boot** 3.4.5  
   - **Group** `com.microservices` **Artifact** `user-service` **Name** `UserService`  
   - **Dependencies**:  
     - Spring Web  
     - Spring Cloud Config Client  
     - Spring Cloud Bus Kafka  
     - Spring Cloud Stream Kafka  
   - Click **Generate** → unzip to **`C:\Projects\UserService`**.

2. **Open folder in VS Code** – **File › Open Folder…** → `C:\Projects\UserService`.

3. **Enable Java extensions** (only if you skipped earlier): open **Extensions** sidebar, search **“Extension Pack for Java”**, click **Install**.

4. **Add Spring Cloud BOM**  
   Insert the **same `<dependencyManagement>` block** from Part 1 Step 3 into `pom.xml`, **then remove `<version>` tags** from Spring Cloud starters exactly as shown in the before/after example above.

5. **Create properties** – file **`src/main/resources/application.properties`**
   ```properties
   server.port=8081
   spring.application.name=user-service

   spring.cloud.config.uri=http://localhost:8888
   spring.cloud.bus.enabled=true
   spring.cloud.stream.kafka.binder.brokers=localhost:9092

   management.endpoints.web.exposure.include=health,info,refresh
   ```

6. **Add controller** – file **`src/main/java/com/microservices/userservice/ConfigController.java`**
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

7. **Run `UserService` in VS Code terminal**  
   ```powershell
   mvn spring-boot:run
   ```
   *Expected* `Started UserServiceApplication on port 8081`.

8. **Call the endpoint (new PowerShell Admin window)**  
   Open a **new** PowerShell (Admin) window elsewhere:  
   ```powershell
   curl http://localhost:8081/message
   ```
   *Expected output*  
   ```text
   Default message
   ```

---

### **Part 4 – Broadcasting Config Changes**

1. **Add property in GitHub web UI**  
   - Replace with your username and navigate to " https://github.com/enter-your-username-here/config-repo ".  
   - Click **Add file › Create new file**.  
   - **File name**: `user-service.properties`.  
   - Paste:  
     ```properties
     message=Hello from Config Server!
     ```  
   - Scroll down → **Commit new file**.

2. **Trigger Bus refresh (PowerShell Admin window)**  
   Open a **new** PowerShell (Admin) window:  
   ```powershell
   curl -X POST http://localhost:8888/actuator/busrefresh
   ```
   *Expected output*  
   ```json
   {"status":"OK","timestamp":...}
   ```

3. **Verify update (same window)**  
   ```powershell
   curl http://localhost:8081/message
   ```
   *Expected output*  
   ```text
   Hello from Config Server!
   ```

---

### **Part 5 – Adding a Second Client Service (`ProductService`)**

1. **Generate project (`ProductService`)**  
   - Go to <https://start.spring.io>  
   - **Project** Maven **Language** Java **Spring Boot** 3.4.5  
   - **Group** `com.microservices` **Artifact** `product-service` **Name** `ProductService`  
   - **Dependencies**: Spring Web, Spring Cloud Config Client, Spring Cloud Bus Kafka, Spring Cloud Stream Kafka  
   - Click **Generate** → unzip to **`C:\Projects\ProductService`**.

2. **Open folder in VS Code** → **File › Open Folder…** → `C:\Projects\ProductService`.

3. **Add Spring Cloud BOM** to `pom.xml` and remove `<version>` tags as demonstrated earlier.

4. **Create properties** – file **`src/main/resources/application.properties`**
   ```properties
   server.port=8082
   spring.application.name=product-service

   spring.cloud.config.uri=http://localhost:8888
   spring.cloud.bus.enabled=true
   spring.cloud.stream.kafka.binder.brokers=localhost:9092

   management.endpoints.web.exposure.include=health,info,refresh
   ```

5. **Add controller** – file **`src/main/java/com/microservices/productservice/ConfigController.java`**  
   *(Same code as UserService controller.)*

6. **Run `ProductService` in VS Code terminal**  
   ```powershell
   mvn spring-boot:run
   ```
   *Expected* `Started ProductServiceApplication on port 8082`.

7. **Global refresh test (new PowerShell Admin window)**  
   ```powershell
   curl -X POST http://localhost:8888/actuator/busrefresh
   ```
   Then check both services:  
   ```powershell
   curl http://localhost:8081/message   # UserService
   curl http://localhost:8082/message   # ProductService
   ```
   *Expected output* both show the latest value from GitHub.

---

## **Conclusion**
You installed every tool, configured GitHub, and built three Spring Boot microservices that share configuration through **Spring Cloud Bus over Kafka**. A single `/actuator/busrefresh` request now updates all running services in real time.
