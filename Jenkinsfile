pipeline {
    agent any

    environment {
        APP_NAME       = "my-app"
        DOCKER_IMAGE   = "dockerhubusername/my-app"
        SONARQUBE_ENV  = "sonarqube"
        GIT_REPO       = "github.com/your-org/your-repo.git"
        GIT_BRANCH    = "main"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: "${GIT_BRANCH}",
                    credentialsId: 'github-creds',
                    url: "https://${GIT_REPO}"
            }
        }

        stage('SonarQube Analysis') {
            environment {
                SONAR_TOKEN = credentials('sonar-token')
            }
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh """
                      sonar-scanner \
                      -Dsonar.projectKey=${APP_NAME} \
                      -Dsonar.sources=. \
                      -Dsonar.host.url=$SONAR_HOST_URL \
                      -Dsonar.login=$SONAR_TOKEN
                    """
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

        stage('Build Docker Image') {
            steps {
                sh """
                  docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                """
            }
        }

        stage('Push Docker Image') {
            environment {
                DOCKER_CREDS = credentials('dockerhub-creds')
            }
            steps {
                sh """
                  echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin
                  docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                """
            }
        }

        stage('Update Image Tag in K8s Manifest') {
            steps {
                sh """
                  sed -i 's|image: .*|image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|' k8s/deployment.yaml
                """
            }
        }

        stage('Commit & Push Manifest Changes') {
            environment {
                GIT_CREDS = credentials('github-creds')
            }
            steps {
                sh """
                  git config user.email "jenkins@ci.com"
                  git config user.name "Jenkins CI"

                  git add k8s/deployment.yaml
                  git commit -m "Update image tag to ${BUILD_NUMBER}"
                  
                  git push https://${GIT_CREDS_USR}:${GIT_CREDS_PSW}@${GIT_REPO} ${GIT_BRANCH}
                """
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully"
        }
        failure {
            echo "❌ Pipeline failed"
        }
    }
}
