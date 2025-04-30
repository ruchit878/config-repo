# **Lab 9: Set Up Event-Based Messaging Between Microservices Using Spring Cloud Bus (Spring Boot 3.4.x, Windows Edition)**

## **Objective**
Learn how to use **Spring Cloud Bus** with **Spring Boot 3.4.x** to propagate configuration changes (and other events) among microservices in real time — this time on **Windows 10/11** with **PowerShell**. You’ll integrate **Spring Cloud Config Server**, **Kafka**, and **Spring Cloud Bus** so that multiple microservices automatically refresh updates without manual restarts.

---

## **Prerequisites**

1. **Java 17+** — `java -version`  
2. **Git** — `git --version`  
3. **Maven** (or use the provided `mvnw`) — `mvn -v`  
4. **IDE** (IntelliJ IDEA, Eclipse, VS Code)  
5. **Apache Kafka 3.x** with ZooKeeper (ZIP extracted to `C:\kafka`)  
6. **Windows PowerShell 5+** or **PowerShell Core** (run as *Administrator* when noted)  

> **Firewall note:** open ports **8888**, **9092**, and **2181** or disable Defender temporarily.  
> **Docker alternative:** see tips at the end if you’d rather run Kafka/ZooKeeper in containers.

---

## **Lab Steps**

### **Part 1 – Setting Up the Configuration Server**

1. **Generate the project (`ConfigServer`).**  
   - Visit **<https://start.spring.io>** and configure:  
     - *Project*: **Maven**  *Language*: **Java**  
     - *Spring Boot*: **3.4.5** (latest 3.4.x line)  
     - *Group*: `com.microservices`  *Artifact*: `config-server`  
     - *Name*: `ConfigServer`  
     - *Dependencies*:  
       - Spring Cloud Config Server  
       - Spring Boot Actuator  
       - Spring Cloud Bus (Kafka)  
       - Spring Cloud Stream Kafka  
   - Click **Generate**, unzip to `C:\Projects\ConfigServer`.

2. **Import the project into your IDE.**

3. **Add Spring Cloud BOM (one‑time).**  
   Insert this before `<dependencies>` in `pom.xml` and **remove explicit versions** from any Spring Cloud starters:
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

4. **Enable the Config Server.**  
   Verify `ConfigServerApplication.java`:
   ```java
   @SpringBootApplication
   @EnableConfigServer
   public class ConfigServerApplication {
     public static void main(String[] args) {
       SpringApplication.run(ConfigServerApplication.class, args);
     }
   }
   ```

5. **Create `application.properties`.**
   ```properties
   # --- Config Server ---
   server.port=8888
   spring.application.name=ConfigServer

   spring.cloud.config.server.git.uri=https://github.com/<your-username>/config-repo
   spring.cloud.bus.enabled=true
   spring.cloud.stream.kafka.binder.brokers=localhost:9092

   # Actuator
   management.endpoints.web.exposure.include=health,info,refresh,busrefresh
   management.endpoint.health.show-details=always
   ```

6. **Run the Config Server (PowerShell).**
   ```powershell
   cd C:\Projects\ConfigServer
   mvn spring-boot:run
   ```
   You should see **“Started ConfigServerApplication … on port 8888”**.

---

### **Part 2 – Setting Up Kafka on Windows**

1. **Open two PowerShell windows** (*Admin* if necessary).

