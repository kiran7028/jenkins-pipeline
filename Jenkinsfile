pipeline {
    agent any                     // Run this pipeline on any available Jenkins agent/node

    tools {
        maven 'maven'             // Use the Maven installation named "maven" (configured in Jenkins global tools)
    }

    environment {
        TOMCAT_HOST    = '172.31.35.219'        // Tomcat server private or public or Elastic IP address
        TOMCAT_PORT    = '8080'                    // Tomcat port where manager app is available
        APP_CONTEXT    = '/Springdemo-0.0.1-SNAPSHOT'  // Application context path for Tomcat deployment
    }

    stages {

        stage('checkout') {
            steps {
                // Clone the repo from GitHub using the 'main' branch
                git branch: 'main',
                    url: 'https://github.com/kiran7028/jenkins-pipeline.git'
            }
        }

        stage('build') {
            steps {
                sh '''
                    echo "Building my code using Maven..."   # Print message on console
                    mvn clean install                       # Clean previous builds and compile + package new build
                '''
            }
        }

        stage('test') {
            steps {
                sh 'mvn test'       // Run all unit tests in the project
            }
        }

        stage('deploy-to-tomcat') {
            steps {
                // Bind Jenkins credentials (ID: tomcat) to environment variables
                // TOMCAT_USER = username, TOMCAT_PASS = password
                withCredentials([usernamePassword(
                    credentialsId: 'tomcat',        // Credential ID stored in Jenkins
                    usernameVariable: 'TOMCAT_USER', // Variable name for Username
                    passwordVariable: 'TOMCAT_PASS'  // Variable name for Password
                )]) {

                    sh '''
                        echo "Finding WAR file..."
                        # Pick the first .war file generated inside target/
                        WAR_FILE=$(ls target/*.war | head -n 1)

                        echo "Deploying $WAR_FILE to Tomcat..."

                        # Use curl to deploy the WAR to Tomcat using Tomcat Manager
                        curl --fail -u "$TOMCAT_USER:$TOMCAT_PASS" /   # Authenticate with Tomcat
                        -T "$WAR_FILE" /                              # Upload WAR file
                        "http://$TOMCAT_HOST:$TOMCAT_PORT/manager/text/deploy?path=$APP_CONTEXT&update=true"

                        # Print deployment success message
                        echo "Deployment triggered. Check Tomcat logs if needed."
                    '''
                }
            }
        }
    }
}