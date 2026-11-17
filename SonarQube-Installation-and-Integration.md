# SonarQube Installation and Integration – Ubuntu & AWS Amazon Linux 2023

## 1. Purpose

This document covers the native Linux installation and configuration of SonarQube Server, followed by integration with Maven, JaCoCo and Jenkins.

Two operating systems are covered separately:

1. Ubuntu 24.04 LTS.
2. AWS Amazon Linux 2023.

The examples use:

| Component | Version / Configuration |
|---|---|
| SonarQube Server | 2026.5.5 LTA |
| Java | JDK 21 |
| PostgreSQL | 16 |
| SonarQube web port | 9000 |
| Database port | 5432 |
| Installation method | Native ZIP installation |

**Important:** ZIP installation is still supported in the current release, but SonarSource has deprecated this method and recommends Docker or Helm for new deployments. This document deliberately uses ZIP installation to understand the native Linux setup and its components.

The commands are intended for a learning environment. Review security, backup, monitoring and capacity requirements before using a similar configuration in production.

---

# Part A – SonarQube Installation on Ubuntu 24.04 LTS

## 2. Understand the installation architecture

For this learning setup, the components can run on one Ubuntu server:

- SonarQube Server: provides the web interface and processes analysis results.
- Java 21 JDK: runs the SonarQube Server application.
- PostgreSQL: stores persistent application data.
- Elasticsearch: bundled with SonarQube and used internally for indexing and searching.
- SonarScanner: runs from the build environment and submits analysis reports to SonarQube.

The scanner does not need to run on the same machine as the SonarQube Server. For a Jenkins environment, the scanner can execute on a Jenkins controller or agent, provided it can reach the SonarQube server.

### Lab architecture

