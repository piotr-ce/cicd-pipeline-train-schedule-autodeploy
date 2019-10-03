pipeline {
    agent any
    // global job environment variables
    environment {
        //be sure to replace "willbla" with your own Docker Hub username
        DOCKER_IMAGE_NAME = "pio2pio/train-schedule"
        CANARY_REPLICAS = 0
    }
    stages {
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    app = docker.build(DOCKER_IMAGE_NAME)
                    app.inside {
                        sh 'echo Hello, World!'
                    }
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-password-pio2pio') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('CanaryDeploy') {
            when        { branch 'master'     }
            environment { CANARY_REPLICAS = 3 }
            steps {
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'train-schedule-kube-canary.yml',
                    enableConfigSubstitution: true
                )
            }
        }
        stage('SmokeTest') {
            when  { branch 'master' }
            steps {
                script  {
                    def response = httpRequest (
                        url    : "http://$KUBE_MASTER_IP:8081",
                        timeout: 20
                    )
                    if (response.status != 200)
                        error("Smoke test against canary http://$KUBE_MASTER_IP:8081 failed")
                }
            }
        }
        
        stage('DeployToProduction') {
            when  { branch 'master' }
            steps {
                // input 'Deploy to Production?' // commented out and replaced with smoke-test
                milestone(1)
                kubernetesDeploy(
                    configs                 : 'train-schedule-kube.yml',
                    kubeconfigId            : 'kubeconfig',
                    enableConfigSubstitution: true
                )
            }
        }
        post {
            cleanup {
                kubernetesDeploy(
                    configs                 : 'train-schedule-kube-canary.yml',
                    kubeconfigId            : 'kubeconfig',
                    enableConfigSubstitution: true
                )
            }
        }
    }
}
