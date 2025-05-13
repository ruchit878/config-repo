# **Lab 10: Use Spring Cloud Stream to Implement Event-Driven Communication Between Microservices (Spring Boot 3.4.5)**

## **Objective**
Learn how to use **Spring Cloud Stream** with **Kafka** as the message broker to enable event-driven communication among microservices. You’ll create a producer (publishing `OrderEvent`s) and a consumer (receiving those events) in Spring Boot **3.4.5**.

---

## **Installing Prerequisites**

| Tool | One-line Install Command | Purpose in This Lab |
|------|---------------------------|----------------------|
| **Java 17+** | `winget install --id=EclipseAdoptium.Temurin.17.JDK` | Required to run Spring Boot applications |
| **Maven 3.9+** | `winget install Apache.Maven` | To build and run Spring Boot projects |
| **Kafka 3.5+** | [Download Kafka](https://kafka.apache.org/downloads) | Message broker to connect services |
| **IDE** (IntelliJ or VS Code) | IntelliJ: [Download](https://www.jetbrains.com/idea/) | Develop and run the microservices |

> **What is Kafka?**  
> **Apache Kafka** is a distributed message broker. In this lab, it allows the producer (`OrderService`) to send events and the consumer (`NotificationService`) to receive them asynchronously.

> **What is Spring Cloud Stream?**  
> A Spring abstraction for messaging that decouples your microservices from specific message brokers. Here, it simplifies Kafka integration with minimal boilerplate.

---

## **Lab Steps**

### **Part 1: Installing and Running Kafka (Windows)**

1. **Install Java.**
   ```cmd
   java -version
   ```
   > Expected Output:
   ```
   java version "17.x.x"
   ```

2. **Download Kafka.**
   - Visit [Kafka Downloads](https://kafka.apache.org/downloads) and get the latest binary (`.tgz`).

3. **Extract Kafka.**
   - Use a tool like 7-Zip to extract the `.tgz` and `.tar` files into `C:\kafka`.

4. **Start Zookeeper.**
   ```cmd
   cd C:\kafka\kafka_2.13-3.5.0
   .\bin\windows\zookeeper-server-start.bat .\config\zookeeper.properties
   ```

5. **Start Kafka broker.**
   ```cmd
   .\bin\windows\kafka-server-start.bat .\config\server.properties
   ```

6. **Create a Kafka topic.**
   ```cmd
   .\bin\windows\kafka-topics.bat --create --topic order-events --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
   ```

7. **Verify the topic creation.**
   ```cmd
   .\bin\windows\kafka-topics.bat --list --bootstrap-server localhost:9092
   ```
   > Expected Output:
   ```
   order-events
   ```

---

### **Part 2: Setting Up the Producer Microservice (`OrderService`)**

8. **Generate Spring Boot project.**
   - Visit [start.spring.io](https://start.spring.io/)
   - Use:
     - Spring Boot Version: **3.4.5**
     - Group: `com.microservices`
     - Artifact: `order-service`
     - Dependencies:
       - Spring Web
       - Spring Cloud Stream Kafka

   > ⚠️ **Note:** If `Spring Cloud Stream Kafka` is not available via UI, add this manually in `pom.xml`:
   ```xml
   <dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-stream-kafka</artifactId>
   </dependency>
   ```

9. **Unzip & Open in IDE.**
   - Extract to `OrderService` folder.
   - Open in IntelliJ or VS Code.

10. **Configure Kafka connection.**  
    File: `src\main\resources\application.properties`
    ```properties
    spring.application.name=order-service
    spring.cloud.stream.kafka.binder.brokers=localhost:9092
    spring.cloud.stream.bindings.output.destination=order-events
    ```

11. **Create OrderEvent model.**  
    File: `src\main\java\com\microservices\orderservice\OrderEvent.java`
    ```java
    package com.microservices.order_service;

    public class OrderEvent {
        private String orderId;
        private String status;

        public OrderEvent() {}

        public OrderEvent(String orderId, String status) {
            this.orderId = orderId;
            this.status = status;
        }

        public String getOrderId() { return orderId; }
        public void setOrderId(String orderId) { this.orderId = orderId; }

        public String getStatus() { return status; }
        public void setStatus(String status) { this.status = status; }

        @Override
        public String toString() {
            return "OrderEvent{" +
                    "orderId='" + orderId + '\'' +
                    ", status='" + status + '\'' +
                    '}';
        }
    }
    ```

12. **Create OrderProducer.**  
    File: `OrderProducer.java`
    ```java
    package com.microservices.order_service;

    import org.springframework.cloud.stream.function.StreamBridge;
    import org.springframework.stereotype.Component;

    @Component
    public class OrderProducer {
        private final StreamBridge streamBridge;
        public OrderProducer(StreamBridge streamBridge) {
            this.streamBridge = streamBridge;
        }
        public void sendOrderEvent(OrderEvent event) {
            streamBridge.send("output", event);
        }
    }
    ```

13. **Add REST Controller.**  
    File: `OrderController.java`
    ```java
    package com.microservices.order_service;

    import org.springframework.web.bind.annotation.PostMapping;
    import org.springframework.web.bind.annotation.RequestBody;
    import org.springframework.web.bind.annotation.RestController;

    @RestController
    public class OrderController {
        private final OrderProducer orderProducer;
        public OrderController(OrderProducer orderProducer) {
            this.orderProducer = orderProducer;
        }
        @PostMapping("/orders")
        public String createOrder(@RequestBody OrderEvent orderEvent) {
            orderProducer.sendOrderEvent(orderEvent);
            return "Order event sent for order ID: " + orderEvent.getOrderId();
        }
    }
    ```

14. **Run OrderService.**
    ```cmd
    mvn spring-boot:run
    ```
    > Should start on port 8080 by default.

15. **Test Event Publishing.**
    ```cmd
    curl.exe --% -X POST -H "Content-Type: application/json" -d "{\"orderId\":\"123\",\"status\":\"CREATED\"}" http://localhost:8080/orders
    ```
    > Console output:
    ```
    Order event sent for order ID: 123
    ```

---

### **Part 3: Setting Up the Consumer Microservice (`NotificationService`)**

16. **Generate Spring Boot project.**
    - Artifact: `notification-service`
    - Dependencies:
      - Spring Web
      - Spring Cloud Stream Kafka

    > ⚠️ If not available in start.spring.io UI, add manually:
    ```xml
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-stream-kafka</artifactId>
    </dependency>
    ```

17. **Unzip & Open in IDE.**
    - Extract to `NotificationService`.
    - Open in IntelliJ or VS Code.

18. **Configure Kafka connection.**  
    File: `src\main\resources\application.properties`
    ```properties
    spring.application.name=notification-service
    server.port=8081
    spring.cloud.stream.kafka.binder.brokers=localhost:9092
    spring.cloud.stream.bindings.input.destination=order-events
    ```

19. **Create Event Consumer.**  
    File: `OrderEventConsumer.java`
    ```java
    package com.microservices.notification_service;

    import org.springframework.context.annotation.Bean;
    import org.springframework.stereotype.Component;
    import java.util.function.Consumer;

    @Component
    public class OrderEventConsumer {
        @Bean
        public Consumer<OrderEvent> input() {
            return orderEvent -> System.out.println("Received order event: " + orderEvent);
        }
    }
    ```

20. **Create Event Model.**  
    File: `OrderEvent.java`
    ```java
    package com.microservices.notification_service;

    public class OrderEvent {
        private String orderId;
        private String status;

        public OrderEvent() {}

        public OrderEvent(String orderId, String status) {
            this.orderId = orderId;
            this.status = status;
        }

        public String getOrderId() { return orderId; }
        public void setOrderId(String orderId) { this.orderId = orderId; }
        public String getStatus() { return status; }
        public void setStatus(String status) { this.status = status; }

        @Override
        public String toString() {
            return "OrderEvent{" +
                    "orderId='" + orderId + '\'' +
                    ", status='" + status + '\'' +
                    '}';
        }
    }
    ```

21. **Run NotificationService.**
    ```cmd
    mvn spring-boot:run
    ```
    > Should start on port 8081.

22. **Verify Event Reception.**
    ```cmd
    curl.exe --% -X POST -H "Content-Type: application/json" -d "{\"orderId\":\"456\",\"status\":\"CREATED\"}" http://localhost:8080/orders
    ```
    > Console output in `NotificationService`:
    ```
    Received order event: OrderEvent{orderId='456', status='CREATED'}
    ```

---

## **Conclusion 🎉**
You now:
- Installed and ran Kafka on Windows.
- Built a producer that sends events.
- Built a consumer that reacts to those events.
- Verified end-to-end event-driven messaging with Spring Cloud Stream.

**Next Experiment:** Add error-handling or retry logic to the consumer, or implement dead-letter topic routing.

> **Troubleshooting Tip:**
> If no events are received, check topic name consistency, Kafka logs, and whether both services are actively running and listening on their ports.
