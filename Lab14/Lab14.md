
# **Lab 14: Set Up a CI Pipeline with Jenkins for Automated Testing and Deployment of Microservices (Spring Boot 3.4.5)**

## ✅ Objective
Learn how to install and configure **Jenkins** to automate the build, testing, and deployment process for **Spring Boot 3.4.5** microservices. You will create a CI pipeline that builds two microservices (`UserService` and `OrderService`) and optionally deploys them.

---

## ⚙️ Installing Prerequisites

| Tool         | Install Command (Windows ▶ macOS ▶ Ubuntu) | Purpose in this Lab                        |
|--------------|---------------------------------------------|--------------------------------------------|
| **Java 17+** | `winget install EclipseAdoptium.Temurin.17.JDK` <br> `brew install openjdk@17` <br> `sudo apt install openjdk-17-jdk -y` | Required to run Jenkins and Spring Boot    |
| **Maven 3.9+** | `winget install Apache.Maven` <br> `brew install maven` <br> `sudo apt install maven -y` | Build and test Java microservices          |
| **Jenkins**  | Download from [jenkins.io/download](https://www.jenkins.io/download/) | Automate CI/CD workflow                    |
| **Git**      | `winget install Git.Git` <br> `brew install git` <br> `sudo apt install git -y` | Clone projects from GitHub                 |

---

## 🚀 Beginner Notes
> **What is Jenkins?**  
> Jenkins is an open-source automation server. In this lab, you’ll use it to automatically test and build your Spring Boot microservices when you push code.

---

## 🧪 Lab Steps

### Part 1: Installing Jenkins

1. **Check if Java is installed**
   ```bash
   java -version
   ```
   **✅ Expected output**
   ```
   openjdk version "17.x.x"
   ```

2. **Install Jenkins**
   - Go to [Jenkins Downloads](https://www.jenkins.io/download/).
   - Download and install for your OS.
   - **Windows**: Select “Run Jenkins as a Service”.
   - **macOS/Linux**: Follow installer instructions.

3. **Start Jenkins**
   - **Windows**: Open "Services" → Start Jenkins
   - **macOS/Linux**:
     ```bash
     sudo systemctl start jenkins
     ```

4. **Open Jenkins in your browser**
   - Navigate to:
     ```
     http://localhost:8080
     ```

5. **Unlock Jenkins**
   - Terminal (Linux/macOS):
     ```bash
     sudo cat /var/lib/jenkins/secrets/initialAdminPassword
     ```
   - CMD (Windows):
     ```cmd
     type C:\Jenkins\secrets\initialAdminPassword
     ```

6. **Install Suggested Plugins**
   - Click the default option: **Install Suggested Plugins**

7. **Create Jenkins Admin User**
   - Fill in the admin form to complete setup.

8. **Install Maven & Set PATH**
   - Unzip Maven into `C:\Toolspache-maven-3.9.9`
   - Add the path in Jenkins:
     ```
     PATH+MAVEN = C:\Toolspache-maven-3.9.9in
     ```

---

### Part 2: Prepare Spring Boot Microservices

> **What is Maven?**  
> Maven is a tool that compiles, tests, and packages your Java code. Jenkins uses it to automate microservice builds.

9. **Clone or Push Both Microservices to GitHub**
   - Make sure `UserService` and `OrderService` are separate GitHub repos.

10. **Update `pom.xml` in both services**

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

💡 **TIP**: Avoid manually adding `<version>` tags for Spring dependencies. Use Spring Cloud BOM to manage versions.

---

### Part 3: Create Jenkins Jobs

11. **Freestyle Job for `UserService`**
   - Dashboard → **New Item**
   - Name: `UserService-CI` → Select **Freestyle project**

12. **Add GitHub Repo**
   - Under **Source Code Management** → Git
   - Paste your repository URL

13. **Add Maven Build Step**
   - Under **Build** → `clean install`

14. **Click Build Now**

📟 **Expected Output**
```
BUILD SUCCESS
```

15. **Repeat Steps for `OrderService`**
   - Name it `OrderService-CI`

---

### Part 4: Pipeline Job

> **What is a Pipeline?**  
> A Jenkins pipeline defines your entire CI/CD process as code. It helps manage multiple builds more efficiently.

16. **Install Plugins**
   - **Manage Jenkins** → **Manage Plugins** → Install:
     - Pipeline: API
     - Pipeline: Stage View

17. **Create Pipeline Job**
   - **New Item** → **Pipeline** → Name: `Microservices-CI-Pipeline`

18. **Add the following script** under **Pipeline Script**:

📄 `Microservices-CI-Pipeline/Jenkinsfile`:
```groovy
pipeline {
  agent any

  stages {
    stage('Check cmd') {
      steps {
        bat 'echo CMD is working'
      }
    }
    stage('Build UserService') {
      steps {
        checkout([$class: 'GitSCM', branches: [[name: '*/main']],
        userRemoteConfigs: [[url: 'https://github.com/RushiSharma1999/user-service.git']]])
        bat 'mvn -B clean install'
      }
    }
    stage('Build OrderService') {
      steps {
        checkout([$class: 'GitSCM', branches: [[name: '*/main']],
        userRemoteConfigs: [[url: 'https://github.com/RushiSharma1999/order-service.git']]])
        bat 'mvn -B clean install'
      }
    }
  }
}
```

19. **Run the Pipeline**

✅ **Expected Output**
```
Stage: Build UserService
BUILD SUCCESS

Stage: Build OrderService
BUILD SUCCESS
```

---

### Part 5: Optional Deployment Stage

20. **Add deployment to your pipeline** (edit Jenkinsfile):

```groovy
stage('Deploy to Server') {
  steps {
    script {
      sh 'scp user-service/target/*.jar user@server:/path/to/deploy/'
      sh 'scp order-service/target/*.jar user@server:/path/to/deploy/'
    }
  }
}
```

---

### GitHub Webhook Setup

> Go to your GitHub repo → **Settings → Webhooks**  
> Add URL: `http://<your-public-jenkins-url>/github-webhook/`  
> Events: "Just the push event"

---

## 🔧 Troubleshooting Tips

- Jenkins not loading? Check `http://localhost:8080`
- Maven command not found? Confirm it's in your `PATH`
- Git repo not cloning? Double-check SSH keys or public access
- "BUILD FAILURE"? Expand console output to find root cause

---

## 💡 Scaling Tip

To build more microservices, duplicate the stages in the pipeline and update the Git URLs and service names.

---

## 🏁 Conclusion

You’ve just built a full CI pipeline that:

- Installs Jenkins and configures Maven
- Runs build and test jobs for Spring Boot microservices
- Uses a scripted Jenkins pipeline for automation
- (Optionally) deploys to a remote server

🎉 Try expanding your pipeline with Docker, test reports, or Blue Ocean UI!

