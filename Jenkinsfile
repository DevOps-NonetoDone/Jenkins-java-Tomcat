pipeline {
  agent any

  tools {
    // Use the exact tool names you configure in Jenkins Global Tool Configuration
    jdk 'jdk-17'               // change to the JDK name you added in Jenkins
    maven 'maven-3.9.11'       // change to the Maven name you added in Jenkins
  }

  environment {
    WAR_NAME   = "simple-webapp-1.0.0.war"
    TARGET_WAR = "target/${env.WAR_NAME}"
    TOMCAT_PATH = "/simple-webapp"         // context path on Tomcat
    TOMCAT_HOST = "localhost:8081"         // change if Tomcat runs elsewhere/port
  }

  stages {
    stage('Clean workspace') {
      steps {
        echo "Cleaning workspace to avoid stale files..."
        // wipes the entire workspace so checkout is fresh
        deleteDir()
      }
    }

    stage('Checkout') {
      steps {
        echo "Checking out latest code from SCM..."
        checkout scm
      }
    }

    stage('Show index.html (repo)') {
      steps {
        echo "Printing the index.html that will be packaged (repo copy):"
        // on Windows age
