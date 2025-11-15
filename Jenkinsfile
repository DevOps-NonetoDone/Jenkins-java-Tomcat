pipeline {
  agent any

  tools {
    jdk 'jdk-17'          // adjust to your Jenkins JDK tool name
    maven 'maven-3.9.11'  // adjust to your Jenkins Maven tool name
  }

  environment {
    WAR_NAME    = 'simple-webapp-1.0.0.war'
    TARGET_WAR  = "target/${WAR_NAME}"
    TOMCAT_PATH = '/simple-webapp'
    TOMCAT_HOST = 'localhost:8081'
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
        echo 'Checking out latest code from SCM...'
        checkout scm
      }
    }

    stage('Show index.html (repo)') {
      steps {
        echo 'Printing the index.html that will be packaged (repo copy):'
        bat '''
if exist src\\main\\webapp\\index.html (
  type src\\main\\webapp\\index.html
) else (
  echo index.html not found
)
'''
      }
    }

    stage('Build') {
      steps {
        echo 'Running mvn -B clean package...'
        bat 'mvn -B clean package'
      }
      post {
        always {
          archiveArtifacts artifacts: 'target/*.war', fingerprint: true
        }
      }
    }

    stage('Verify WAR contents') {
      steps {
        echo 'Listing WAR contents and showing packaged index.html (if present)...'
        bat '''
if not exist "%TARGET_WAR%" (
  echo ERROR: WAR not found at %TARGET_WAR% && exit /b 1
)

echo === WAR file listing (showing index.html entry) ===
jar -tf "%TARGET_WAR%" | findstr /I index.html || echo index.html not found in WAR

echo === Extract and display index.html from WAR (for verification) ===
jar -xvf "%TARGET_WAR%" index.html >nul 2>&1 || echo failed to extract index.html

if exist index.html (
  type index.html
  del /f /q index.html
) else (
  echo index.html not present after extraction
)
'''
      }
    }

    stage('Deploy to Tomcat (HTTP manager)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'tomcat-creds',
                                          usernameVariable: 'TOMCAT_USER',
                                          passwordVariable: 'TOMCAT_PASS')]) {
          echo 'Undeploying existing app (if present) then deploying new WAR...'
          bat '''
REM tolerate failure of undeploy (app may not exist)
curl --connect-timeout 10 "http://%TOMCAT_USER%:%TOMCAT_PASS%@%TOMCAT_HOST%/manager/text/undeploy?path=%TOMCAT_PATH%" || echo undeploy failed or app not present

REM upload new WAR (checks file exists first)
if not exist "%TARGET_WAR%" (
  echo ERROR: WAR %TARGET_WAR% not found && exit /b 1
)

curl --upload-file "%TARGET_WAR%" "http://%TOMCAT_USER%:%TOMCAT_PASS%@%TOMCAT_HOST%/manager/text/deploy?path=%TOMCAT_PATH%&update=true"
'''
        }
      }
    }

    stage('Post-deploy sanity check') {
      steps {
        echo 'Checking HTTP response from deployed app (head request)...'
        bat 'curl -I "http://%TOMCAT_HOST%%TOMCAT_PATH%/index.html" || echo HTTP check failed'
      }
    }
  }

  post {
    success {
      echo 'Pipeline finished: SUCCESS'
    }
    failure {
      echo 'Pipeline finished: FAILED'
    }
  }
}
