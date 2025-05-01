# **Lab 14: Set Up a CI Pipeline with Jenkins for Automated Testing and Deployment of Microservices (Spring Boot 3.4.5)**

## **Objective**
Learn how to install and configure **Jenkins** to automate the build, testing, and deployment process for **Spring Boot 3.4.5** microservices. You will create a CI pipeline that builds two microservices (`UserService` and `OrderService`) and optionally deploys them.

---

## **Lab Steps**

### **Part 1: Installing Jenkins**

1. **Install Java (Jenkins prerequisite).**  
   - Confirm Java by running:  
     ```bash
     java -version
     ```  
   - If not found, install **JDK 17** (e.g., from [Adoptium](https://adoptium.net/)).  
   - Verify installation:  
     ```bash
     java -version
     ```

2. **Download Jenkins.**  
   - Visit [Jenkins Downloads](https://www.jenkins.io/download/) and get the installer for your OS (Windows, macOS, or Linux).

3. **Install Jenkins.**  
   - Run the installer and follow the wizard:  
     - **Linux**: Use your package manager or `.deb`/`.rpm` packages.  
     - **Windows**: Choose “**Run Jenkins as a Service**” option.  
     - **macOS**: Use the `.pkg` installer.

4. **Start Jenkins.**  
   - Jenkins usually starts on **port 8080**.  
   - **Linux/macOS**:  
     ```bash
     sudo systemctl start jenkins
     ```  
   - **Windows**: Start it from **Services**.

5. **Verify Jenkins is running.**  
   - Go to `http://localhost:8080`.

6. **Unlock Jenkins.**  
   - Follow the prompt to retrieve the initial admin password:  
     - **Linux/macOS**:  
       ```bash
       sudo cat /var/lib/jenkins/secrets/initialAdminPassword
       ```  
     - **Windows**:  
       ```cmd
       type C:\Jenkins\secrets\initialAdminPassword
       ```  
   - Copy/paste it into Jenkins.

7. **Install suggested plugins.**  
   - Jenkins will prompt you to install **Suggested plugins**. Do so.

8. **Create an admin user.**  
   - Finalize the setup by creating an admin account.

9. **Install Maven manually and extend `PATH`.**  
   - Download the binary ZIP from <https://maven.apache.org/download.cgi>.  
   - Unzip to e.g. `C:\Tools\apache-maven-3.9.9`.  
   - Add `C:\Tools\apache-maven-3.9.9\bin` to `PATH`:  
     - **Via Jenkins node properties → Environment variables**  
       - **Name** `PATH+MAVEN` **Value** `C:\Tools\apache-maven-3.9.9\bin`  
     - **—or globally** in *Windows System variables* (same dialog as above).  
   - **Restart the Jenkins Agent Windows service** (or reboot) so the new `PATH` is picked up.

---

### **Part 2: Preparing the Microservices**

10. **Ensure `UserService` and `OrderService` are in Git repositories.**  
    - Each microservice is a **Spring Boot 3.4.5** project pushed to a hosting platform like **GitHub**.

11. **Configure `pom.xml` for Jenkins.**  
    - In each microservice’s `pom.xml`, ensure the **Surefire plugin** is present for running tests:  
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
    - This ensures Jenkins can run `mvn test` or `mvn clean install`.

---

### **Part 3: Configuring Jenkins for CI**

12. **Create a Jenkins job for `UserService`.**  
    - From the Jenkins dashboard, click **New Item**.  
    - Name it `UserService-CI`, choose **Freestyle project**, and click **OK**.

13. **Set up the Git repository in Jenkins.**  
    - Under **Source Code Management**, select **Git** and enter the repository URL for `UserService`.

14. **Add a Maven build step.**  
    - In the **Build** section, add **Invoke top-level Maven targets**  
      ```
      clean install
      ```

15. **Save and run the job.**  
    - Click **Build Now** and confirm the build passes.

16. **Create a job for `OrderService`.**  
    - Repeat the same steps: name it `OrderService-CI`, configure the Git repo, and add `clean install`.

---

### **Part 4: Creating a Jenkins Pipeline**

17. **Install Pipeline plugins (Pipeline: API, and Pipeline: Stage View).**  
    - Navigate to **Manage Jenkins** → **Manage Plugins**, search for the plugins above, and install them.

18. **Create a new pipeline job.**  
    - In the dashboard, click **New Item**, select **Pipeline**, name it `Microservices-CI-Pipeline`, and click **OK**.

19. **Write the pipeline script.**  
    - Under **Pipeline** → **Pipeline script**:  
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
                      checkout([$class: 'GitSCM',
                                branches: [[name: '*/main']],
                                userRemoteConfigs: [[url: 'https://github.com/RushiSharma1999/user-service.git']]
                      ])
                      bat 'mvn -B clean install'
                  }
              }

              stage('Build OrderService') {
                  steps {
                      checkout([$class: 'GitSCM',
                                branches: [[name: '*/main']],
                                userRemoteConfigs: [[url: 'https://github.com/RushiSharma1999/order-service.git']]
                      ])
                      bat 'mvn -B clean install'
                  }
              }
          }
      }
      ```

20. **Save and run the pipeline.**  
    - Click **Build Now** and confirm both microservices build successfully in one pipeline run.

---

### **Part 5: Deployment (Optional)**

21. **Add a deployment stage.**  
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
    - Replace with your actual deployment logic (Docker push, Kubernetes apply, etc.).

22. **Configure automated triggers.**  
    - In GitHub (or another Git server), set up **webhooks** to trigger Jenkins on each push.

---

## **Optional Exercises**

1. **Integrate automated tests.** Add stages for integration or Docker-based tests.  
2. **Set up a multi-branch pipeline.** Use the **Multibranch Pipeline Plugin** to create jobs per branch automatically.  
3. **Add Docker integration.** Build Docker images in your pipeline and push them to a registry.  
4. **Monitor Jenkins builds.** Try plugins like **Build Monitor View** or **Blue Ocean** for rich visualization.

---

## **Conclusion**
By completing this lab, you have:

- **Installed and configured Jenkins** (running on port 8080).  
- **Set up Freestyle jobs** for each microservice (`UserService`, `OrderService`).  
- **Created a pipeline** that clones, builds, and optionally deploys both microservices in a single run.  
- **Learned** how Jenkins can automate your entire CI flow, ensuring code changes are consistently tested and deployed.

Enjoy building more advanced CI/CD pipelines!
