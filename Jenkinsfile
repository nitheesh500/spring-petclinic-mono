pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    environment {
        DOCKER_IMAGE = "naraharinitheesh/petclinic"
        // DOCKER_TAG = sh(
        //     script: "git rev-parse --short HEAD",
        //     returnStdout: true).trim()
        DOCKER_TAG   = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git url: 'https://github.com/nitheesh500/spring-petclinic-mono.git',
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
                mvn clean verify -Dspring.profiles.active=default \
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
                      -Dsonar.login=$SONAR_AUTH_TOKEN \
                      -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
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
        stage('Package'){
            steps{
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image'){
            steps{
                sh '''
                export DOCKER_BUILDKIT=1
                ls
                docker build -t $DOCKER_IMAGE:$DOCKER_TAG .

                '''
            }
        }

        stage('Trivy Image Scan'){

            steps{
                sh '''
                trivy image --severity HIGH,CRITICAL --exit-code 1 $DOCKER_IMAGE:$DOCKER_TAG
                '''
            }
        }

        stage(' Push to Docker Hub'){
            steps{

                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                 usernameVariable: 'DOCKERHUB_USERNAME',
                 passwordVariable: 'DOCKERHUB_PASSWORD')]){

                    sh '''
                        set +x
                         echo "$DOCKERHUB_PASSWORD" | docker login \
                                 -u "$DOCKERHUB_USERNAME" \
                                   --password-stdin
                         docker push "$DOCKER_IMAGE:$DOCKER_TAG"
                        docker logout

                    '''
                 }
            }
        }

        // stage('cleanup Docker Image from Jenkins Host'){
        //     steps{
        //         sh '''
        //         docker rmi $DOCKER_IMAGE:$DOCKER_TAG || true
        //         '''
        //     }
        // }

        stage('Update Manifest Repo') {
            environment {
                DOCKER_IMAGE = 'naraharinitheesh/petclinic'
                // DOCKER_TAG   = '10'
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-creds',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_PASS'
                )]) {
                    sh '''
                      set -e

                      rm -rf argocd-helm-cd
                      git clone https://github.com/nitheesh500/argocd-helm-cd.git
                      cd argocd-helm-cd

                      git remote set-url origin https://$GIT_USER:$GIT_PASS@github.com/nitheesh500/argocd-helm-cd.git

                      sed -i "s|tag: .*|tag: $DOCKER_TAG|" helm/petclinic/values.yaml

                      git config user.email "naraharinitheesh@gmail.com"
                      git config user.name $GIT_USER

                      git add helm/petclinic/values.yaml
                      git commit -m "Update image tag to $DOCKER_TAG" || echo "No changes to commit"
                      git push origin main
                    '''
                }
            }
        }

    }
}
