pipeline {
    agent any

    environment {
       // APP_NAME       = "jenkins-sonar-docker"
        DOCKER_IMAGE   = "ashish2999/go-web-app"
       // SONARQUBE_ENV  = "sonarqube"
        GIT_REPO       = "github.com/DAshishReddy/go-web-app-devops.git"
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
        withSonarQubeEnv('sonarqube') {
            script {
                def scannerHome = tool 'sonar-scanner'
                sh '''
                  ''' + scannerHome + '''/bin/sonar-scanner \
                  -Dsonar.projectKey=jenkins-sonar-docker \
                  -Dsonar.sources=. \
                  -Dsonar.login=$SONAR_TOKEN
                '''
            }
        }
    }
}

      /* stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        } */

        stage('Build Docker Image') {
            steps {
                sh '''
                  docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Push Docker Image') {
            environment {
                DOCKER_CREDS = credentials('dockerhub-creds')
            }
            steps {
                sh '''
                  echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin
                  docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Update Image Tag in K8s Manifest') {
            steps {
                sh '''
                  sed -i 's|image: .*|image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|' go-web-app-devops/blob/main/k8s/manifests/deployment.yaml
                '''
            }
        }

        stage('Commit & Push Manifest Changes') {
            environment {
                GIT_CREDS = credentials('github-creds')
            }
            steps {
                sh '''
                  git config user.email "ashishreddy.dulla@gmail.com"
                  git config user.name "Ashish"

                  git add go-web-app-devops/blob/main/k8s/manifests/deployment.yaml
                  git commit -m "Update image tag to ${BUILD_NUMBER}"
                  
                  git push https://${GIT_CREDS_USR}:${GIT_CREDS_PSW}@${GIT_REPO} ${GIT_BRANCH}
                '''
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
