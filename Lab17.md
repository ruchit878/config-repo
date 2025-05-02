# **Lab 17: Deploy and Orchestrate Microservices on Kubernetes with ConfigMaps (Spring Boot 3.4.5)**

## **Objective**
Learn how to deploy two **Spring Boot 3.4.5** microservices (`UserService` and `OrderService`) to **Kubernetes** using **ConfigMaps** for externalized configuration. You will containerize each service, load images into Minikube, and configure your Pods to use environment variables from ConfigMaps.

---

## **Lab Steps**

### **Part 1: Installing Kubernetes and Minikube**

1. **Install Minikube.**
   - Visit [Minikube Installation](https://minikube.sigs.k8s.io/docs/start/) and follow the OS-specific instructions.
   - Make it available in your `PATH`.

2. **Install Kubectl.**
   - Download **kubectl** from [Kubernetes Tools](https://kubernetes.io/docs/tasks/tools/).
   - Make it available in your `PATH`.

3. **Verify Minikube Installation.**
   - Run:
     ```bash
     minikube version
     ```
   - Confirm the installed version.

4. **Start Minikube.**
   - Launch a cluster using:
     ```bash
     minikube start --driver=docker
     ```

5. **Check Kubernetes Setup.**
   - Verify the cluster information:
     ```bash
     kubectl cluster-info
     ```
   - Check Minikube status:
     ```bash
     minikube status
     ```

---

### **Part 2: Preparing Microservices**

6. **Ensure Docker Images for `UserService` and `OrderService`.**
   - From **Lab 16**, you should have `user-service:1.0` and `order-service:1.0`.
   - Load these images into Minikube’s local registry:
     ```bash
     minikube image load user-service:1.0
     minikube image load order-service:1.0
     ```

7. **Externalize Configurations with `application.properties`.**
   - For **UserService**, e.g. `src/main/resources/application.properties` might have:
     ```properties
     server.port=${SERVER_PORT:8081}
     app.name=${APP_NAME:user-service}
     database.url=${DATABASE_URL:jdbc:mysql://localhost:3306/userdb}
     ```
   - For **OrderService**, e.g.:
     ```properties
     server.port=${SERVER_PORT:8082}
     app.name=${APP_NAME:order-service}
     user.service.url=${USER_SERVICE_URL:http://user-service:8081/users}
     ```
   - The placeholders (`${...}`) allow environment variables to override defaults.

---

### **Part 3: Creating ConfigMaps**

8. **Create a ConfigMap for `UserService`.**
   - **user-service-configmap.yaml**:
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

9. **Create a ConfigMap for `OrderService`.**
   - **order-service-configmap.yaml**:
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

10. **Apply ConfigMaps to the cluster.**
    ```bash
    kubectl apply -f user-service-configmap.yaml
    kubectl apply -f order-service-configmap.yaml
    ```
    - This stores the configuration in Kubernetes.

---

### **Part 4: Writing Kubernetes Manifests**

11. **Deployment for `UserService`.**
    - **user-service-deployment.yaml**:
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
    - `envFrom` uses `user-service-config` for environment variables.

12. **Service for `UserService`.**
    - **user-service-service.yaml**:
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

13. **Deployment for `OrderService`.**
    - **order-service-deployment.yaml**:
      ```yaml
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: order-service
      spec:
        replicas: 2
        selector:
          matchLabels:
            app: order-service
        template:
          metadata:
            labels:
              app: order-service
          spec:
            containers:
            - name: order-service
              image: order-service:1.0
              ports:
                - containerPort: 8082
              envFrom:
                - configMapRef:
                    name: order-service-config
      ```
    - Also references its own ConfigMap.

14. **Service for `OrderService`.**
    - **order-service-service.yaml**:
      ```yaml
      apiVersion: v1
      kind: Service
      metadata:
        name: order-service
      spec:
        selector:
          app: order-service
        ports:
        - protocol: TCP
          port: 8082
          targetPort: 8082
        type: ClusterIP
      ```

---

### **Part 5: Deploying to Kubernetes**

15. **Apply `UserService` Deployment/Service.**
    ```bash
    kubectl apply -f user-service-deployment.yaml
    kubectl apply -f user-service-service.yaml
    ```

16. **Apply `OrderService` Deployment/Service.**
    ```bash
    kubectl apply -f order-service-deployment.yaml
    kubectl apply -f order-service-service.yaml
    ```

17. **Verify Pods and Services.**
    - Check Pods:
      ```bash
      kubectl get pods
      ```
    - Check Services:
      ```bash
      kubectl get services
      ```

---

### **Part 6: Testing and Scaling**

18. **Test Services with Port-Forwarding.**
    - For `OrderService`, for example:
      ```bash
      kubectl port-forward service/order-service 8082:8082
      ```
    - Then open:
      ```
      http://localhost:8082/orders
      ```
    - The service should call `UserService` by name `user-service` inside the cluster.

19. **Scale the `OrderService` deployment.**
    - e.g., to 3 replicas:
      ```bash
      kubectl scale deployment order-service --replicas=3
      ```
    - Check new pods:
      ```bash
      kubectl get pods -l app=order-service
      ```

20. **Clean Up.**
    - Delete the Deployments and Services:
      ```bash
      kubectl delete deployment user-service order-service
      kubectl delete service user-service order-service
      ```

---

## **Optional Exercises**

1. **Enable Horizontal Pod Autoscaling (HPA).**
   - Create an HPA for `OrderService` that scales based on CPU usage.

2. **Add ConfigMap Updates.**
   - Modify the ConfigMaps to change environment variables and see if the pods pick up changes (requires re-deployment or additional strategies).

3. **Simulate Failures.**
   - Kill a running pod (`kubectl delete pod <pod-name>`) and observe Kubernetes automatically recreating it.

---

## **Conclusion**
In this lab, you have:
- **Deployed** two microservices (`UserService`, `OrderService`) to Kubernetes with **ConfigMaps** for externalized config.
- **Managed** container images via **Minikube** (`minikube image load`).
- **Demonstrated** how to scale and test them with port-forwarding.
- **Simplified updates** to environment variables with ConfigMaps, providing more flexibility and best practices for dynamic configurations in a Kubernetes environment!

Enjoy orchestrating microservices at scale with Kubernetes!
