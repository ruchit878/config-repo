# **Lab 14: Set Up a CI Pipeline with Jenkins for Automated Testing and Deployment of Microservices (Spring Boot 3.4.5)**

## **Objective**
Learn how to install and configure **Jenkins** to automate the build, testing, and deployment process for **Spring Boot 3.4.5** microservices. You will create a CI pipeline that builds two microservices (`UserService` and `OrderService`) and optionally deploys them.

---

## **Lab Steps**

### **Part 1: Installing Jenkins**

1. **Install Java (Jenkins prerequisite).**
   - Confirm Java by running:
     ```cmd
     java -version
     ```
     ✅ Expected output:
     ```
     openjdk version "17.x.x"
     ```
   - If not found, install **JDK 17** from [Adoptium](https://adoptium.net/).

2. **Download Jenkins.**
   - Visit [Jenkins Downloads](https://www.jenkins.io/download/) and get the installer for **Windows**.

3. **Install Jenkins.**
   - Run the installer and choose “**Run Jenkins as a Service**”.

4. **Start Jenkins.**
   - Open **Services** from the Start Menu and start the Jenkins service.

5. **Verify Jenkins is running.**
   - Open browser and go to:
     ```
     http://localhost:8080
     ```

6. **Unlock Jenkins.**
   - Retrieve the initial admin password:
     ```cmd
     type C:\Jenkins\secrets\initialAdminPassword
     ```
   - Copy/paste it into Jenkins.

7. **Install suggested plugins.**
   - Jenkins will prompt you to install **Suggested plugins**. Do so.

8. **Create an admin user.**
   - Finalize the setup by creating an admin account.

> **What is Jenkins?**  
> Jenkins is an open-source automation server. In this lab, you’ll use it to automatically test and build your Spring Boot microservices when you push code.

---

### **Part 2: Preparing the Microservices**

> ⚙️ **Creating Microservices from start.spring.io**
>
> Create two projects from [https://start.spring.io](https://start.spring.io) with the following settings:
>
> | Service        | Spring Boot | Group ID            | Artifact ID      | Dependencies                     |
> |----------------|-------------|----------------------|------------------|----------------------------------|
> | UserService    | 3.4.5       | `com.microservices`  | `user-service`   | Spring Web, Spring Boot DevTools |
> | OrderService   | 3.4.5       | `com.microservices`  | `order-service`  | Spring Web, Spring Boot DevTools |

9. **Push services to GitHub.**
   - Make sure `UserService` and `OrderService` are hosted in GitHub repositories.

10. **Verify `pom.xml` contains Surefire plugin.**
📄 `UserService/pom.xml` and `OrderService/pom.xml`:
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.0.0</version>
        </plugin>
    </plugins>
</build>
```
💡 Note: Do not manually add versions for Spring Boot dependencies. Use Spring Initializr's managed versions.

---

### **Part 3: Configuring Jenkins for CI**

11. **Create a Jenkins job for `UserService`.**
    - From the Jenkins dashboard, click **New Item**.
    - Name it `UserService-CI`, choose **Freestyle project**, and click **OK**.

12. **Set up the Git repository in Jenkins.**
    - Under **Source Code Management**, select **Git**.
    - Under **Branch Specifier**, Enter exactly ***/main**
    - Enter the GitHub repository URL for `UserService`.

13. **Add a Maven build step.**
    - In the **Build** section, add **Invoke top-level Maven targets**.
    - Goals:
      ```
      clean install
      ```

14. **Save and run the job.**
    - Click **Build Now**.
    - ✅ Expected output:
      ```
      BUILD SUCCESS
      ```

15. **Repeat for `OrderService`.**
    - Name it `OrderService-CI`.

---

### **Part 4: Creating a Jenkins Pipeline**

16. **Install the Pipeline plugin if not installed.**
    - Go to **Manage Jenkins** → **Manage Plugins**.
    - Search for and install **Pipeline**.

17. **Create a pipeline job.**
    - Dashboard → **New Item** → Name: `Microservices-CI-Pipeline` → Select **Pipeline** → Click **OK**.

18. **Add the pipeline script.**
    - Under **Pipeline** → **Pipeline script**:
```groovy
pipeline {
    agent any
    stages {
        stage('Build UserService') {
            steps {
                script {
                    git branch:'main', url: '<UserService Git URL>'
                    dir('user-service') {
                        bat 'mvn clean install'
                    }
                }
            }
        }
        stage('Build OrderService') {
            steps {
                script {
                    git branch:'main', url: '<OrderService Git URL>'
                    dir('order-service') {
                        bat 'mvn clean install'
                    }
                }
            }
        }
    }
}

```

19. **Run the pipeline.**
    - Click **Build Now**.
    - ✅ Expected output:
      ```
      BUILD SUCCESS
      ```

---

### **Part 5: Deployment (Optional)**

20. **Configure webhook triggers.**
    - In your GitHub repository → **Settings** → **Webhooks** → Add URL to Jenkins webhook endpoint.

---

## **Conclusion**
You have:
- Installed Jenkins on **Windows**
- Created Freestyle and Pipeline jobs
- Automated build and optional deployment of two Spring Boot microservices

🎉 Your first CI pipeline is now live!
