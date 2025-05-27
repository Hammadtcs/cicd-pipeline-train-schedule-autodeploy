pipeline {
    agent any

    environment {
        DOCKER_IMAGE_NAME = "hammaddockerhub/train-schedule"
        KUBECONFIG = credentials('kubeconfig-credentials') // Jenkins secret file credential
    }

    stages {
        stage('Build') {
            steps {
                echo "Running build automation on branch: ${env.BRANCH_NAME}"
                script {
                    docker.image('gradle:7.6.1-jdk8').inside {
                        sh './gradlew build --no-daemon'
                    }
                }
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def app = docker.build("${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}", "--no-cache .")
                    sh "docker tag ${DOCKER_IMAGE_NAME}:${BUILD_NUMBER} ${DOCKER_IMAGE_NAME}:latest"
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-hub-credentials') {
                        sh "docker push ${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}"
                        sh "docker push ${DOCKER_IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('CanaryDeploy') {
            environment {
                CANARY_REPLICAS = 1
            }
            steps {
                script {
                    def template = readFile('train-schedule-kube-canary-template.yml')
                    def resolved = template
                        .replace('__CANARY_REPLICAS__', "${CANARY_REPLICAS}")
                        .replace('__DOCKER_IMAGE_NAME__', "${DOCKER_IMAGE_NAME}")
                        .replace('__BUILD_NUMBER__', "${BUILD_NUMBER}")
                    writeFile file: 'train-schedule-kube-canary.yml', text: resolved
                    sh 'kubectl --kubeconfig="$KUBECONFIG" apply -f train-schedule-kube-canary.yml'
                }
            }
        }

        stage('DeployToProduction') {
            environment {
                CANARY_REPLICAS = 0
            }
            steps {
                input 'Deploy to Production?'
                milestone(1)
                script {
                    def template = readFile('train-schedule-kube-canary-template.yml')
                    def resolved = template
                        .replace('__CANARY_REPLICAS__', "${CANARY_REPLICAS}")
                        .replace('__DOCKER_IMAGE_NAME__', "${DOCKER_IMAGE_NAME}")
                        .replace('__BUILD_NUMBER__', "${BUILD_NUMBER}")
                    writeFile file: 'train-schedule-kube-canary.yml', text: resolved
                    sh 'kubectl --kubeconfig="$KUBECONFIG" apply -f train-schedule-kube-canary.yml'
                    sh 'kubectl --kubeconfig="$KUBECONFIG" apply -f train-schedule-kube.yml'
                }
            }
        }
    }
}
