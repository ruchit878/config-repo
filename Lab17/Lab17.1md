# **Lab 17: Deploy & Orchestrate Microservices on Kubernetes (🎉 Spring Boot 3.4.5)**

> Spin up a local **Kubernetes** cluster with **Minikube**, containerise two Spring Boot services, inject settings with **ConfigMaps**, scale them, and clean up — all on your laptop.

---

## Installing Prerequisites 🚀

| Tool                         | One‑line install<br>(Windows → `winget` · macOS → `brew` · Ubuntu → `apt`)                         | Purpose in this Lab           |
| ---------------------------- | -------------------------------------------------------------------------------------------------- | ----------------------------- |
| **Java 17+**                 | `winget install EclipseAdoptium.Temurin.17.JDK -h`                                                 | Compile & run Spring apps     |
| **Maven 3.9+**               | `winget install Apache.Maven`                                                                      | Build JARs & container images |
| **Docker Desktop**           | [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/) | Build + run containers        |
| **Minikube 1.33+**           | `winget install Kubernetes.Minikube`                                                               | Local single‑node cluster     |
| **kubectl**                  | `winget install Kubernetes.kubectl`                                                                | Talk to the cluster API       |
| **IDE** (IntelliJ / VS Code) | Download from vendor                                                                               | Edit Java code                |

> **Why Minikube?** It gives you a full Kubernetes cluster locally so you can test manifests exactly as they will run in prod.

---

## Lab Repo (two services in one repo)

```bash
# ➊ Grab the sample code
 git clone https://github.com/your‑org/users‑orders‑demo.git
 cd users‑orders‑demo
```

Folder structure (important!):

```text
users‑orders‑demo/
 ├─ user‑service/
 ├─ order‑service/
 └─ k8s/                 # we will create manifests here
```

---

## Part 1 — Start the Cluster

1. **Launch Minikube.**

   ```bash
   minikube start --driver=docker
   ```

   **Expected final line**

   ```text
   Done! kubectl is now configured to use "minikube" cluster.
   ```

2. **Verify the node is ready.**

   ```bash
   kubectl get nodes
   ```

   ```text
   NAME       STATUS   ROLES           AGE   VERSION
   minikube   Ready    control-plane   30s   v1.30.0
   ```

---

## Part 2 — Build & Load Container Images

3. **Build `user-service` image.**

   ```bash
   cd user-service
   mvn -ntp spring-boot:build-image \
       -Dspring-boot.build-image.imageName=user-service:1.0
   ```

   *Success tail*:

   ```text
   Successfully built image 'docker.io/library/user-service:1.0'
   ```

4. **Build `order-service` image.**

   ```bash
   cd ../order-service
   mvn -ntp spring-boot:build-image \
       -Dspring-boot.build-image.imageName=order-service:1.0
   ```

5. **Load images into Minikube’s registry.**

   ```bash
   minikube image load user-service:1.0
   minikube image load order-service:1.0
   ```

   Expected: `Loaded image: user-service:1.0` (and similar for order).

> **Troubleshoot:** If `image load` fails, restart Docker Desktop and rerun the command.

---

## Part 3 — Create ConfigMaps

*All YAML files live in `k8s/`.*

6. **`k8s/user-service-configmap.yaml`**

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

7. **`k8s/order-service-configmap.yaml`**

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

8. **Apply them.**

   ```bash
   kubectl apply -f k8s/user-service-configmap.yaml
   kubectl apply -f k8s/order-service-configmap.yaml
   ```

   Output:

   ```text
   configmap/user-service-config created
   configmap/order-service-config created
   ```

---

## Part 4 — Deploy the Services

9. **`k8s/user-service-deployment.yaml`**

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

10. **`k8s/user-service-service.yaml`**

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

11. **`k8s/order-service-deployment.yaml`**

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

12. **`k8s/order-service-service.yaml`**

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

13. **Apply all four manifests.**

    ```bash
    kubectl apply -f k8s/
    ```

    You should see four *created* lines.

14. **Watch Pods come up.**

    ```bash
    kubectl get pods -w
    ```

    Wait until all are `Running`.

---

## Part 5 — Smoke‑Test + Scale

15. **Port‑forward OrderService.**

    ```bash
    kubectl port-forward svc/order-service 8082:8082
    ```

    In another terminal:

    ```bash
    curl http://localhost:8082/orders
    ```

    *Expected (sample):*

    ```json
    [
      {"orderId":1,"user":"Alice"}
    ]
    ```

16. **Scale OrderService to 3 pods.**

    ```bash
    kubectl scale deploy/order-service --replicas=3
    kubectl get pods -l app=order-service
    ```

    All three pods should reach `Running`.

---

## Part 6 — Clean‑Up 🧹

17. **Delete resources.**

    ```bash
    kubectl delete -f k8s/
    minikube stop
    ```

---

## Optional Challenges 🌟

1. **Horizontal Pod Autoscaler (HPA)** – auto‑scale `order-service` on CPU.
2. **Rolling Config Updates** – edit a ConfigMap, then apply + rollout restart the deployment.
3. **Chaos Test** – `kubectl delete pod` to watch self‑healing.

---

## Finished ✅

You have:

* Built two Spring Boot images.
* Loaded them into Minikube.
* Injected configuration using ConfigMaps.
* Deployed, tested & scaled microservices on Kubernetes.

High‑five – you just orchestrated microservices like a pro! ✋
