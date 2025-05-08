
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
         <version>2025.0.0-SNAPSHOT</version>
         <type>pom</type>
         <scope>import</scope>
       </dependency>
     </dependencies>
   </dependencyManagement>
   ```
   Then remove any `<version>` tags from Spring Cloud starters.

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
   ```powershell
   mvn spring-boot:run
   ```

---

### **Part 2 – Starting Kafka & ZooKeeper**

*(commands unchanged)*

---

### **Part 3 – Creating the Client Service (`UserService`)**

*(steps identical except for dependency list, BOM, and properties)*

5. **Create properties** – file **`src/main/resources/application.properties`**
   ```properties
   server.port=8081
   spring.application.name=user-service

   spring.config.import=configserver:http://localhost:8888
   spring.cloud.bus.enabled=true
   spring.cloud.stream.kafka.binder.brokers=localhost:9092

   management.endpoints.web.exposure.include=health,info,refresh,busrefresh
   ```

---

### **Part 4 – Broadcasting Config Changes**

*(unchanged)*

---

### **Part 5 – Adding a Second Client Service (`ProductService`)**

4. **Create properties** – file **`src/main/resources/application.properties`**
   ```properties
   server.port=8082
   spring.application.name=product-service

   spring.config.import=configserver:http://localhost:8888
   spring.cloud.bus.enabled=true
   spring.cloud.stream.kafka.binder.brokers=localhost:9092

   management.endpoints.web.exposure.include=health,info,refresh,busrefresh
   ```

*(remaining steps unchanged)*

---

## **Conclusion**
You installed every tool, configured GitHub, and built three Spring Boot microservices that share configuration through **Spring Cloud Bus over Kafka** on **Spring Boot 3.4.5 + Spring Cloud 2025.0.0-SNAPSHOT**. A single `/actuator/busrefresh` request now updates all running services in real time.