\`\`\`text
Developer
   |
   v
GitHub Repository
   |
   v
Jenkins Controller / Agent
   |
   | Build + Test + SonarScanner
   |
   v
SonarQube Server (Ubuntu)
   |
   +------ PostgreSQL Database
   |
   +------ Embedded Elasticsearch
   |
   v
SonarQube Web Interface
\`\`\`

For a small lab, PostgreSQL can run on the same EC2 instance or VM. For production, a separate database host or managed database is generally preferable.

## 3. Prepare the Ubuntu server

Update the package information and installed packages:

\`\`\`bash
sudo apt update
sudo apt upgrade -y
\`\`\`

Install the required utilities:

\`\`\`bash
sudo apt install -y wget curl unzip ca-certificates
\`\`\`

Check the operating system:

\`\`\`bash
cat /etc/os-release
\`\`\`

Check the system architecture:

\`\`\`bash
uname -m
\`\`\`

The examples below assume a 64-bit x86 Ubuntu system.

### Recommended starting resources

SonarSource's small-scale starting requirements are:

- 2 CPU cores.
- 4 GB RAM.
- 30 GB disk space.
- At least 10% free disk space.

These are starting requirements, not a guarantee of good performance. Running PostgreSQL on the same machine adds resource requirements.

## 4. Install Java 21 JDK

SonarQube Server 2026.5 requires a full JDK. A JRE-only installation is not sufficient for the native ZIP installation.

Install Java 21:

\`\`\`bash
sudo apt install -y openjdk-21-jdk
\`\`\`

Verify Java:

\`\`\`bash
java -version
\`\`\`

Verify the Java compiler:

\`\`\`bash
javac -version
\`\`\`

Check the Java installation path:

\`\`\`bash
readlink -f "$(which java)"
\`\`\`

Check the Java environment variable:

\`\`\`bash
echo "$JAVA_HOME"
\`\`\`

If \`JAVA_HOME\` is not configured, determine the JDK path:

\`\`\`bash
dirname "$(dirname "$(readlink -f "$(which javac)")")"
\`\`\`

For example, the path may be:

\`\`\`text
/usr/lib/jvm/java-21-openjdk-amd64
\`\`\`

Configure it for the current shell if required:

\`\`\`bash
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH
\`\`\`

For a persistent configuration, add the environment variables to \`/etc/environment\` or configure them directly in the SonarQube systemd service.

**Important distinction:**

- SonarQube Server uses Java to run its server application.
- Maven uses Java to build the application.
- The Java version used by the application source code can be different from the Java version used to run SonarQube Server.

Installing Java 21 for SonarQube does not mean every application being analyzed must be developed using Java 21.

## 5. Install PostgreSQL 16

SonarQube requires a supported external database for persistent application data.

Install PostgreSQL:

\`\`\`bash
sudo apt install -y postgresql-16 postgresql-client-16
\`\`\`

Check the PostgreSQL version:

\`\`\`bash
psql --version
\`\`\`

Check the service:

\`\`\`bash
sudo systemctl status postgresql
\`\`\`

If it is not running:

\`\`\`bash
sudo systemctl enable --now postgresql
\`\`\`

### 5.1 Create the SonarQube database and user

Open the PostgreSQL command-line interface:

\`\`\`bash
sudo -u postgres psql
\`\`\`

Create a dedicated database user:

\`\`\`sql
CREATE USER sonarqube WITH ENCRYPTED PASSWORD 'REPLACE_WITH_A_STRONG_PASSWORD';
\`\`\`

Create the database:

\`\`\`sql
CREATE DATABASE sonarqube OWNER sonarqube;
\`\`\`

Exit PostgreSQL:

\`\`\`sql
\\q
\`\`\`

Test the database connection:

\`\`\`bash
psql -h 127.0.0.1 -U sonarqube -d sonarqube
\`\`\`

Enter the database user's password when prompted.

Check the database connection:

\`\`\`sql
SELECT current_database(), current_user;
\`\`\`

Exit:

\`\`\`sql
\\q
\`\`\`

### Why use a separate database user?

The SonarQube application should connect using its own database account rather than the PostgreSQL superuser.

This follows the principle of least privilege and prevents the application from unnecessarily receiving administrative database permissions.

**Important:** Keep the PostgreSQL database port private. If the database is on the same machine, SonarQube can connect through \`127.0.0.1\`; there is no reason to expose port 5432 publicly.

## 6. Configure Linux prerequisites for Elasticsearch

SonarQube includes Elasticsearch internally.

Elasticsearch requires certain Linux kernel and process limits. If these requirements are not satisfied, SonarQube may fail to start or Elasticsearch may fail during initialization.

Create a configuration file:

\`\`\`bash
sudo nano /etc/sysctl.d/99-sonarqube.conf
\`\`\`

Add:

\`\`\`text
vm.max_map_count=524288
fs.file-max=131072
\`\`\`

Apply the configuration:

\`\`\`bash
sudo sysctl --system
\`\`\`

Verify the values:

\`\`\`bash
sysctl vm.max_map_count
sysctl fs.file-max
\`\`\`

Expected minimum values:

\`\`\`text
vm.max_map_count = 524288
fs.file-max = 131072
\`\`\`

The SonarQube service also requires suitable process limits. These will be configured in its systemd service file.

Elasticsearch also requires read and write access to \`/tmp\`. Do not configure the service with a restricted temporary directory that prevents this access.

## 7. Create a dedicated SonarQube user

SonarQube must not run as the Linux root user.

Create a dedicated system user:

\`\`\`bash
sudo useradd --system --create-home --shell /bin/bash sonar
\`\`\`

Create the installation directory:

\`\`\`bash
sudo mkdir -p /opt/sonarqube
\`\`\`

This user will own the SonarQube installation and its working directories.

## 8. Download and extract SonarQube Server

Download the SonarQube Server 2026.5.5 LTA ZIP distribution from the official SonarSource download page.

Use the actual download URL provided by the official download page:

\`\`\`bash
cd /tmp

wget -O sonarqube.zip "<OFFICIAL_SONARQUBE_ZIP_DOWNLOAD_URL>"
\`\`\`

Extract the archive:

\`\`\`bash
unzip sonarqube.zip
\`\`\`

The extracted directory name may include the release and build information.

Move the extracted SonarQube directory to:

\`\`\`text
/opt/sonarqube
\`\`\`

For example, if the extracted directory is named \`sonarqube-2026.5.5.XXXXX\`:

\`\`\`bash
sudo mv sonarqube-2026.5.5.XXXXX /opt/sonarqube
\`\`\`

Replace the example directory name with the actual extracted directory name.

Set ownership:

\`\`\`bash
sudo chown -R sonar:sonar /opt/sonarqube
\`\`\`

Verify:

\`\`\`bash
ls -ld /opt/sonarqube
\`\`\`

### Why use a dedicated directory?

Keeping the application under \`/opt/sonarqube\` separates the installed application from normal system files.

It also makes it easier to manage the application, its configuration and its permissions.

## 9. Configure SonarQube database connection

Open the SonarQube configuration file:

\`\`\`bash
sudo nano /opt/sonarqube/conf/sonar.properties
\`\`\`

Find the database configuration section and configure:

\`\`\`properties
sonar.jdbc.username=sonarqube
sonar.jdbc.password=REPLACE_WITH_A_STRONG_PASSWORD
sonar.jdbc.url=jdbc:postgresql://127.0.0.1:5432/sonarqube
\`\`\`

Save the file.

The configuration tells SonarQube:

- Which database user to use.
- Which password to use.
- Where the PostgreSQL database is located.
- Which database to connect to.

Protect the configuration because it contains a database password:

\`\`\`bash
sudo chown sonar:sonar /opt/sonarqube/conf/sonar.properties
sudo chmod 600 /opt/sonarqube/conf/sonar.properties
\`\`\`

For production, use an appropriate secrets-management approach rather than leaving credentials in broadly readable configuration files.

## 10. Create a systemd service

A systemd service allows Linux to manage SonarQube like other background services.

Create the service file:

\`\`\`bash
sudo nano /etc/systemd/system/sonarqube.service
\`\`\`

Add:

\`\`\`ini
[Unit]
Description=SonarQube Server
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=forking
User=sonar
Group=sonar

ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

LimitNOFILE=131072
LimitNPROC=8192

TimeoutStartSec=300
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
\`\`\`

The example uses the x86-64 startup script. If the server uses AArch64, use the architecture-specific startup directory supplied in the downloaded distribution.

Reload systemd:

\`\`\`bash
sudo systemctl daemon-reload
\`\`\`

Enable SonarQube at system startup:

\`\`\`bash
sudo systemctl enable sonarqube
\`\`\`

Start SonarQube:

\`\`\`bash
sudo systemctl start sonarqube
\`\`\`

Check the service:

\`\`\`bash
sudo systemctl status sonarqube
\`\`\`

Check the logs:

\`\`\`bash
sudo journalctl -u sonarqube -f
\`\`\`

SonarQube application logs are also available under:

\`\`\`text
/opt/sonarqube/logs/
\`\`\`

Important log files include:

- \`sonar.log\`
- \`web.log\`
- \`ce.log\`
- \`es.log\`

These logs help identify problems with application startup, the web server, background processing and Elasticsearch.

## 11. Access SonarQube through the browser

The default web port is:

\`\`\`text
9000
\`\`\`

Open:

\`\`\`text
http://<UBUNTU_SERVER_IP>:9000
\`\`\`

For the initial login, the default credentials are:

\`\`\`text
Username: admin
Password: admin
\`\`\`

SonarQube will require the initial password to be changed.

### Network configuration

If SonarQube is running on an AWS EC2 instance, configure the security group to allow TCP port 9000 only from your trusted IP address or the appropriate internal network.

Do not expose PostgreSQL port 5432 to the public internet.

For production, place SonarQube behind an HTTPS-enabled reverse proxy or load balancer and restrict direct access to the application port.

## 12. Verify the Ubuntu installation

Run:

\`\`\`bash
sudo systemctl is-active sonarqube
\`\`\`

Check the listening port:

\`\`\`bash
sudo ss -lntp | grep 9000
\`\`\`

Check the Java process:

\`\`\`bash
ps -ef | grep -i sonarqube
\`\`\`

Open the browser and confirm that the SonarQube login page loads.

At this stage, SonarQube Server is installed and connected to PostgreSQL.

---

# Part B – SonarQube Installation on AWS Amazon Linux 2023

## 13. Understand the Amazon Linux differences

The overall SonarQube architecture remains the same, but the operating system uses different package-management commands and package names.

| Task | Ubuntu | Amazon Linux 2023 |
|---|---|---|
| Package manager | APT | DNF |
| Java 21 package | \`openjdk-21-jdk\` | \`java-21-amazon-corretto-devel\` |
| PostgreSQL 16 server package | \`postgresql-16\` | \`postgresql16-server\` |
| Install packages | \`apt install\` | \`dnf install\` |
| Update package information | \`apt update\` | \`dnf check-update\` |
| Service manager | systemd | systemd |

Amazon Linux 2023 uses DNF as its default package manager. Although the \`yum\` command remains available as a compatibility pointer, this guide uses DNF.

## 14. Prepare the EC2 instance

Launch an EC2 instance with:

- Amazon Linux 2023 AMI.
- 64-bit architecture.
- At least 2 CPU cores.
- At least 4 GB RAM for a small-scale starting point.
- Sufficient EBS storage.
- An appropriate security group.

Connect to the instance using SSH.

Update the installed packages:

\`\`\`bash
sudo dnf update -y
\`\`\`

Install utilities:

\`\`\`bash
sudo dnf install -y wget curl unzip
\`\`\`

Check the operating system:

\`\`\`bash
cat /etc/os-release
\`\`\`

Check the architecture:

\`\`\`bash
uname -m
\`\`\`

## 15. Install Java 21 using Amazon Corretto

Amazon Linux 2023 provides Amazon Corretto packages.

Install the full JDK:

\`\`\`bash
sudo dnf install -y java-21-amazon-corretto-devel
\`\`\`

Verify Java:

\`\`\`bash
java -version
\`\`\`

Verify the Java compiler:

\`\`\`bash
javac -version
\`\`\`

Check the Java executable:

\`\`\`bash
which java
\`\`\`

Check the JDK installation:

\`\`\`bash
ls -l /usr/lib/jvm/
\`\`\`

Java 21 is required here to run the native SonarQube Server installation.

Remember that the Java runtime used by SonarQube is separate from the Java version used by a project being analyzed.

## 16. Install PostgreSQL 16

Install the PostgreSQL server and client packages:

\`\`\`bash
sudo dnf install -y postgresql16 postgresql16-server
\`\`\`

Check the client version:

\`\`\`bash
psql --version
\`\`\`

### 16.1 Initialize the PostgreSQL database cluster

Unlike some Ubuntu installations, the Red Hat-family installation process requires the database cluster to be initialized manually.

Run:

\`\`\`bash
sudo postgresql-setup --initdb
\`\`\`

This creates the initial PostgreSQL database cluster and its configuration files.

Enable and start PostgreSQL:

\`\`\`bash
sudo systemctl enable --now postgresql
\`\`\`

Check the service:

\`\`\`bash
sudo systemctl status postgresql
\`\`\`

Check the installed PostgreSQL version:

\`\`\`bash
sudo -u postgres psql -c "SELECT version();"
\`\`\`

**Important:** Initialize the database cluster only when it has not already been initialized. Do not run initialization commands against an existing database containing data.

## 17. Create the SonarQube database and user

Open PostgreSQL:

\`\`\`bash
sudo -u postgres psql
\`\`\`

Create the database user:

\`\`\`sql
CREATE USER sonarqube WITH ENCRYPTED PASSWORD 'REPLACE_WITH_A_STRONG_PASSWORD';
\`\`\`

Create the database:

\`\`\`sql
CREATE DATABASE sonarqube OWNER sonarqube;
\`\`\`

Exit:

\`\`\`sql
\\q
\`\`\`

Test the connection:

\`\`\`bash
psql -h 127.0.0.1 -U sonarqube -d sonarqube
\`\`\`

Verify:

\`\`\`sql
SELECT current_database(), current_user;
\`\`\`

Exit:

\`\`\`sql
\\q
\`\`\`

The same database principles apply on Ubuntu and Amazon Linux:

- Use a dedicated application database user.
- Do not use the PostgreSQL superuser for SonarQube.
- Keep the database network access restricted.
- Protect database credentials.

## 18. Configure Elasticsearch Linux prerequisites

SonarQube includes Elasticsearch, so a separate Elasticsearch installation is not required.

Create the kernel configuration:

\`\`\`bash
sudo nano /etc/sysctl.d/99-sonarqube.conf
\`\`\`

Add:

\`\`\`text
vm.max_map_count=524288
fs.file-max=131072
\`\`\`

Apply the configuration:

\`\`\`bash
sudo sysctl --system
\`\`\`

Verify:

\`\`\`bash
sysctl vm.max_map_count
sysctl fs.file-max
\`\`\`

The minimum values should be:

\`\`\`text
vm.max_map_count = 524288
fs.file-max = 131072
\`\`\`

The SonarQube systemd service will also need:

\`\`\`text
LimitNOFILE=131072
LimitNPROC=8192
\`\`\`

These limits are important because Elasticsearch creates many files and threads during its operation.

## 19. Create the SonarQube Linux user and directory

Create the dedicated user:

\`\`\`bash
sudo useradd --system --create-home --shell /bin/bash sonar
\`\`\`

Create the installation directory:

\`\`\`bash
sudo mkdir -p /opt/sonarqube
\`\`\`

The SonarQube application must run as this dedicated user, not as root.

## 20. Download and extract SonarQube

Download the SonarQube Server 2026.5.5 LTA ZIP distribution from the official SonarSource download page.

Example:

\`\`\`bash
cd /tmp

wget -O sonarqube.zip "<OFFICIAL_SONARQUBE_ZIP_DOWNLOAD_URL>"
\`\`\`

Extract the archive:

\`\`\`bash
unzip sonarqube.zip
\`\`\`

Move the extracted directory to:

\`\`\`text
/opt/sonarqube
\`\`\`

For example:

\`\`\`bash
sudo mv sonarqube-2026.5.5.XXXXX /opt/sonarqube
\`\`\`

Replace the example directory name with the actual extracted directory name.

Set the correct ownership:

\`\`\`bash
sudo chown -R sonar:sonar /opt/sonarqube
\`\`\`

Verify:

\`\`\`bash
ls -ld /opt/sonarqube
\`\`\`

## 21. Configure the SonarQube database connection

Open:

\`\`\`bash
sudo nano /opt/sonarqube/conf/sonar.properties
\`\`\`

Configure:

\`\`\`properties
sonar.jdbc.username=sonarqube
sonar.jdbc.password=REPLACE_WITH_A_STRONG_PASSWORD
sonar.jdbc.url=jdbc:postgresql://127.0.0.1:5432/sonarqube
\`\`\`

Protect the configuration:

\`\`\`bash
sudo chown sonar:sonar /opt/sonarqube/conf/sonar.properties
sudo chmod 600 /opt/sonarqube/conf/sonar.properties
\`\`\`

The configuration is the same as Ubuntu because the SonarQube application and PostgreSQL JDBC connection properties do not change based on the Linux distribution.

## 22. Create the systemd service on Amazon Linux 2023

Create:

\`\`\`bash
sudo nano /etc/systemd/system/sonarqube.service
\`\`\`

Add:

\`\`\`ini
[Unit]
Description=SonarQube Server
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=forking
User=sonar
Group=sonar

ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

LimitNOFILE=131072
LimitNPROC=8192

TimeoutStartSec=300
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
\`\`\`

The example uses the x86-64 startup script. Select the architecture-specific startup directory if the EC2 instance uses AArch64.

Reload systemd:

\`\`\`bash
sudo systemctl daemon-reload
\`\`\`

Enable the service:

\`\`\`bash
sudo systemctl enable sonarqube
\`\`\`

Start SonarQube:

\`\`\`bash
sudo systemctl start sonarqube
\`\`\`

Check the service:

\`\`\`bash
sudo systemctl status sonarqube
\`\`\`

Follow the logs:

\`\`\`bash
sudo journalctl -u sonarqube -f
\`\`\`

SonarQube application logs:

\`\`\`text
/opt/sonarqube/logs/
\`\`\`

## 23. Configure EC2 security group access

For browser access, the EC2 security group must permit TCP port 9000 from a trusted source.

Example:

| Protocol | Port | Source | Purpose |
|---|---:|---|---|
| TCP | 22 | Trusted administrator IP | SSH |
| TCP | 9000 | Trusted administrator IP or internal network | SonarQube web access |
| TCP | 5432 | Localhost or private database network only | PostgreSQL |

If PostgreSQL is installed on the same EC2 instance, SonarQube can connect through localhost. Do not create a public inbound rule for port 5432.

For a production environment, use HTTPS through a reverse proxy or load balancer rather than exposing the application directly over public HTTP.

## 24. Verify the Amazon Linux installation

Check the service:

\`\`\`bash
sudo systemctl is-active sonarqube
\`\`\`

Check the listening port:

\`\`\`bash
sudo ss -lntp | grep 9000
\`\`\`

Check PostgreSQL:

\`\`\`bash
sudo systemctl is-active postgresql
\`\`\`

Open:

\`\`\`text
http://<EC2_PUBLIC_IP>:9000
\`\`\`

Complete the initial login and password change.

At this point, SonarQube Server should be running on Amazon Linux 2023 and connected to PostgreSQL.

---

# Part C – Maven, JaCoCo and Jenkins Integration

## 25. Understand the integration flow

Installing SonarQube Server does not automatically analyze application source code.

A scanner must execute in the build environment and submit the analysis results to the SonarQube Server.

For a Maven project, the workflow is:

\`\`\`text
GitHub
   |
   v
Jenkins
   |
   v
Maven Build
   |
   v
Unit Tests
   |
   v
JaCoCo Coverage Report
   |
   v
SonarScanner for Maven
   |
   v
SonarQube Server
   |
   v
Quality Gate Evaluation
\`\`\`

The scanner and SonarQube Server have different responsibilities:

- SonarScanner for Maven analyzes the project and submits the analysis.
- SonarQube Server processes the analysis and evaluates the Quality Gate.
- Jenkins coordinates the CI/CD workflow and can decide whether to continue or fail the pipeline.

## 26. Configure Maven for SonarQube analysis

For Maven projects, use the SonarScanner for Maven.

The analysis command follows this pattern:

\`\`\`bash
mvn clean verify \\
  org.sonarsource.scanner.maven:sonar-maven-plugin:<PINNED_PLUGIN_VERSION>:sonar
\`\`\`

Replace \`<PINNED_PLUGIN_VERSION>\` with the scanner plugin version selected and tested for the environment.

Pinning the plugin version helps make builds repeatable instead of allowing an unexpected plugin update to change the build environment.

### Authentication

Create a SonarQube analysis token using an appropriately privileged account.

Use a dedicated service account or project-scoped token with the permissions needed to execute analysis.

Do not use a System Administrator account or token for routine Jenkins analysis.

For a local shell test, provide the token through an environment variable:

\`\`\`bash
export SONAR_TOKEN="YOUR_SONAR_TOKEN"
\`\`\`

Then run the Maven scanner command.

The token must not be committed to GitHub or stored directly in a \`pom.xml\` file.

The current scanner authentication property is:

\`\`\`text
sonar.token
\`\`\`

The older \`sonar.login\` and \`sonar.password\` parameters are deprecated.

## 27. Generate code coverage using JaCoCo

SonarQube does not generate Java code coverage reports itself.

JaCoCo is responsible for measuring coverage during test execution and generating the coverage report. SonarQube imports that report during analysis.

A typical Maven JaCoCo report is generated at:

\`\`\`text
target/site/jacoco/jacoco.xml
\`\`\`

Configure the SonarQube coverage report path when needed:

\`\`\`properties
sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
\`\`\`

The Maven build must be configured to generate the report before SonarScanner runs.

Run:

\`\`\`bash
mvn clean verify
\`\`\`

Then confirm that the report exists:

\`\`\`bash
ls -l target/site/jacoco/jacoco.xml
\`\`\`

Run the SonarScanner analysis after the report has been generated.

**Important distinction:** Code coverage indicates how much code was exercised by tests. It does not prove that the tests are effective or that the application is free from defects.

## 28. Integrate SonarQube with Jenkins

Install the Jenkins plugin:

\`\`\`text
SonarQube Scanner for Jenkins
\`\`\`

This plugin provides Jenkins integration, including server configuration, Pipeline steps and Quality Gate handling.

It is different from the SonarScanner for Maven, which performs the actual Maven project analysis.

### 28.1 Configure the SonarQube server in Jenkins

Open:

\`\`\`text
Manage Jenkins
    |
    v
System
    |
    v
SonarQube servers
\`\`\`

Add a SonarQube server configuration.

Example:

\`\`\`text
Name: My SonarQube Server
Server URL: http://<SONARQUBE_SERVER_IP>:9000
Server authentication token: Jenkins credential
\`\`\`

Store the token in Jenkins Credentials as a secret rather than writing it directly in a job or Jenkinsfile.

### 28.2 Configure the webhook in SonarQube

Open the SonarQube administration settings and create a webhook.

Use:

\`\`\`text
https://<JENKINS-DOMAIN>/sonarqube-webhook/
\`\`\`

The trailing slash is required.

The webhook allows SonarQube to notify Jenkins when analysis and Quality Gate evaluation have completed.

Jenkins must be reachable from SonarQube for the webhook request to succeed.

For a private AWS deployment, ensure that the security groups and network routes permit this communication.

## 29. Jenkins Pipeline integration

A Declarative Pipeline can run analysis using:

\`\`\`groovy
pipeline {
    agent any

    stages {
        stage('Build and SonarQube Analysis') {
            steps {
                withSonarQubeEnv('My SonarQube Server') {
                    sh '''
                        mvn clean verify \\
                        org.sonarsource.scanner.maven:sonar-maven-plugin:<PINNED_PLUGIN_VERSION>:sonar
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }
}
\`\`\`

Replace the plugin version placeholder with the pinned version configured for the project.

### How the integration works

1. Jenkins starts the Maven build.
2. Maven compiles the application and runs configured tests.
3. JaCoCo generates the coverage report when configured.
4. SonarScanner for Maven analyzes the project.
5. \`withSonarQubeEnv\` supplies the configured SonarQube environment.
6. The analysis report is submitted to SonarQube.
7. SonarQube processes the analysis and evaluates the Quality Gate.
8. The webhook notifies Jenkins when the result is available.
9. \`waitForQualityGate\` retrieves the result.
10. If \`abortPipeline: true\` is configured, Jenkins aborts the pipeline when the Quality Gate is not successful.

The \`waitForQualityGate\` step does not require a Jenkins executor while it waits.

**Important:** Running the scanner does not automatically guarantee that Jenkins will fail when the Quality Gate fails. Jenkins must explicitly wait for and act on the Quality Gate result.

---

# Part D – Troubleshooting and Practical Scenarios

## 30. Scenario: SonarQube service fails to start

**Situation:** SonarQube has been installed, but the service does not become active.

**Symptoms:**

\`\`\`bash
sudo systemctl status sonarqube
\`\`\`

The service shows a failure or exits shortly after starting.

**Investigation:**

Check the service logs:

\`\`\`bash
sudo journalctl -u sonarqube -n 100
\`\`\`

Check SonarQube logs:

\`\`\`bash
sudo ls -l /opt/sonarqube/logs/
\`\`\`

Check Java:

\`\`\`bash
java -version
\`\`\`

Check the Linux prerequisites:

\`\`\`bash
sysctl vm.max_map_count
sysctl fs.file-max
\`\`\`

**Possible root causes:**

- Incorrect Java version.
- JRE installed instead of the required JDK.
- SonarQube is running as root.
- Incorrect file ownership.
- Elasticsearch prerequisites are not satisfied.
- Insufficient RAM or disk space.
- Database connection failure.

**Resolution:** Correct the specific configuration or resource issue identified in the logs, then restart the service.

\`\`\`bash
sudo systemctl restart sonarqube
\`\`\`

**Why it matters:** A service failure is a symptom, not a root cause. Checking the application and Elasticsearch logs helps identify the actual problem.

## 31. Scenario: Elasticsearch fails during startup

**Situation:** The SonarQube web interface does not become available because Elasticsearch fails to initialize.

**Investigation:**

\`\`\`bash
sysctl vm.max_map_count
\`\`\`

If the value is lower than required, update the persistent kernel configuration:

\`\`\`text
vm.max_map_count=524288
\`\`\`

Apply:

\`\`\`bash
sudo sysctl --system
\`\`\`

Check the Elasticsearch log:

\`\`\`bash
sudo tail -n 100 /opt/sonarqube/logs/es.log
\`\`\`

Also verify that the service has sufficient open-file and thread limits.

**Resolution:** Correct the kernel or process limits, confirm sufficient disk space and permissions, and restart SonarQube.

**Why it matters:** Elasticsearch is an internal SonarQube component. Its failure can prevent the entire SonarQube application from becoming available.

## 32. Scenario: SonarQube cannot connect to PostgreSQL

**Situation:** SonarQube starts but cannot establish a database connection.

**Possible symptoms:**

- Database connection errors in \`sonar.log\`.
- Authentication failures.
- Connection refused.
- SonarQube startup failure.

**Investigation:**

Check PostgreSQL:

\`\`\`bash
sudo systemctl status postgresql
\`\`\`

Test the database connection:

\`\`\`bash
psql -h 127.0.0.1 -U sonarqube -d sonarqube
\`\`\`

Review the configuration:

\`\`\`bash
sudo nano /opt/sonarqube/conf/sonar.properties
\`\`\`

Confirm that the username, password, database name and JDBC URL match the PostgreSQL configuration.

**Possible root causes:**

- PostgreSQL service is stopped.
- Database does not exist.
- Incorrect password.
- Incorrect JDBC URL.
- Database permissions are incorrect.

**Resolution:** Correct the database configuration and verify connectivity before restarting SonarQube.

## 33. Scenario: SonarQube is running but the browser cannot access it

**Situation:** The systemd service is active, but the browser cannot load the SonarQube page.

**Investigation:**

Check the listening port:

\`\`\`bash
sudo ss -lntp | grep 9000
\`\`\`

Check the service:

\`\`\`bash
sudo systemctl status sonarqube
\`\`\`

For AWS EC2, check the security group inbound rules.

**Possible root causes:**

- Port 9000 is not listening.
- SonarQube web service has not completed startup.
- EC2 security group blocks the connection.
- Network routing or firewall rules block the request.

**Resolution:** Confirm that SonarQube is listening and that the trusted client can reach TCP port 9000.

Do not solve the issue by opening all ports to the public internet.

## 34. Scenario: SonarQube analysis reports zero coverage

**Situation:** SonarQube displays 0% coverage even though Maven tests have executed.

**Investigation:**

Check whether the JaCoCo report exists:

\`\`\`bash
ls -l target/site/jacoco/jacoco.xml
\`\`\`

Check whether the report was generated before the scanner executed.

Check the configured property:

\`\`\`properties
sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
\`\`\`

**Possible root causes:**

- JaCoCo is not configured.
- Tests did not execute.
- The XML report was not generated.
- The report path is incorrect.
- Scanner ran before report generation.

**Resolution:** Configure JaCoCo correctly, run the tests and generate the report before SonarScanner starts.

**Why it matters:** Running tests and generating coverage data are related but separate operations. SonarQube needs the coverage report to import coverage information.

## 35. Scenario: Jenkins analysis succeeds but the pipeline does not wait for the Quality Gate

**Situation:** Jenkins successfully submits analysis, but the pipeline continues without checking the Quality Gate.

**Investigation:**

Check:

- Whether the SonarQube Scanner for Jenkins plugin is installed.
- Whether \`withSonarQubeEnv\` wraps the analysis step.
- Whether \`waitForQualityGate\` is configured.
- Whether the SonarQube webhook URL is correct.
- Whether the trailing slash is present.
- Whether SonarQube can reach Jenkins.

**Possible root causes:**

- Webhook is missing.
- Webhook URL is incorrect.
- Network connectivity is blocked.
- Pipeline does not call \`waitForQualityGate\`.

**Resolution:** Correct the webhook and Pipeline configuration, then execute a new analysis.

**Why it matters:** Analysis submission and Quality Gate evaluation are different steps. Jenkins needs the Quality Gate result before it can make a reliable pipeline decision.

---

# Part E – Production Considerations

## 36. Lab environment versus production environment

A single EC2 instance running SonarQube and PostgreSQL is useful for learning.

However, production environments should consider separation of responsibilities.

| Area | Learning environment | Production environment |
|---|---|---|
| SonarQube Server | Single VM | Properly sized and monitored server |
| PostgreSQL | Same VM possible | Separate or managed database preferred |
| Database access | Local connection | Private network only |
| Web access | Port 9000 restricted to trusted IP | HTTPS through reverse proxy or load balancer |
| Credentials | Protected configuration | Secrets management and controlled access |
| Backups | Manual backup practice | Scheduled database and configuration backups |
| Monitoring | systemd and log checks | Centralized logs, metrics and alerting |
| Storage | Basic EBS volume | Adequate capacity, performance and backup strategy |

### Why separate PostgreSQL?

SonarQube stores persistent application data in PostgreSQL, while Elasticsearch maintains its own search indexes.

Separating the database can improve operational isolation, simplify database management and reduce resource contention.

A database backup is essential. Elasticsearch indexes should not be treated as a replacement for the PostgreSQL database.

## 37. Important operational checks

Useful Linux commands for ongoing administration:

Check SonarQube:

\`\`\`bash
sudo systemctl status sonarqube
\`\`\`

Restart SonarQube:

\`\`\`bash
sudo systemctl restart sonarqube
\`\`\`

Stop SonarQube:

\`\`\`bash
sudo systemctl stop sonarqube
\`\`\`

Check PostgreSQL:

\`\`\`bash
sudo systemctl status postgresql
\`\`\`

Check disk space:

\`\`\`bash
df -h
\`\`\`

Check memory:

\`\`\`bash
free -h
\`\`\`

Check listening ports:

\`\`\`bash
sudo ss -lntp
\`\`\`

Follow SonarQube logs:

\`\`\`bash
sudo tail -f /opt/sonarqube/logs/sonar.log
\`\`\`

Follow Elasticsearch logs:

\`\`\`bash
sudo tail -f /opt/sonarqube/logs/es.log
\`\`\`

---

# 38. Important Technical Understanding

1. SonarQube Server is the central platform that processes and stores code analysis results.
2. SonarScanner performs project analysis and submits the results to SonarQube Server.
3. PostgreSQL stores persistent SonarQube application data.
4. Elasticsearch is bundled with SonarQube and handles internal indexing and search.
5. SonarQube must run under a dedicated Linux user rather than root.
6. Java 21 JDK is used to run the native ZIP installation in this guide.
7. Ubuntu uses APT, while Amazon Linux 2023 uses DNF.
8. SonarQube uses port 9000 by default.
9. JaCoCo generates Java coverage reports; SonarQube imports them.
10. Jenkins coordinates the build, analysis and Quality Gate workflow.
11. \`withSonarQubeEnv\` prepares the SonarQube environment for analysis.
12. \`waitForQualityGate\` allows Jenkins to wait for the analysis result.
13. A SonarQube webhook is required for Jenkins Quality Gate notification.
14. A successful scanner execution does not automatically mean that the Quality Gate passed.
15. ZIP installation remains available in the current release but is deprecated for future removal.

## 39. Final learning outcome

The installation process helped establish the relationship between Linux administration, Java, PostgreSQL, Elasticsearch, SonarQube and Jenkins.

The important operational understanding is that SonarQube is not simply a web application that can be installed and immediately used for CI/CD. It depends on a compatible Java runtime, supported database, Linux system requirements, correct permissions, network connectivity and a scanner running in the build environment.

The same overall architecture can be implemented on Ubuntu and Amazon Linux 2023, but package installation, database initialization and system administration commands must match the operating system.

The next practical step is to validate the installation, execute a Maven project analysis, import a JaCoCo coverage report and configure Jenkins to enforce the Quality Gate.
