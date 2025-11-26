pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        // Tomcat details
        TOMCAT_HOST = '43.205.236.253'
        TOMCAT_PORT = '8080'
        APP_CONTEXT = '/Springdemo-0.0.1-SNAPSHOT'

        // SonarQube details
        SONAR_HOST_URL     = 'http://13.235.54.0:9000/'
        SONAR_PROJECT_KEY  = 'myprodcode'
        SONAR_PROJECT_NAME = 'prod-app-code'
    }

    stages {
        stage('checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/kiran7028/jenkins-pipeline.git'
            }
        }

        stage('build') {
            steps {
                sh '''
                    echo "Building my code using Maven..."
                    mvn clean install
                '''
            }
        }

        stage('test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('sonar-analysis') {
            steps {
                // SonarQube server name: "sonar" (configured in Jenkins global settings)
                withSonarQubeEnv('sonarqube') {
                    withCredentials([
                        string(credentialsId: 'sonar', variable: 'SONAR_TOKEN')
                    ]) {
                        sh '''
                            echo "Running SonarQube analysis..."

                            mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                              -Dsonar.projectKey=$SONAR_PROJECT_KEY \
                              -Dsonar.projectName=$SONAR_PROJECT_NAME \
                              -Dsonar.host.url=$SONAR_HOST_URL \
                              -Dsonar.token=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        stage('quality-gate') {
            steps {
                // Wait for SonarQube to compute Quality Gate result (via webhook)
                timeout(time: 5, unit: 'MINUTES') {
                    //waitForQualityGate abortPipeline: true
                    def qg = waitForQualityGate()
                    if (qg.status != 'OK') {
                        error "Pipeline aborted due to quality gate failure: ${qg.status}"
                    }
                }
            }
        }
        stage('deploy-to-tomcat') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'tomcat',
                        usernameVariable: 'TOMCAT_USER',
                        passwordVariable: 'TOMCAT_PASS'
                    )
                ]) {
                    sh '''
                        echo "Finding WAR file..."
                        WAR_FILE=$(ls target/*.war | head -n 1)
                        echo "Deploying $WAR_FILE to Tomcat..."

                        curl --fail -u "$TOMCAT_USER:$TOMCAT_PASS" \
                          -T "$WAR_FILE" \
                          "http://$TOMCAT_HOST:$TOMCAT_PORT/manager/text/deploy?path=$APP_CONTEXT&update=true"

                        echo "Deployment triggered at: http://$TOMCAT_HOST:$TOMCAT_PORT$APP_CONTEXT"
                    '''
                }
            }
        }
    }

    post {
        success {
            withCredentials([
                string(credentialsId: 'webhook', variable: 'SLACK_WEBHOOK_URL')
            ]) {
                sh '''
                    SONAR_DASHBOARD_URL="${SONAR_HOST_URL}/dashboard?id=${SONAR_PROJECT_KEY}"

                    PAYLOAD=$(cat <<EOF
{
  "text": "✅ *Deployment SUCCESS*\\n• Job: ${JOB_NAME}\\n• Build: #${BUILD_NUMBER}\\n• App: http://${TOMCAT_HOST}:${TOMCAT_PORT}${APP_CONTEXT}\\n• SonarQube: ${SONAR_DASHBOARD_URL}\\n• Channel: #webhook-test"
}
EOF
                    )

                    curl -X POST -H 'Content-type: application/json' \
                         --data "$PAYLOAD" \
                         "$SLACK_WEBHOOK_URL"
                '''
            }
        }

        failure {
            withCredentials([
                string(credentialsId: 'webhook', variable: 'SLACK_WEBHOOK_URL')
            ]) {
                sh '''
                    PAYLOAD=$(cat <<EOF
{
  "text": "❌ *Deployment FAILED*\\n• Job: ${JOB_NAME}\\n• Build: #${BUILD_NUMBER}\\n• Console: ${BUILD_URL}\\n• Channel: #webhook-test"
}
EOF
                    )

                    curl -X POST -H 'Content-type: application/json' \
                         --data "$PAYLOAD" \
                         "$SLACK_WEBHOOK_URL"
                '''
            }
        }
    }
}