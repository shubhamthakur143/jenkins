# EasyCRUD – CI/CD Pipeline with Jenkins, SonarQube & AWS S3

This repository contains the **EasyCRUD** (Student Registration) Spring Boot application, along with a fully automated CI/CD pipeline built using **Jenkins**, **SonarQube** (for code quality/static analysis), and **AWS S3** (as the artifact delivery destination).

---

## 📋 Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Step 1: Launch EC2 Instance](#step-1-launch-ec2-instance)
- [Step 2: Install Jenkins](#step-2-install-jenkins)
- [Step 3: Install SonarQube (Community Build)](#step-3-install-sonarqube-community-build)
- [Step 4: Run SonarQube as a systemd Service](#step-4-run-sonarqube-as-a-systemd-service)
- [Step 5: Configure SonarQube Webhook](#step-5-configure-sonarqube-webhook)
- [Step 6: Connect Jenkins to SonarQube](#step-6-connect-jenkins-to-sonarqube)
- [Step 7: Configure GitHub Webhook](#step-7-configure-github-webhook)
- [Step 8: Configure AWS S3 Access](#step-8-configure-aws-s3-access)
- [Step 9: The Jenkinsfile](#step-9-the-jenkinsfile)
- [Pipeline Stages Explained](#pipeline-stages-explained)
- [Troubleshooting](#troubleshooting)
- [Tech Stack](#tech-stack)

---

## Architecture Overview

```
GitHub Push
     │
     ▼
GitHub Webhook ──► Jenkins (EC2, port 8080)
                       │
                       ▼
              ┌─────────────────┐
              │   Pull-Job      │  → clone repo
              ├─────────────────┤
              │   Build-Job     │  → mvn clean package
              ├─────────────────┤
              │   Test-Job      │  → SonarQube static analysis
              ├─────────────────┤
              │  Quality-Gate   │  → wait for SonarQube verdict
              ├─────────────────┤
              │  Delivery-Job   │  → upload JAR to AWS S3
              └─────────────────┘
                       │
                       ▼
              SonarQube (EC2, port 9000)
                       │
                       ▼
              AWS S3 Bucket (artifact storage)
```

---

## Prerequisites

- An AWS EC2 instance (Ubuntu, recommended `t2.medium` or larger — SonarQube is memory-hungry)
- Java 21 (JDK, **not** JRE)
- PostgreSQL (for SonarQube's database)
- Maven (for building the Spring Boot backend)
- An AWS S3 bucket for storing build artifacts
- A GitHub repository with the source code

---

## Step 1: Launch EC2 Instance

1. Launch an Ubuntu EC2 instance.
2. Configure the **Security Group** with the following inbound rules:

   | Type       | Protocol | Port | Source      | Purpose                  |
   |------------|----------|------|-------------|---------------------------|
   | Custom TCP | TCP      | 8080 | 0.0.0.0/0   | Jenkins UI + webhooks     |
   | Custom TCP | TCP      | 9000 | 0.0.0.0/0   | SonarQube UI              |
   | SSH        | TCP      | 22   | My IP       | SSH access                |

3. SSH into the instance:
   ```bash
   ssh -i your-key.pem ubuntu@<EC2-public-IP>
   ```

---

## Step 2: Install Jenkins

```bash
# Install Java (Jenkins requires Java 17+)
sudo apt update
sudo apt install openjdk-21-jdk -y

# Add Jenkins repository
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install jenkins -y

sudo systemctl enable jenkins
sudo systemctl start jenkins
```

Access Jenkins at `http://<EC2-public-IP>:8080` and complete the setup wizard using the initial admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Install the following Jenkins plugins (Manage Jenkins → Plugins):
- **SonarQube Scanner**
- **Pipeline**
- **Git**

---

## Step 3: Install SonarQube (Community Build)

> ⚠️ SonarQube's free tier is now called **Community Build** and uses calendar-based versioning (e.g. `26.6.0.123539`).

```bash
# 1. Create a dedicated non-root user (Elasticsearch refuses to run as root)
sudo useradd -r -M sonar

# 2. Download and unzip SonarQube
cd /tmp
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-26.6.0.123539.zip
unzip sonarqube-26.6.0.123539.zip
sudo mv sonarqube-26.6.0.123539 /opt/sonarqube

# 3. Set ownership
sudo chown -R sonar:sonar /opt/sonarqube

# 4. Increase OS limits required by Elasticsearch (bundled with SonarQube)
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf

# 5. Set up PostgreSQL database
sudo -u postgres psql -c "CREATE DATABASE sonarqube;"
sudo -u postgres psql -c "CREATE USER sonar_user WITH PASSWORD 'your_password';"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE sonarqube TO sonar_user;"

# 6. Configure sonar.properties
sudo nano /opt/sonarqube/conf/sonar.properties
```

Add/uncomment in `sonar.properties`:
```properties
sonar.jdbc.username=sonar_user
sonar.jdbc.password=your_password
sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube
```

Start SonarQube manually (for a first test):
```bash
sudo -u sonar /opt/sonarqube/bin/linux-x86-64/sonar.sh start
tail -f /opt/sonarqube/logs/sonar.log
```

Once you see `SonarQube is operational`, visit `http://<EC2-public-IP>:9000` (default login: `admin` / `admin`).

---

## Step 4: Run SonarQube as a systemd Service

To keep SonarQube running automatically across reboots, create a systemd unit:

```bash
sudo nano /etc/systemd/system/sonarqube.service
```

```ini
[Unit]
Description=SonarQube service
After=syslog.target network.target postgresql.service

[Service]
Type=forking
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
User=sonar
Group=sonar
Restart=always
LimitNOFILE=131072
LimitNPROC=8192

[Install]
WantedBy=multi-user.target
```

> **Important:** `User=` and `Group=` in this file must match the actual Linux user that owns `/opt/sonarqube` (in this setup, `sonar`). A mismatch causes the service to fail repeatedly and hit systemd's `start-limit-hit` error.

Enable and start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable sonarqube
sudo systemctl start sonarqube
sudo systemctl status sonarqube
```

---

## Step 5: Configure SonarQube Webhook

This lets SonarQube notify Jenkins the moment an analysis is complete, so the pipeline's `waitForQualityGate` step doesn't hang waiting indefinitely.

1. In SonarQube: **Administration → Configuration → Webhooks → Create**
2. Fill in:
   - **Name:** `jenkins`
   - **URL:** `http://<EC2-public-IP>:8080/sonarqube-webhook/`
3. Save.

> **Common mistake:** typing the URL with a duplicated scheme (`http://http://...`) or a stray slash before the port (`...232.37/:8080`). Double-check the URL is exactly `http://<ip>:8080/sonarqube-webhook/`.

---

## Step 6: Connect Jenkins to SonarQube

1. Generate a token in SonarQube: **Administration → Security → Users → Tokens**
2. In Jenkins: **Manage Jenkins → Credentials** → add a new **Secret text** credential containing that token. Give it an ID (e.g. `cred`).
3. In Jenkins: **Manage Jenkins → System → SonarQube servers**:
   - **Name:** `sonar-server` (must exactly match what's used in the Jenkinsfile)
   - **Server URL:** `http://localhost:9000` (or the EC2 IP if Jenkins/SonarQube are on separate hosts)
   - **Server authentication token:** select the credential added above

---

## Step 7: Configure GitHub Webhook

This triggers Jenkins automatically on every `git push`.

1. In your GitHub repo: **Settings → Webhooks → Add webhook**
2. **Payload URL:** `http://<EC2-public-IP>:8080/github-webhook/`
3. **Content type:** `application/json`
4. **Which events:** Just the `push` event
5. Save.

> Use the EC2 **public** IP, not the private IP (`172.31.x.x`) — GitHub's servers live outside your VPC and cannot reach private addresses.

Verify delivery under **Recent Deliveries** in the webhook settings — a `200` response confirms Jenkins received it successfully.

---

## Step 8: Configure AWS S3 Access

The final pipeline stage uploads the built JAR to S3. Grant the EC2 instance (and therefore Jenkins) access via an **IAM Role** attached to the instance (preferred over hardcoding credentials):

1. Create an IAM Role with an `AmazonS3FullAccess` (or a scoped custom policy) permission.
2. Attach the role to the EC2 instance running Jenkins.
3. Verify access:
   ```bash
   sudo -u jenkins aws s3 ls s3://s3-delivery-bkt/
   ```

---

## Step 9: The Jenkinsfile

```groovy
pipeline {
    agent any
    stages {
        stage ('Pull-Job') {
            steps {
                git branch: 'main', url: 'https://github.com/shubhamthakur143/EasyCRUD.git'
            }
        }
        stage ('Build-Job') {
            steps {
                sh ''' cd backend
                mvn clean package -DskipTests'''
            }
        }
        stage ('Test-Job') {
            steps {
                timeout(5) {
                    withSonarQubeEnv(installationName: 'sonar-server', credentialsId: 'cred') {
                        sh '''cd backend
                        mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=demo '''
                    }
                }
            }
        }
        stage ('Quality-test') {
            steps {
                timeout(5) {
                    waitForQualityGate abortPipeline: true, credentialsId: 'cred'
                }
            }
        }
        stage ('Delivery-job') {
            steps {
                sh 'aws s3 cp backend/target/student-registration-backend-0.0.1-SNAPSHOT.jar s3://s3-delivery-bkt/easycrud.jar'
            }
        }
    }
}
```

---

## Pipeline Stages Explained

| Stage           | Purpose                                                                 |
|-----------------|--------------------------------------------------------------------------|
| **Pull-Job**     | Clones the `main` branch from the GitHub repository.                    |
| **Build-Job**    | Compiles and packages the Spring Boot backend using Maven (`-DskipTests` to keep the build fast; tests are covered by the analysis stage). |
| **Test-Job**     | Runs a SonarQube static-analysis scan against the codebase to check code quality, bugs, vulnerabilities, and code smells. |
| **Quality-test**  | Waits for SonarQube's Quality Gate verdict via webhook; aborts the pipeline if the code fails the gate. |
| **Delivery-job** | Uploads the packaged JAR artifact to an AWS S3 bucket for storage/deployment. |

---

## Troubleshooting

**`can not run elasticsearch as root`**
SonarQube's bundled Elasticsearch refuses to start as the `root` user. Always run it as a dedicated non-root user (`sonar` in this setup).

**`systemd: start-limit-hit`**
Usually caused by a mismatch between the `User=`/`Group=` in the systemd unit file and the actual owner of `/opt/sonarqube`. Fix the unit file, then:
```bash
sudo systemctl daemon-reload
sudo systemctl reset-failed sonarqube.service
sudo systemctl start sonarqube.service
```

**`Quality-test` stage hangs / times out**
The SonarQube → Jenkins webhook isn't configured (or its URL is malformed). Double check the URL under SonarQube's **Administration → Webhooks**.

**GitHub webhook: "failed to connect to host"**
- Confirm you're using the EC2 **public** IP, not the private IP.
- Confirm the EC2 Security Group allows inbound traffic on port 8080 from `0.0.0.0/0`.
- Confirm Jenkins is listening on all interfaces, not just localhost:
  ```bash
  sudo ss -tlnp | grep 8080
  ```

**Groovy syntax error: `unexpected token` in `withSonarQubeEnv`**
The parameter is `installationName` (single camelCase word), not `installation name`.

---

## Tech Stack

- **CI/CD:** Jenkins 2.555.3
- **Code Quality:** SonarQube Community Build 26.6
- **Build Tool:** Maven
- **Application:** Spring Boot (Java)
- **Cloud:** AWS EC2, AWS S3
- **OS:** Ubuntu

---

## Author

**Shubham Thakur**
[GitHub](https://github.com/shubhamthakur143) · [LinkedIn](https://linkedin.com/in/shubham-thakur-a73476368)
