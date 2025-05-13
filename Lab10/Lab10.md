# **Lab 10: Use Spring Cloud Stream to Implement Event-Driven Communication Between Microservices (Spring Boot 3.4.1)**

## **Objective**
Learn how to use **Spring Cloud Stream** with **Kafka** as the message broker to enable event-driven communication among microservices. You’ll create a producer (publishing `OrderEvent`s) and a consumer (receiving those events) in Spring Boot **3.4.1**.

---

## **Installing Prerequisites**

| Tool | One-line Install Command | Purpose in This Lab |
|------|---------------------------|----------------------|
| **Java 17+** | `winget install --id=EclipseAdoptium.Temurin.17.JDK` | Required to run Spring Boot applications |
| **Maven 3.9+** | `winget install Apache.Maven` | To build and run Spring Boot projects |
| **Kafka 3.4+** | [Download Kafka](https://kafka.apache.org/downloads) | Message broker to connect services |
| **IDE** (IntelliJ or VS Code) | IntelliJ: [Download](https://www.jetbrains.com/idea/) | Develop and run the microservices |

> **What is Kafka?**  
> **Apache Kafka** is a distributed message broker. In this lab, it allows the producer (`OrderService`) to send events and the consumer (`NotificationService`) to receive them asynchronously.

> **What is Spring Cloud Stream?**  
> A Spring abstraction for messaging that decouples your microservices from specific message brokers. Here, it simplifies Kafka integration with minimal boilerplate.

---

## **Lab Steps**

### **Part 1: Installing and Running Kafka**

1. **Install Java.**
   ```bash
   java -version
   ```
   > Expected Output:
   ```
   java version "17.x.x" 20xx-xx-xx
   ```

2. **Download Kafka.**
   - Visit [Kafka Downloads](https://kafka.apache.org/downloads) and get the latest binary (`.tgz`).

3. **Extract Kafka.**
   ```bash
   tar -xvzf kafka_2.13-3.4.0.tgz -C /opt/kafka
   ```

4. **Start Zookeeper.**
   ```bash
   cd /opt/kafka
   ./bin/zookeeper-server-start.sh config/zookeeper.properties
   ```
   > If Zookeeper fails to start, ensure no other process is using port **2181**.

5. **Start Kafka broker.**
   ```bash
   ./bin/kafka-server-start.sh config/server.properties
   ```
   > Expected Output: Logs showing Kafka server started and listening on port **9092**.

6. **Create a Kafka topic.**
   ```bash
   ./bin/kafka-topics.sh --create --topic order-events --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
   ```

7. **Verify the topic creation.**
   ```bash
   ./bin/kafka-topics.sh --list --bootstrap-server localhost:9092
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
     - Spring Boot Version: 3.4.1
     - Group: `com.microservices`
     - Artifact: `order-service`
     - Dependencies:
       - Spring Web
       - Spring Cloud Stream
       - Spring Cloud Stream Kafka Binder

9. **Unzip & Open in IDE.**
   - Extract to `OrderService` folder.
   - Open folder in IntelliJ IDEA or VS Code.

10. **Configure Kafka connection.**  
    File: `src/main/resources/application.properties`
    ```properties
    spring.application.name=order-service
    spring.cloud.stream.kafka.binder.brokers=localhost:9092
    spring.cloud.stream.bindings.output.destination=order-events
    ```

11. **Create OrderEvent model.**  
    File: `src/main/java/com/microservices/orderservice/OrderEvent.java`
    ```java
    public class OrderEvent {
        private String orderId;
        private String status;
        // constructors, getters, setters, toString()
    }
    ```

12. **Create OrderProducer.**  
    File: `src/main/java/com/microservices/orderservice/OrderProducer.java`
    ```java
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
    File: `src/main/java/com/microservices/orderservice/OrderController.java`
    ```java
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
    ```bash
    ./mvnw spring-boot:run
    ```
    > If using IntelliJ, right-click the main class and choose **Run**.

15. **Test Event Publishing.**
    ```bash
    curl -X POST -H "Content-Type: application/json" -d '{"orderId":"123","status":"CREATED"}' http://localhost:8080/orders
    ```
    > Expected Output in Console:
    ```
    Order event sent for order ID: 123
    ```

---

### **Part 3: Setting Up the Consumer Microservice (`NotificationService`)**

16. **Generate Spring Boot project.**
   - Artifact: `notification-service`
   - Dependencies:
     - Spring Web
     - Spring Cloud Stream
     - Spring Cloud Stream Kafka Binder

17. **Unzip & Open in IDE.**
   - Extract to `NotificationService`.
   - Open in IntelliJ or VS Code.

18. **Configure Kafka connection.**  
    File: `src/main/resources/application.properties`
    ```properties
    spring.application.name=notification-service
    spring.cloud.stream.kafka.binder.brokers=localhost:9092
    spring.cloud.stream.bindings.input.destination=order-events
    ```

19. **Create Event Consumer.**  
    File: `src/main/java/com/microservices/notificationservice/OrderEventConsumer.java`
    ```java
    @Component
    public class OrderEventConsumer {
        @Bean
        public Consumer<OrderEvent> input() {
            return orderEvent -> System.out.println("Received order event: " + orderEvent);
        }
    }
    ```

20. **Create Event Model.**  
    File: `src/main/java/com/microservices/notificationservice/OrderEvent.java`
    ```java
    public class OrderEvent {
        private String orderId;
        private String status;
        // constructors, getters, setters, toString()
    }
    ```

21. **Run NotificationService.**
    ```bash
    ./mvnw spring-boot:run
    ```
    > If using IntelliJ, right-click the main class and run.

22. **Verify Event Reception.**
    ```bash
    curl -X POST -H "Content-Type: application/json" -d '{"orderId":"456","status":"CREATED"}' http://localhost:8080/orders
    ```
    > Expected Output in NotificationService Console:
    ```
    Received order event: OrderEvent{orderId='456', status='CREATED'}
    ```

---

## **Optional Exercises**

- Add a **dead-letter queue** for failed events.
- Enhance `OrderEvent` with fields like `timestamp`, `customerId`.
- Run multiple NotificationService instances to observe **Kafka partitioning**.

---

## **Conclusion 🎉**
You now:
- Installed and ran Kafka.
- Built a producer that sends events.
- Built a consumer that reacts to those events.
- Tested end-to-end event-driven messaging with Spring Cloud Stream.

**Next Experiment:** Integrate this flow into a full CI/CD pipeline or add retry/backoff logic for failed message handling.

> **Troubleshooting Tip:**
> If messages are not received, confirm both services are running, the topic name matches, and no firewall blocks `localhost:9092`.
