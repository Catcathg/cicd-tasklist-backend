pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('catherine-dockerhub-credentials')
        DOCKERHUB_NAMESPACE   = "${DOCKERHUB_CREDENTIALS_USR}"
        IMAGE_NAME            = "${DOCKERHUB_NAMESPACE}/tasklist-backend"
        IMAGE_TAG             = "${env.BUILD_NUMBER}"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Build') {
            steps {
                sh 'npx prisma generate'
                sh 'npm run build'
            }
        }

        stage('Unit tests') {
            steps {
                sh 'npm run test:coverage'
            }
            post {
                always {
                    junit 'reports/junit.xml'
                }
            }
        }

        stage('E2E tests') {
            steps {
                sh 'npm run test:e2e:coverage'
            }
        }

        stage('SonarQube analysis') {
            steps {
                withSonarQubeEnv('sonarqube-server-1') {
                    withCredentials([string(credentialsId: 'catherine-sonar-token', variable: 'MY_SONAR_TOKEN')]) {
                        sh 'npx sonar-scanner -Dsonar.token=$MY_SONAR_TOKEN'
                    }
                }
            }
        }

        stage('Docker build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest ."
            }
        }

        stage('Trivy scan') {
            steps {
                sh """
                    trivy image --exit-code 0 --severity HIGH,CRITICAL \
                        --format table -o trivy-report.txt ${IMAGE_NAME}:${IMAGE_TAG}
                """
                archiveArtifacts artifacts: 'trivy-report.txt', allowEmptyArchive: true
            }
        }

        stage('Generate SBOM (SPDX)') {
            steps {
                sh """
                    syft ${IMAGE_NAME}:${IMAGE_TAG} -o spdx-json > sbom-spdx.json
                """
                archiveArtifacts artifacts: 'sbom-spdx.json', allowEmptyArchive: true
            }
        }

        stage('Docker push') {
            steps {
                sh """
                    echo "${DOCKERHUB_CREDENTIALS_PSW}" | docker login -u "${DOCKERHUB_CREDENTIALS_USR}" --password-stdin
                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${IMAGE_NAME}:latest
                    docker logout
                """
            }
        }
    }

    post {
        always {
            sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest || true"
        }
    }
}