2. **Start ZooKeeper (window #1).**
   ```powershell
   cd C:\kafka
   .\bin\windows\zookeeper-server-start.bat .\config\zookeeper.properties
   ```

3. **Start Kafka (window #2).**
   ```powershell
   cd C:\kafka
   .\bin\windows\kafka-server-start.bat .\config\server.properties
   ```
   > **Stuck on older batch files?** You can replace the contents of  
   > `C:\kafka\bin\windows\kafka-server-start.bat` with the official 3.x script shown in the Appendix at the end of this lab.

4. **Verify Kafka (window #3).**
   ```powershell
   cd C:\kafka
   .\bin\windows\kafka-topics.bat --list --bootstrap-server localhost:9092
   ```
   An empty list is fine—Kafka is running.

---

### **Part 3 – Creating a Client Service (`UserService`)**

1. **Generate the project.**  
   - **Artifact**: `user-service`  
   - **Dependencies**: Spring Web, Spring Cloud Config Client, Spring Cloud Bus (Kafka), Spring Cloud Stream Kafka  
   - Unzip to `C:\Projects\UserService`, import into IDE, add the same BOM.

2. **Configure `application.properties`.**
   ```properties
   server.port=8081
   spring.application.name=user-service

   spring.cloud.config.uri=http://localhost:8888
   spring.cloud.bus.enabled=true
   spring.cloud.stream.kafka.binder.brokers=localhost:9092

   management.endpoints.web.exposure.include=health,info,refresh
   ```

3. **Create a controller.**
   ```java
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

4. **Run `UserService`.**
   ```powershell
   cd C:\Projects\UserService
   mvn spring-boot:run
   ```
   Browse to <http://localhost:8081/message>.

---

### **Part 4 – Testing Event‑Based Refresh**

1. **Edit the Git config repo.**  
   Update `user-service.properties` (or `.yml`):
   ```properties
   message=Updated from Config Server at %date% %time%
   ```

2. **Commit & push.**
   ```powershell
   git add .
   git commit -m "Update message"
   git push origin main
   ```

3. **Trigger a bus refresh.**
   ```powershell
   Invoke-RestMethod -Method Post http://localhost:8888/actuator/busrefresh
   ```

4. **Verify** that <http://localhost:8081/message> shows the new text **without restarting** the service.

---

### **Part 5 – Adding Another Client Service (`ProductService`)**

1. Repeat Part 3, but use:  
   ```properties
   server.port=8082
   spring.application.name=product-service
   ```
2. Start `ProductService`, then hit `/actuator/busrefresh` once; both services refresh together.

---

## **Optional Exercises**

1. Add a new property (e.g., `service.version=1.0`) and confirm it propagates.  
2. Use Kafka CLI (`kafka-console-consumer.bat`) to watch Spring Cloud Bus topics.  
3. Launch multiple instances of `UserService` (`-Dserver.port=…`) to see cluster‑wide refresh.

---

## **Conclusion**

In this Windows‑tuned lab you:

- **Configured a Spring Cloud Config Server** on **port 8888**.  
- **Started Kafka/ZooKeeper** locally on **ports 9092 / 2181**.  
- Built **UserService** and **ProductService** that fetch remote config and auto‑refresh via **Spring Cloud Bus**.  
- Proved real‑time propagation of property changes—**no restarts required**, even on Windows.

> **Next steps:** Containerize each service with Docker Compose or move Kafka to Confluent Cloud for a fully managed bus.

---

### **Appendix – Updated `kafka-server-start.bat`**  
*(use this if the batch file bundled in your Kafka ZIP is outdated)*

```bat
@echo off
rem Licensed to the Apache Software Foundation (ASF) under one or more
rem contributor license agreements. See the NOTICE file distributed with
rem this work for additional information regarding copyright ownership.
rem The ASF licenses this file to You under the Apache License, Version 2.0
rem (the "License"); you may not use this file except in compliance with
rem the License. You may obtain a copy of the License at
rem
rem     http://www.apache.org/licenses/LICENSE-2.0
rem
rem Unless required by applicable law or agreed to in writing, software
rem distributed under the License is distributed on an "AS IS" BASIS,
rem WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
rem See the License for the specific language governing permissions and
rem limitations under the License.

SetLocal

rem --------------------------------------------------
rem Parameter handling: use KRaft config if none given
rem --------------------------------------------------
IF "%~1"=="" (
    echo No config file provided; defaulting to KRaft mode.
    set CONFIG_FILE=%~dp0..\..\config\kraft\server.properties
) ELSE (
    set CONFIG_FILE=%~1
)

rem --------------------------------------------------
rem Log4j configuration (from default)
rem --------------------------------------------------
IF "%KAFKA_LOG4J_OPTS%"=="" (
    set KAFKA_LOG4J_OPTS=-Dlog4j.configuration=file:%~dp0..\..\config\log4j.properties
)

rem --------------------------------------------------
rem Heap settings (detect 32 vs 64 bit)
rem --------------------------------------------------
IF "%KAFKA_HEAP_OPTS%"=="" (
    wmic os get osarchitecture | find /i "32-bit" >nul 2>&1
    IF ERRORLEVEL 1 (
        rem 64-bit OS
        set KAFKA_HEAP_OPTS=-Xmx1G -Xms1G
    ) ELSE (
        rem 32-bit OS
        set KAFKA_HEAP_OPTS=-Xmx512M -Xms512M
    )
)

rem --------------------------------------------------
rem KRaft (ZooKeeper-less) cluster settings
rem --------------------------------------------------
set KAFKA_CFG_PROCESS_ROLES=broker,controller
set KAFKA_CFG_NODE_ID=1
set KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=1@localhost:9093
set KAFKA_LOG_DIRS=%~dp0..\..\data\kraft-logs
set KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER

rem --------------------------------------------------
rem Start Kafka broker (KRaft mode or user-specified config)
rem --------------------------------------------------
"%~dp0kafka-run-class.bat" kafka.Kafka "%CONFIG_FILE%"

EndLocal
```

---

### **Common Windows Pitfalls & Tips**

- **Encoding:** ensure `application.properties` files are UTF‑8 without BOM.  
- **Line endings:** use LF or configure Git to auto‑convert CRLF.  
- **Ports in use:** if **8888 / 8081 / 9092** are taken, change them everywhere.  
- **Firewall & antivirus:** can block Kafka; temporarily disable if brokers won’t bind.  
- **`mvnw` permissions:** if `./mvnw` fails, run `git update-index --chmod=+x mvnw` or just use `mvn spring-boot:run`.  
- **Docker instead of ZIP:**  

  ```powershell
  docker run -d --name zookeeper -p 2181:2181 bitnami/zookeeper
  docker run -d --name kafka -p 9092:9092 `
    -e KAFKA_BROKER_ID=1 `
    -e KAFKA_ZOOKEEPER_CONNECT=host.docker.internal:2181 `
    -e ALLOW_PLAINTEXT_LISTENER=yes `
    bitnami/kafka
  ```

Enjoy the lab!
