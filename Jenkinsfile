pipeline {
    agent any

    environment {
        DOCKER_IMAGE_NAME = "hammaddockerhub/train-schedule"
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
                    app = docker.build("${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}")
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-hub-credentials') {
                        app.push()
                        app.push("latest")
                    }
                }
            }
        }

        stage('CanaryDeploy') {
            environment {
                CANARY_REPLICAS = 1
            }
            steps {
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'train-schedule-kube-canary.yml',
                    enableConfigSubstitution: true
                )
            }
        }

        stage('DeployToProduction') {
            environment {
                CANARY_REPLICAS = 0
            }
            steps {
                input 'Deploy to Production?'
                milestone(1)
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'train-schedule-kube-canary.yml',
                    enableConfigSubstitution: true
                )
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'train-schedule-kube.yml',
                    enableConfigSubstitution: true
                )
            }
        }
    }
}
