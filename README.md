 Complete Guide — Git · GitHub · Maven · Jenkins · Tomcat

> A single, practical document that explains what we set up, **why** each tool is used, what you must configure, how deployment works end-to-end, and exactly how to diagnose and fix common failures. It's written so you can follow it step‑by‑step for your Java WAR/Tomcat project.

---

## 1. Summary — what we did

You built a simple Java webapp project (WAR) and a CI/CD pipeline that:

* Stores source in **Git** and **GitHub**
* Uses **Maven** to build and package the project into a WAR (`target/*.war`)
* Uses **Jenkins** to automate the build and deploy stages (Jenkinsfile in repo)
* Deploys the WAR to **Tomcat** using either Tomcat Manager API or by copying the WAR into `webapps/` (or via SSH/SCP to remote host)
* Validates deployment by requesting `index.html` and checking the content (we added a visible `Deployed` badge to make verification obvious)

This document explains *why* we used each tool, the configuration details, and how to diagnose problems when the deployed page does not update.

---

## 2. Why each tool — purpose & need

### Git (local) and GitHub (remote)

**Purpose:** source control, version history, branches, collaboration. GitHub hosts the remote repo and provides webhooks to notify Jenkins.

**Why we need it:**

* Track code changes and revert mistakes.
* Collaborate, review via pull requests.
* Trigger automated builds when you push code (webhooks).

If Git/GitHub is missing: you would have to manually transfer code to Jenkins; reproducibility and traceability suffer.

---

### Maven (build & dependency manager)

**Purpose:** build lifecycle management and dependency resolution. Produces standardized artifacts (WAR/JAR).

**Why we need it:**

* Declarative build (`pom.xml`) lets any machine reproduce the build.
* Manages external libraries and transitive dependencies.
* Standard lifecycle (`clean`, `compile`, `package`) used in CI.

**Key point with Tomcat:** mark servlet API and container-provided libraries as `<scope>provided</scope>` in `pom.xml` so they are not packaged inside the WAR (Tomcat already provides them).

If you did not use Maven (or Gradle), the build would be ad-hoc and less portable between machines and CI.

---

### Jenkins (automation / CI server)

**Purpose:** run the pipeline automatically on code changes: checkout, build, test, package, deploy.

**Why we need it:**

* Repeatable, logged, and automated builds.
* Central place to store credentials, artifacts, and build history.
* Can run scripts (shell/PowerShell) to deploy to Tomcat, call the manager API, or connect to remote hosts via SSH.

If Jenkins is missing: deployments are manual and error-prone.

---

### Tomcat (servlet container)

**Purpose:** run Java web applications (WARs); serves servlets/JSP and static content.

**Why we need it:**

* Lightweight production-ready servlet container.
* Handles HTTP, servlet lifecycle and classloading.

Tomcat can be controlled via the **Manager web app** (HTTP API) or by file operations in `webapps/`.

If Tomcat is missing: you cannot run the WAR as a Java webapp.

---

## 3. What we deploy — artifact details

* **Artifact type:** WAR (Web ARchive) produced by Maven: `target/simple-webapp-1.0.0.war`.
* **Packed contents:** `index.html` under the webapp root, `WEB-INF/` with `web.xml` (if any), compiled classes under `WEB-INF/classes`, and dependency JARs under `WEB-INF/lib` (except those marked `provided`).
* **Context path:** by default, Tomcat deploys `simple-webapp-1.0.0.war` to `/simple-webapp-1.0.0/`. To deploy at root `/`, use `ROOT.war`.

---

## 4. Required configuration (exact files & settings)

### 4.1 GitHub

* Push code to a public or private repo.
* Optionally configure a webhook: `http://<JENKINS_HOST>:8080/github-webhook/` → push events.

### 4.2 Maven (`pom.xml`)

* Standard `packaging` = `war`.
* Add `provided` scope for servlet API:

