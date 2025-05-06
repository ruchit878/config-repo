# **Lab 10: Use Spring Cloud Stream to Implement Event‑Driven Communication Between Microservices (Spring Boot 3.4.5)**

## **Objective**
Learn how to use **Spring Cloud Stream 4.1** with **Apache Kafka 3.7** as the message broker to enable event‑driven communication among microservices. You’ll create a **producer** (publishing `OrderEvent`s) and a **consumer** (receiving those events) in Spring Boot **3.4.5**.

---

## **Lab Steps**

### **Part 1: Installing and Running Kafka 3.7**

1. **Install Java 17 +**  
   Verify Java:  
   ```bash
   java -version
   ```
   If missing, install an OpenJDK 17 build (e.g. [Adoptium Temurin 17](https://adoptium.net)).  

2. **Download Kafka 3.7.1 (Scala 2.13).**  
   ```bash
   curl -LO https://downloads.apache.org/kafka/3.7.1/kafka_2.13-3.7.1.tgz
   ```

3. **Extract Kafka.**  
   ```bash
   tar -xzf kafka_2.13-3.7.1.tgz -C $HOME
   mv $HOME/kafka_2.13-3.7.1 $HOME/kafka
   ```

4. **Start ZooKeeper** (Kafka ≤ 3.x still requires it).  
   ```bash
   $HOME/kafka/bin/zookeeper-server-start.sh $HOME/kafka/config/zookeeper.properties
   ```

5. **Start the Kafka broker** **in another terminal**:  
   ```bash
   $HOME/kafka/bin/kafka-server-start.sh $HOME/kafka/config/server.properties
   ```

6. **Create a Kafka topic** — `order-events`:  
   ```bash
   $HOME/kafka/bin/kafka-topics.sh --create        --topic order-events        --bootstrap-server localhost:9092        --partitions 1 --replication-factor 1
   ```

7. **Verify the topic**:  
   ```bash
   $HOME/kafka/bin/kafka-topics.sh --list --bootstrap-server localhost:9092
   ```
   You should see `order-events`.

---

### **Part 2: Setting Up the Producer Microservice (`OrderService`)**

8. **Generate a new Spring Boot project.**  
   * Go to <https://start.spring.io>.  
   * **Spring Boot**: `3.4.5`  
   * **Group**: `com.microservices`  
   * **Artifact**: `order-service`  
   * **Dependencies**:  
     * *Spring Web*  
     * *Spring Cloud Stream*  
     * *Apache Kafka*  
   * Extract the project to a folder named **OrderService**.

9. **Import `OrderService` into your IDE.**

10. **Configure Spring Cloud Stream for Kafka.**  
    In `src/main/resources/application.properties`:
    ```properties
    spring.application.name=order-service
    spring.kafka.bootstrap-servers=localhost:9092

    # Bind the logical channel “output” to the Kafka topic
    spring.cloud.stream.bindings.output.destination=order-events
    ```
11. **Create the event model.**  
    `src/main/java/com/microservices/orderservice/OrderEvent.java`
    ```java
    package com.microservices.orderservice;

    public record OrderEvent(String orderId, String status) { }
    ```

12. **Create a message producer.**  
    `OrderProducer.java`
    ```java
    package com.microservices.orderservice;

    import org.springframework.cloud.stream.function.StreamBridge;
    import org.springframework.stereotype.Component;

    @Component
    public class OrderProducer {

        private final StreamBridge streamBridge;

        public OrderProducer(StreamBridge streamBridge) {
            this.streamBridge = streamBridge;
        }

        public void send(OrderEvent event) {
            // “output” matches the binding name in application.properties
            streamBridge.send("output", event);
        }
    }
    ```

13. **Expose a REST endpoint to publish events.**  
    `OrderController.java`
    ```java
    package com.microservices.orderservice;

    import org.springframework.web.bind.annotation.*;

    @RestController
    @RequestMapping("/orders")
    public class OrderController {

        private final OrderProducer producer;

        public OrderController(OrderProducer producer) {
            this.producer = producer;
        }

        @PostMapping
        public String create(@RequestBody OrderEvent event) {
            producer.send(event);
            return "Sent order event for ID: " + event.orderId();
        }
    }
    ```

14. **Run `OrderService`.**  
    ```bash
    ./mvnw spring-boot:run
    ```
    Boot output ends with:  
    ```
    Started OrderServiceApplication in … seconds (JVM running for …)
    ```

15. **Test the producer.**  
    ```bash
    curl -X POST       -H "Content-Type: application/json"       -d '{"orderId":"123","status":"CREATED"}'       http://localhost:8080/orders
    ```

---

### **Part 3: Setting Up the Consumer Microservice (`NotificationService`)**

16. **Generate a new Spring Boot project.**  
    * **Artifact**: `notification-service`  
    * Same dependencies as before.  
    * Extract to **NotificationService**.

17. **Import `NotificationService` into your IDE.**

18. **Configure Spring Cloud Stream for Kafka.**  
    `application.properties`
    ```properties
    spring.application.name=notification-service
    server.port=8081           # Avoid port clash
    spring.kafka.bootstrap-servers=localhost:9092

    # Bind the logical channel “input” to the same topic
    spring.cloud.stream.bindings.input.destination=order-events
    ```

19. **Create the consumer.**  
    `OrderEventConsumer.java`
    ```java
    package com.microservices.notificationservice;

    import java.util.function.Consumer;
    import org.springframework.context.annotation.Bean;
    import org.springframework.stereotype.Component;

    @Component
    public class OrderEventConsumer {

        @Bean
        public Consumer<OrderEvent> input() {
            return event -> System.out.println("📬 Received: " + event);
        }
    }
    ```

20. **Reuse the event model.**  
    `OrderEvent.java`
    ```java
    package com.microservices.notificationservice;

    public record OrderEvent(String orderId, String status) { }
    ```

21. **Run `NotificationService`.**  
    ```bash
    ./mvnw spring-boot:run
    ```
    You should see:  
    ```
    Started NotificationServiceApplication in … seconds
    ```

22. **Verify end‑to‑end flow.**  
    Send another event:
    ```bash
    curl -X POST       -H "Content-Type: application/json"       -d '{"orderId":"456","status":"CREATED"}'       http://localhost:8080/orders
    ```
    **NotificationService** console prints:  
    ```
    📬 Received: OrderEvent[orderId=456, status=CREATED]
    ```

---

## **Optional Exercises**

1. **Add a dead‑letter topic (DLT).**  
   ```properties
   spring.cloud.stream.kafka.bindings.input.consumer.enable-dlq=true
   ```

2. **Enrich `OrderEvent` with `timestamp` & `customerId`.**

3. **Horizontal scaling.**  
   Run multiple instances of `NotificationService` and observe Kafka’s partition‑based load‑balancing.

---

## **Conclusion**
You have:

* **Installed Kafka 3.7** locally.  
* **Built a producer** microservice that publishes events.  
* **Built a consumer** microservice that processes those events.  
* Demonstrated **asynchronous, decoupled communication** via Spring Cloud Stream and Apache Kafka on Spring Boot 3.4.5.
