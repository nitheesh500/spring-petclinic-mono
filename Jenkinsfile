pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    stages {

        stage('Checkout') {
            steps {
                git url: 'https://github.com/spring-projects/spring-petclinic.git',
                    branch: 'main'
            }
        }

        stage('GitLeaks Scan') {
            steps {
                sh '''
                  gitleaks detect \
                  --source . \
                  --no-git \
                  --exit-code 1
                '''
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Trivy FS Scan (Dependencies)') {
            steps {
                sh '''
                  trivy fs \
                  --severity HIGH,CRITICAL \
                  --exit-code 1 \
                  .
                '''
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                mvn test -Dspring.profiles.active=default \
                -Dspring.docker.compose.enabled=false \
                -Dtest='!*PostgresIntegrationTests'
          '''
            }
        }

        stage('SonarQube Scan') {
            environment {
                SONAR_AUTH_TOKEN = credentials('sonar-token')
            }
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                      mvn sonar:sonar \
                      -Dsonar.projectKey=petclinic \
                      -Dsonar.login=$SONAR_AUTH_TOKEN
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }
}