```xml
<dependency>
  <groupId>javax.servlet</groupId>
  <artifactId>javax.servlet-api</artifactId>
  <version>4.0.1</version>
  <scope>provided</scope>
</dependency>
```

* Use maven-war-plugin only if you need to customize packaging.

### 4.3 Jenkins

* Install Git and Maven on the Jenkins node or configure Jenkins tool locations.
* Create credentials:

  * Username/password for Tomcat Manager (ID: `tomcat-creds`) if using manager API.
  * SSH key credential (ID: `tomcat-ssh-key`) if using SSH/SCP deploy.
* Create Pipeline job that checks out the repo and runs the Jenkinsfile.
* Optional: configure webhook on GitHub to trigger builds.

### 4.4 Tomcat

* `TOMCAT_HOME` path (Windows example: `C:\Program Files\Apache Software Foundation\Tomcat 9.0`).
* Edit `conf/tomcat-users.xml` to add a manager user (for remote manager deploy):

```xml
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<user username="jenkins" password="StrongPassHere" roles="manager-script,manager-gui"/>
```

* Secure `tomcat-users.xml` and restrict manager access by firewall or internal network.
* Ensure Tomcat listens on the intended port (default `8080`).

---

## 5. End-to-end workflow — exactly how it will happen

**Developer**: edit files → `git commit` → `git push` → **GitHub** receives push → GitHub webhook (optional) triggers **Jenkins** → **Jenkins** runs Jenkinsfile:

1. **Checkout**: Jenkins uses Git plugin to clone the repo.
2. **Build**: Jenkins runs `mvn -B -DskipTests clean package` producing `target/*.war`.
3. **Archive**: Jenkins archives the WAR as a build artifact.
4. **Deploy**: Jenkins deploys WAR to Tomcat via one of the methods (Manager API / copy / scp).
5. **Sanity check**: Jenkins runs `curl -I http://<tomcat-host>:<port>/<context>/index.html` and fails the build if response not `200`.
6. **Result**: Tomcat serves the new content at the configured context path.

---

## 6. Example Jenkinsfile(s)

### 6.1 Tomcat Manager (Windows agent) — recommended for manager available

```groovy
pipeline {
  agent any
  environment {
    WAR_NAME = 'simple-webapp-1.0.0.war'
    TARGET_WAR = "target/${WAR_NAME}"
    TOMCAT_MANAGER_HOST = 'localhost:8080'
    TOMCAT_CONTEXT = '/simple-webapp-1.0.0'
  }
  stages {
    stage('Checkout & Build') {
      steps {
        checkout scm
        bat 'mvn -B -DskipTests clean package'
        archiveArtifacts artifacts: 'target/*.war', fingerprint: true
      }
    }
    stage('Deploy (Tomcat Manager)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'tomcat-creds', usernameVariable: 'TUSER', passwordVariable: 'TPASS')]) {
          bat "curl --upload-file %TARGET_WAR% \"http://%TUSER%:%TPASS%@${TOMCAT_MANAGER_HOST}/manager/text/deploy?path=${TOMCAT_CONTEXT}&update=true\""
        }
      }
    }
    stage('Sanity') {
      steps { bat "curl -I \"http://${TOMCAT_MANAGER_HOST}${TOMCAT_CONTEXT}/index.html\"" }
    }
  }
}
```

### 6.2 Copy & restart Tomcat (Jenkins on same host as Tomcat)

