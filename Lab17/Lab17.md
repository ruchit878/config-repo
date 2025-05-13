# **Lab 17: Deploy and Orchestrate Microservices on Kubernetes with ConfigMaps (Spring Boot 3.4.5)**

## **Objective**

Learn how to deploy two **Spring Boot 3.4.5** microservices (`UserService` and `OrderService`) to **Kubernetes** using **ConfigMaps** for externalized configuration. You will containerize each service, load images into Minikube, and configure your Pods to use environment variables from ConfigMaps.

---

## **Installing Prerequisites**

| Tool               | Install Command                                                            | Purpose in this Lab                      |
| ------------------ | -------------------------------------------------------------------------- | ---------------------------------------- |
| **Java 17**        | `winget install --id=EclipseAdoptium.Temurin.17.JDK` (Windows)             | Build and run Spring Boot apps           |
| **Maven 3.9+**     | `winget install Apache.Maven` (Windows)                                    | Build Java projects                      |
| **Docker**         | [Download Docker Desktop](https://www.docker.com/products/docker-desktop/) | Required for Minikube with Docker driver |
| **Minikube**       | `choco install minikube` (Windows via Chocolatey)                          | Runs local Kubernetes cluster            |
| **kubectl**        | `winget install Kubernetes.kubectl` (Windows)                              | CLI to interact with Kubernetes cluster  |
| **IDE (IntelliJ)** | [Download IntelliJ IDEA](https://www.jetbrains.com/idea/download/)         | Develop and run Java code                |

> **What is Kubernetes?**
> **Kubernetes** is an open-source platform to automate deployment, scaling, and management of containerized applications. In this lab, we use it to deploy our microservices with environment-specific configuration.

---

## **Lab Steps**

### **Part 1: Installing Kubernetes and Minikube**

1. **Install Minikube.**
   Follow the link and download based on your OS:
   [https://minikube.sigs.k8s.io/docs/start/](https://minikube.sigs.k8s.io/docs/start/)

2. **Install Kubectl.**
   Use: [https://kubernetes.io/docs/tasks/tools/](https://kubernetes.io/docs/tasks/tools/)

3. **Verify Minikube Installation.**

   ```bash
   minikube version
   ```

   **Expected Output:**

   ```bash
   minikube version: v1.xx.x
   ```

4. **Start Minikube.**

   ```bash
   minikube start --driver=docker
   ```

   **Troubleshooting:** If Docker is not running, start Docker Desktop first.

5. **Check Kubernetes Setup.**

   ```bash
   kubectl cluster-info
   minikube status
   ```

   **Expected Output:**

   ```bash
   Kubernetes control plane is running...
   host: Running
   kubelet: Running
   ```

---

### **Part 2: Preparing Microservices**

> **What is Spring Boot?**
> **Spring Boot** is a Java framework used to build microservices easily. Here, we use it to create `UserService` and `OrderService`.

6. **Ensure Docker Images.**
   If you have completed Lab 16, you should already have the following images:

   ```bash
   docker images
   ```

   **Expected Output:**

   ```bash
   user-service   1.0
   order-service  1.0
   ```

7. **Load Images into Minikube.**

   ```bash
   minikube image load user-service:1.0
   minikube image load order-service:1.0
   ```

8. **Set Up `application.properties` Files.**
   **Path:** `src/main/resources/application.properties`

   * **UserService:**

     ```properties
     server.port=${SERVER_PORT:8081}
     app.name=${APP_NAME:user-service}
     database.url=${DATABASE_URL:jdbc:mysql://localhost:3306/userdb}
     ```
   * **OrderService:**

     ```properties
     server.port=${SERVER_PORT:8082}
     app.name=${APP_NAME:order-service}
     user.service.url=${USER_SERVICE_URL:http://user-service:8081/users}
     ```

> **What are ConfigMaps?**
> Kubernetes **ConfigMaps** allow you to externalize configuration from code. They provide environment-specific variables that your containers can use.

---

### **Part 3: Creating ConfigMaps**

9. **Create ConfigMap for UserService.**
   **File:** `user-service-configmap.yaml`

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: user-service-config
   data:
     SERVER_PORT: "8081"
     APP_NAME: "user-service"
     DATABASE_URL: "jdbc:mysql://user-database:3306/userdb"
   ```

10. **Create ConfigMap for OrderService.**
    **File:** `order-service-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
data:
  SERVER_PORT: "8082"
  APP_NAME: "order-service"
  USER_SERVICE_URL: "http://user-service:8081/users"
```

11. **Apply ConfigMaps.**

    ```bash
    kubectl apply -f user-service-configmap.yaml
    kubectl apply -f order-service-configmap.yaml
    ```

    **Expected Output:**

    ```bash
    configmap/user-service-config created
    configmap/order-service-config created
    ```

---

### **Part 4: Writing Kubernetes Manifests**

> **What are Deployments and Services?**
> A **Deployment** runs your app in Pods. A **Service** exposes your app so other services or users can access it.

12. **Create UserService Deployment.**
    **File:** `user-service-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
      - name: user-service
        image: user-service:1.0
        ports:
          - containerPort: 8081
        envFrom:
          - configMapRef:
              name: user-service-config
```

13. **Create UserService Service.**
    **File:** `user-service-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
  - protocol: TCP
    port: 8081
    targetPort: 8081
  type: ClusterIP
```

14. **Create OrderService Deployment and Service.**

* **Deployment File:** `order-service-deployment.yaml`
* **Service File:** `order-service-service.yaml`

> (Use similar structure as above with different ports, image, and ConfigMap.)

---

### **Part 5: Deploying to Kubernetes**

15. **Apply Deployments and Services.**

```bash
kubectl apply -f user-service-deployment.yaml
kubectl apply -f user-service-service.yaml
kubectl apply -f order-service-deployment.yaml
kubectl apply -f order-service-service.yaml
```

16. **Verify Resources.**

```bash
kubectl get pods
kubectl get services
```

**Expected Output:**

```bash
NAME              READY   STATUS    RESTARTS   AGE
user-service...   1/1     Running   0          5s
```

---

### **Part 6: Testing and Scaling**

17. **Test Using Port Forwarding.**

```bash
kubectl port-forward service/order-service 8082:8082
```

Open browser:

```
http://localhost:8082/orders
```

18. **Scale OrderService Deployment.**

```bash
kubectl scale deployment order-service --replicas=3
kubectl get pods -l app=order-service
```

19. **Clean Up Resources.**

```bash
kubectl delete deployment user-service order-service
kubectl delete service user-service order-service
```

---

## **Conclusion** 🎉

You:

* Installed Minikube and Kubernetes tools.
* Created Docker images and deployed two Spring Boot microservices.
* Used ConfigMaps to externalize environment variables.
* Verified the services and scaled deployments.

**Next Steps:**
Try adding **Horizontal Pod Autoscaling (HPA)** to auto-scale based on CPU usage!
