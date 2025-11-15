pipeline {
  agent any

  tools {
    // Use the exact tool names you configure in Jenkins Global Tool Configuration
    jdk 'jdk-17'               // change to the JDK name you add in Jenkins
    maven 'maven-3.9.11'       // change to the Maven name you add in Jenkins
  }

  environment {
    WAR_NAME = "simple-webapp-1.0.0.war"
    TARGET_WAR = "target/${env.WAR_NAME}"
    TOMCAT_PATH = "/simple-webapp"   // context path on Tomcat
    TOMCAT_HOST = "localhost:8081"   // change if Tomcat runs elsewhere/port
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build') {
      steps {
        // on Windows use 'bat'
        bat "mvn -B clean package"
      }
      post {
        always {
          archiveArtifacts artifacts: "target/*.war", fingerprint: true
        }
      }
    }

    stage('Deploy to Tomcat (HTTP manager)') {
      steps {
        // Use Jenkins credentials (username/password) -- credential id must exist
        withCredentials([usernamePassword(credentialsId: 'tomcat-creds',
                                          usernameVariable: 'TOMCAT_USER',
                                          passwordVariable: 'TOMCAT_PASS')]) {
          // Use curl which exists on modern Windows
          bat """
            if not exist "%TARGET_WAR%" exit /b 1
            curl --upload-file "%TARGET_WAR%" "http://%TOMCAT_USER%:%TOMCAT_PASS%@${TOMCAT_HOST}/manager/text/deploy?path=${TOMCAT_PATH}&update=true"
          """
        }
      }
    }
  }

  post {
    success {
      echo "Pipeline finished: SUCCESS"
    }
    failure {
      echo "Pipeline finished: FAILED"
    }
  }
}