```groovy
pipeline {
  agent any
  environment {
    TOMCAT_HOME = 'C:\\Program Files\\Apache Software Foundation\\Tomcat 9.0'
    WAR_NAME = 'simple-webapp-1.0.0.war'
    TARGET_WAR = "target/${WAR_NAME}"
    TOMCAT_CONTEXT = '/simple-webapp-1.0.0'
  }
  stages {
    stage('Build') {
      steps {
        checkout scm
        bat 'mvn -B -DskipTests clean package'
        archiveArtifacts artifacts: 'target/*.war', fingerprint: true
      }
    }
    stage('Deploy (copy)') {
      steps {
        bat """
          call "${TOMCAT_HOME}\\bin\\shutdown.bat"
          rmdir /s /q "${TOMCAT_HOME}\\webapps\\${TOMCAT_CONTEXT.substring(1)}" || echo no exploded folder
          del /f /q "${TOMCAT_HOME}\\webapps\\${WAR_NAME}" || echo no old war
          copy /Y "%TARGET_WAR%" "${TOMCAT_HOME}\\webapps\\%WAR_NAME%"
          call "${TOMCAT_HOME}\\bin\\startup.bat"
        """
      }
    }
    stage('Sanity') { steps { bat "curl -I \"http://localhost:8080${TOMCAT_CONTEXT}/index.html\"" } }
  }
}
```

### 6.3 Remote SSH + SCP (Linux Tomcat)

```groovy
pipeline {
  agent any
  environment {
    REMOTE_USER = 'ubuntu'
    REMOTE_HOST = 'ec2-host.example.com'
    REMOTE_TOMCAT_HOME = '/opt/tomcat'
    WAR_NAME = 'simple-webapp-1.0.0.war'
  }
  stages {
    stage('Build') { steps { checkout scm; sh 'mvn -B -DskipTests clean package'; archiveArtifacts artifacts: 'target/*.war' } }
    stage('Deploy via SSH') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'tomcat-ssh-key', keyFileVariable: 'SSH_KEY')]) {
          sh '''
            chmod 600 "$SSH_KEY"
            scp -o StrictHostKeyChecking=no -i "$SSH_KEY" target/${WAR_NAME} ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_TOMCAT_HOME}/webapps/
            ssh -o StrictHostKeyChecking=no -i "$SSH_KEY" ${REMOTE_USER}@${REMOTE_HOST} "${REMOTE_TOMCAT_HOME}/bin/shutdown.sh || true; sleep 2; ${REMOTE_TOMCAT_HOME}/bin/startup.sh"
          '''
        }
      }
    }
    stage('Sanity') { steps { sh "curl -I http://${REMOTE_HOST}:8080/simple-webapp-1.0.0/index.html" } }
  }
}
```

---

## 7. Common failure modes & how to approach them (debugging playbook)

Use this short decision tree whenever the new page does not show.

### 7.1 Step A — Confirm Jenkins build output

* Check Jenkins console → ensure `BUILD SUCCESS` and `target/*.war` was produced.
* Download WAR from Jenkins artifacts and inspect `index.html`:

  ```bash
  jar -tf target/*.war | grep index.html
  jar -xvf target/*.war index.html
  cat index.html
  ```
* If WAR does **not** contain updated file → problem is in Git/checkout or Maven build (wrong branch, stale workspace). Fix by checking commit hash and re-running build.

### 7.2 Step B — Confirm Jenkins attempted deploy & result

* Look at Jenkins console for deploy step:

  * Manager method: `OK - Deployed application at context path [/xxx]` or an HTTP error (401, 403, 500).
  * Copy method: confirm `copy` or `scp` returned success and that `startup.bat` executed.
* If curl returns `curl: (7) Failed to connect` → wrong host/port or Tomcat not running.

### 7.3 Step C — Confirm Tomcat instance & files

* Ensure you are browsing the same Tomcat instance Jenkins targeted (same host & port).
* On Tomcat machine list `webapps/` and timestamps:

  * Windows PowerShell:

    ```powershell
    Get-ChildItem 'C:\path\to\tomcat\webapps' | Where-Object { $_.Name -match "simple-webapp" } | Select Name, LastWriteTime
    Get-Item 'C:\path\to\tomcat\webapps\simple-webapp-1.0.0\index.html' | Select Name, LastWriteTime
    ```
  * Linux:

    ```bash
    ls -l /opt/tomcat/webapps | grep simple-webapp
    stat /opt/tomcat/webapps/simple-webapp-1.0.0/index.html
    ```
* If exploded folder exists and is older than WAR → Tomcat did not re-explode. Stop Tomcat, delete exploded folder, copy WAR, start Tomcat.

