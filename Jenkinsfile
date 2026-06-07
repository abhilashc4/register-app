pipeline {
    agent {
        label 'Jenkins-Agent'
    }

    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "abhilashc9"
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
    }

    tools {
        jdk 'Java17'
        maven 'maven3'
    }

    stages {

        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from SCM') {
            steps {
                git branch: 'main',
                    credentialsId: 'github',
                    url: 'https://github.com/abhilashc4/register-app.git'
            }
        }

        stage('Build Application') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Test Application') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube Servers') {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    def dockerImage = docker.build("${IMAGE_NAME}")

                    docker.withRegistry('https://index.docker.io/v1/', 'DockerHub') {
                        dockerImage.push("${IMAGE_TAG}")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                sh '''
                docker run --rm \
                -v /var/run/docker.sock:/var/run/docker.sock \
                aquasec/trivy image \
                ${IMAGE_NAME}:latest \
                --no-progress \
                --scanners vuln \
                --exit-code 0 \
                --severity HIGH,CRITICAL \
                --format table
                '''
            }
        }

        stage('Cleanup Artifacts') {
            steps {
                sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                sh "docker rmi ${IMAGE_NAME}:latest || true"
            }
        }

        stage('Trigger CD Pipeline') {
            steps {
                sh """
                curl -v -k \
                --user clouduser:${JENKINS_API_TOKEN} \
                -X POST \
                -H 'cache-control: no-cache' \
                -H 'content-type: application/x-www-form-urlencoded' \
                --data 'IMAGE_TAG=${IMAGE_TAG}' \
                'http://ec2-32-197-208-57.compute-1.amazonaws.com:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token'
                """
            }
        }
    }
}
