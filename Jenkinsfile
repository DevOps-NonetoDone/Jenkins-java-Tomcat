pipeline {
  agent any

  // adjust tool names in your Jenkins configuration
  tools {
    jdk 'jdk-17'
    maven 'maven-3.9.11'
  }

  environment {
    // produced artifact
    WAR_NAME        = 'simple-webapp-1.0.0.war'
    TARGET_WAR      = "target/${WAR_NAME}"

    // Tomcat settings - change as needed
    // use context WITHOUT trailing slash, e.g. /simple-webapp-1.0.0
    TOMCAT_CONTEXT  = '/simple-webapp-1.0.0'
    // manager host (host:port). Example: localhost:8080
    TOMCAT_MANAGER_HOST = 'localhost:8081'

    // If you prefer copy method, set TOMCAT_HOME to Tomcat install folder on the Jenkins node:
    // e.g. C:\Program Files\Apache Software Foundation\Tomcat 9.0
    TOMCAT_HOME     = 'C:\\Program Files\\Apache Software Foundation\\Tomcat 9.0'

    // choose 'manager' or 'copy'
    DEPLOY_METHOD   = 'manager'
  }

  stages {
    stage('Clean workspace') {
      steps {
        echo 'Cleaning workspace to avoid stale files...'
        deleteDir()
      }
    }

    stage('Checkout') {
      steps {
        echo 'Checking out latest code...'
        checkout scm
      }
    }

    stage('Show index.html (repo)') {
      steps {
        echo 'Printing repo index.html (if present):'
        bat label: 'show index.html', script: '''
if exist src\\main\\webapp\\index.html (
  type src\\main\\webapp\\index.html
) else (
  echo index.html not found in repo path src\\main\\webapp\\index.html
)
'''
      }
    }

    stage('Build') {
      steps {
        echo 'Building with maven...'
        bat 'mvn -B -DskipTests clean package'
      }
      post {
        always {
          archiveArtifacts artifacts: 'target/*.war', fingerprint: true
        }
      }
    }

    stage('Verify WAR contents') {
      steps {
        echo 'Verifying WAR exists and contains index.html'
        bat label: 'verify war and show index.html', script: """
if not exist "%TARGET_WAR%" (
  echo ERROR: WAR not found at %TARGET_WAR% && exit /b 1
)

echo === WAR file listing (index.html entry) ===
jar -tf "%TARGET_WAR%" | findstr /I "index.html" || echo index.html not present in WAR

echo === Extract and show index.html from WAR ===
REM extract to workspace root temporarily
jar -xvf "%TARGET_WAR%" index.html >nul 2>&1 || echo failed to extract index.html

if exist index.html (
  echo --- index.html from WAR ---
  type index.html
  del /f /q index.html
) else (
  echo index.html not present after extraction
)
"""
      }
    }

    stage('Deploy') {
      steps {
        script {
          if (env.DEPLOY_METHOD == 'manager') {
            // manager: requires credentials in Jenkins (username/password)
            withCredentials([usernamePassword(credentialsId: 'tomcat-creds',
                                              usernameVariable: 'TOMCAT_USER',
                                              passwordVariable: 'TOMCAT_PASS')]) {
              bat label: 'undeploy and deploy via Tomcat Manager', script: """
echo Undeploy (ignore failure if app not present)...
curl --connect-timeout 10 "http://%TOMCAT_USER%:%TOMCAT_PASS%@${TOMCAT_MANAGER_HOST}/manager/text/undeploy?path=${TOMCAT_CONTEXT}" || echo undeploy returned non-zero

echo Uploading WAR to Tomcat manager...
if not exist "%TARGET_WAR%" (
  echo ERROR: WAR %TARGET_WAR% not found && exit /b 1
)
curl --upload-file "%TARGET_WAR%" "http://%TOMCAT_USER%:%TOMCAT_PASS%@${TOMCAT_MANAGER_HOST}/manager/text/deploy?path=${TOMCAT_CONTEXT}&update=true"
"""
            } // withCredentials
          } else if (env.DEPLOY_METHOD == 'copy') {
            // copy method: copy WAR to TOMCAT_HOME/webapps and restart Tomcat
            bat label: 'copy war to tomcat webapps', script: """
if not exist "%TARGET_WAR%" (
  echo ERROR: WAR %TARGET_WAR% not found && exit /b 1
)

REM Stop Tomcat if shutdown script exists
if exist "${TOMCAT_HOME}\\bin\\shutdown.bat" (
  echo Stopping Tomcat via shutdown.bat
  call "${TOMCAT_HOME}\\bin\\shutdown.bat"
  timeout /t 3 /nobreak >nul
) else (
  echo WARNING: shutdown.bat not found at ${TOMCAT_HOME}\\bin\\shutdown.bat - make sure Tomcat is stopped manually
)

REM Remove existing exploded folder and WAR (ignore errors)
if exist "${TOMCAT_HOME}\\webapps\\${TOMCAT_CONTEXT.substring(1)}" (
  echo Removing exploded folder ${TOMCAT_HOME}\\webapps\\${TOMCAT_CONTEXT.substring(1)}
  rmdir /s /q "${TOMCAT_HOME}\\webapps\\${TOMCAT_CONTEXT.substring(1)}" || echo failed to remove exploded folder
)

if exist "${TOMCAT_HOME}\\webapps\\%WAR_NAME%" (
  del /f /q "${TOMCAT_HOME}\\webapps\\%WAR_NAME%" || echo failed to delete old war
)

echo Copying new WAR to webapps...
copy /Y "%TARGET_WAR%" "${TOMCAT_HOME}\\webapps\\%WAR_NAME%"

REM Start Tomcat if startup script exists
if exist "${TOMCAT_HOME}\\bin\\startup.bat" (
  echo Starting Tomcat via startup.bat
  call "${TOMCAT_HOME}\\bin\\startup.bat"
) else (
  echo WARNING: startup.bat not found at ${TOMCAT_HOME}\\bin\\startup.bat - start Tomcat manually
)
"""
          } else {
            error "Unsupported DEPLOY_METHOD: ${env.DEPLOY_METHOD}. Choose 'manager' or 'copy'."
          }
        } // script
      } // steps
    } // stage Deploy

    stage('Post-deploy sanity check') {
      steps {
        echo 'Checking HTTP response from deployed app...'
        // construct URL with context and host; manager host includes port
        bat label: 'http head check', script: """
set CHECK_URL=http://${TOMCAT_MANAGER_HOST}${TOMCAT_CONTEXT}/index.html
echo Checking %CHECK_URL% ...
curl -I "%CHECK_URL%" || echo HTTP check failed (non-zero exit)
"""
      }
    }
  } // stages

  post {
    success {
      echo 'Pipeline finished: SUCCESS'
    }
    failure {
      echo 'Pipeline finished: FAILED - check the console output for error details'
    }
  }
}