### 7.4 Step D — Check logs

* `TOMCAT_HOME/logs/catalina.*.log` and `localhost.*.log` for exceptions during deployment.
* If Manager API returns errors, catalina logs often show why (permission, missing classes, classloader issues).

### 7.5 Step E — Browser caching

* Use Incognito or hard reload `Ctrl+F5`.

### 7.6 Common specific errors and fixes

* `401 Unauthorized` from manager API: wrong credentials or missing `manager-script` role.
* `curl: (7) Failed to connect`: wrong host/port or Tomcat stopped; run `netstat -ano | findstr ":8080"` (Windows) or `ss -ltnp | grep 8080` (Linux).
* Multiple webapps (`simple-webapp.war` vs `simple-webapp-1.0.0.war`) → delete duplicates to avoid context collision.

---

## 8. Useful commands & quick checklists

### Git

```bash
git status
git log -1 --pretty=oneline
git fetch origin
git rebase origin/main
```

### Maven

```bash
mvn -B -DskipTests clean package
mvn dependency:tree
```

### Inspect WAR

```bash
jar -tf target/*.war | grep index.html
jar -xvf target/*.war index.html
```

### Tomcat (Windows)

```powershell
# find listening ports
netstat -ano | findstr ":8080"
# map PID
tasklist /FI "PID eq 1234"
# view logs
Get-Content 'C:\path\to\tomcat\logs\catalina.*.log' -Tail 200
```
C:\Users\Admin\simple-webapp
 ├── pom.xml
 └── src
     └── main
         └── webapp
             └── index.html
  Jenkins File 

----


You are learning a Java-based CI/CD pipeline, so each step has a purpose.
Let’s break it down in the simplest possible way.

🔷 FIRST — What is Tomcat?

Tomcat is not a web server like Nginx.

Tomcat is a Java Application Server.

It can run only 3 types of files:

.jsp (Java Server Pages)

.servlets (Java compiled classes)

.war files (Web Application Archive)

❌ Tomcat cannot run HTML files directly from your computer
✔ Tomcat can run HTML only when they are inside a webapp folder or a WAR.

So to deploy anything to Tomcat, the app must follow Java web structure.

🔶 So why did we create a Maven project?

Because Maven creates a proper Java web project structure.

This structure is required by Tomcat.

Maven gives you:

/src/main/webapp       → Place HTML, JS, CSS
/src/main/java         → Place Java files (if any)
/pom.xml               → Maven configuration

✔ Even if your app has just HTML, Tomcat still requires Maven structure

HTML is allowed only inside:

src/main/webapp/


Tomcat will NOT consider any other location.

🔷 WHY create pom.xml?

pom.xml is the heart of a Maven project.

It tells Maven:

Project name

Version

Build type (packaging: war)

Dependencies

Plugins (like maven-war-plugin)

Example:

<packaging>war</packaging>


This is the most important line.

It tells Maven:

👉 Build my project into a WAR file

🔷 WHY do we need WAR files?

A WAR = Web Application Archive

Think of WAR like a zip file that contains your whole web project.

Tomcat will take this .war file and:

✔ Extract it
✔ Convert it to a runnable Java web application
✔ Deploy it to webapps folder
✔ Serve the pages

Without a WAR file:

❌ Tomcat cannot understand or run your project
❌ Jenkins cannot deploy to Tomcat
❌ CI/CD cannot work

🔷 WHY did we build the project using Maven?

Maven packaging step:

mvn clean package


Does 3 things:
1️⃣ Cleans old builds
2️⃣ Checks project structure
3️⃣ Creates the file:

target/simple-webapp-1.0.0.war


This WAR is what Tomcat can deploy.

🔷 WHY did we put WAR into Tomcat/webapps?

Because webapps is the deployment folder for Tomcat.

Tomcat automatically:

Detects a new WAR

Extracts it into a folder

Deploys it as a running web application

Example:

Tomcat/webapps/simple-webapp-1.0.0/


And you access it at:

http://localhost:8081/simple-webapp-1.0.0/

🔷 So where is Java used here?

Even though your app is HTML only…

Tomcat itself is a Java program.

To run Tomcat → Java is required
To run Maven → Java is required
To build WAR → Java is required
To read WAR structure → Java is required

So Java is used in:

✔ Tomcat engine
✔ Maven compiler
✔ WAR builder
✔ Servlet container (even if you don’t use Java code)

🔷 Why Jenkins comes into picture?

To automate the process:

✔ Pull code from GitHub
✔ Build WAR using Maven
✔ Deploy WAR to Tomcat
✔ Restart automatically when code changes push

Jenkins = CI/CD tool

Without Jenkins:

Every time you change HTML

You must manually build WAR

Manually copy to Tomcat

Manually deploy

That is slow ❌
Jenkins makes it automatic ✔

🌟 NOW THE COMPLETE WORKFLOW MAKES SENSE
1️⃣ Write code in GitHub

(HTML / CSS / JS / Java)

2️⃣ Jenkins pulls code

(from GitHub automatically)

3️⃣ Jenkins runs Maven

(builds WAR file)

4️⃣ Jenkins deploys WAR

(into Tomcat webapps OR via manager API)

5️⃣ Tomcat runs your application

(on browser)

This is the real DevOps pipeline for Java web applications.

Step	                                    Why Needed?                                    	Without It?
Maven project	                  Standard Java web structure	                            Tomcat won’t understand
src/main/webapp               	HTML must be inside this folder	                        Deployment fails
pom.xml	                        Build instructions for WAR	                            WAR cannot be created
WAR (target/...)	              Deployable package	                                    Tomcat cannot deploy
Tomcat	                        Java Web Server	                                        Cannot run WAR
Java (JDK)	                    Runs Maven + Tomcat	                                    Nothing works
Jenkins	Automation	            Manual                                                  deployment every time
webapps folder	                Tomcat’s deploy directory	                              App won’t run

--------------------------------------------

Maven 

Maven POM — What is groupId, artifactId, version, and packaging?

In your pom.xml, you see something like:

<groupId>com.example</groupId>
<artifactId>simple-webapp</artifactId>
<version>1.0.0</version>
<packaging>war</packaging>


Let’s break each one with meaning, why we use it, and what happens if we use it wrong.

1️⃣ artifactId (MOST IMPORTANT)
✔ Meaning

This is the name of your project / app.
Maven uses this name for the output file.

Example:
artifactId = simple-webapp

Output after build:

simple-webapp-1.0.0.war

✔ Why we use it?

It identifies your application uniquely inside the group/company.

Tomcat uses the artifactId (plus version) to create the context path.

Example:

simple-webapp-1.0.0.war  →  http://localhost:8080/simple-webapp-1.0.0/

❌ If you change it

The WAR name changes

Tomcat deploys to a new URL

Old app stays undeleted → causes confusion
(This is exactly the reason you saw old UI sometimes)

2️⃣ groupId
✔ Meaning

This represents your organization or company.

Example:

com.example
com.umesh.devops
io.github.YourName

✔ Why we use it?

Prevents naming conflicts
(Two teams could both have simple-webapp, but their groupId makes them different)

Used when publishing to Maven repositories like Nexus.

❌ If you misuse groupId

Doesn’t break Tomcat

But looks unprofessional and causes conflicts in repos

3️⃣ version
✔ Meaning

Version of your application.

Example:

1.0.0
1.0.1
1.1.0


WAR generated:

simple-webapp-1.0.1.war

✔ Why we use it?

Helps in versioning deployments (v1, v2, v3)

Jenkins can store multiple versions

You can rollback easily

❌ If version changes every time

Tomcat will deploy multiple folders:

simple-webapp-1.0.0/
simple-webapp-1.0.1/
simple-webapp-1.0.2/


→ Creates confusion
→ Old versions remain
→ You may see old UI if wrong URL is loaded

4️⃣ packaging
✔ Meaning

Defines what type of output Maven should create.

Common types:

war → For web apps (Tomcat)

jar → For normal Java apps

pom → For parent/aggregator projects

Example:

<packaging>war</packaging>

✔ Why we use it?

Because Tomcat can only deploy WAR files.

❌ Wrong packaging =

If packaging = jar, then Maven builds:

simple-webapp-1.0.0.jar


Tomcat cannot deploy it → app will not run.





---


1️⃣ <dependencies> — What libraries your project needs
✔ Meaning

This section defines external libraries that your project needs to compile or run.

Example:

<dependencies>
  <dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>4.0.1</version>
    <scope>provided</scope>
  </dependency>
</dependencies>

✔ Why we use it?

Your Java code may need:

Servlet API (for web apps)

JSP API

Logging frameworks

JSON libraries

DB libraries (JDBC driver)

Maven downloads them automatically from Maven Central.


scope	          Meaning	             Included inside WAR?	          Who provides it?
provided	Container provides it	        ❌ No              	        Tomcat/glassfish
compile   (default)	Your app needs it   ✔ Yes              	         You
           at compile & runtime	                
runtime	  Only needed when running	    ✔ Yes	                       You
test	     Only for testing	            ❌ No	                       Unit test

2️⃣ <build> — How your project should be built

This controls:

Output directories

WAR folder structure

Plugin configuration

Build customizations

Example:

<build>
    <finalName>simple-webapp</finalName>
    <plugins>
        <!-- plugins go here -->
    </plugins>
</build>

✔ Why we use it?

To change WAR name

Add plugins (compiler, Tomcat deploy)

Include or exclude files

Custom build instructions

✔ Example: change final WAR name

If you want WAR name to be exact:

<finalName>portfolio</finalName>


Output:

portfolio.war


3️⃣ <properties> — Where all versions & constants go

Example:

<properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>

✔ Why we use it?

Set Java version

Set encoding

Avoid repeating version numbers

Make project portable across systems

✔ Example

Without this:

Code may fail on some systems

Wrong Java version error

4️⃣ <plugins> — Tools used during build & packaging

Plugins live inside <build> → <plugins>.

✔ Common plugins in a webapp
a) Maven Compiler Plugin

Controls Java version.

<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <version>3.11.0</version>
  <configuration>
    <source>17</source>
    <target>17</target>
  </configuration>
</plugin>

Why use it?

To ensure Jenkins, your laptop, and other environments compile code with same Java version.

b) Maven WAR Plugin

Controls how WAR is packaged.

<plugin>
  <artifactId>maven-war-plugin</artifactId>
  <version>3.3.2</version>
</plugin>

Why use it?

It decides:

what goes inside the WAR

where resources come from

web.xml handling

c) Tomcat Maven Plugin (optional)

Allows deploying to Tomcat directly from Maven:

<plugin>
  <groupId>org.apache.tomcat.maven</groupId>
  <artifactId>tomcat7-maven-plugin</artifactId>
  <version>2.2</version>
</plugin>


But you are using Jenkins, so this plugin is not needed.

⭐ Full Example POM Explained
<project>
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.umesh.devops</groupId>
  <artifactId>simple-webapp</artifactId>
  <version>1.0.0</version>
  <packaging>war</packaging>

  <properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencies>
    <dependency>
      <groupId>javax.servlet</groupId>
      <artifactId>javax.servlet-api</artifactId>
      <version>4.0.1</version>
      <scope>provided</scope> 
    </dependency>
  </dependencies>

  <build>
    <finalName>simple-webapp-1.0.0</finalName>

    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.11.0</version>
        <configuration>
          <source>17</source>
          <target>17</target>
        </configuration>
      </plugin>

      <plugin>
        <artifactId>maven-war-plugin</artifactId>
        <version>3.3.2</version>
      </plugin>
    </plugins>
  </build>
</project>
