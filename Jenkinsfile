pipeline {
    agent any
    environment {
        //be sure to replace "willbla" with your own Docker Hub username
        DOCKER_IMAGE_NAME = "mmunta20/train-schedule"
        CANARY_REPLICAS = 0
    }
    stages {
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
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerhublogin') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('CanaryDeploy') {
            when {
                branch 'master'
            }
            environment { 
                CANARY_REPLICAS = 1
            }
            steps {
                withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'sample', contextName: '', credentialsId: 'kubelogin', namespace: '', serverUrl:'https://172.18.71.69:6443']]) {
                sh 'curl -LO "https://storage.googleapis.com/kubernetes-release/release/v1.20.5/bin/linux/amd64/kubectl"'  
                sh 'chmod u+x ./kubectl'  
                sh './kubectl get nodes'
                sh './kubectl apply -f train-schedule-kube-canary.yml'
                sh './kubectl get pods'
            }
                
            }
        }
        stage('SmokeTest') {
            when {
                branch 'master'
            }
            steps{
                script {
                    sleep (time:5)
                    def response = httpRequest (
                        url: "http://$KUBE_MASTER_IP:32010/",
                        timeout: 30
                    )
                    if (response.status != 200){
                        error("smoke test againt canary deployment failed.")
                    }
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            steps {
                milestone(1)
                
                withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'sample', contextName: '', credentialsId: 'kubelogin', namespace: '', serverUrl:'https://172.18.71.69:6443']]) {
                sh 'curl -LO "https://storage.googleapis.com/kubernetes-release/release/v1.20.5/bin/linux/amd64/kubectl"'  
                sh 'chmod u+x ./kubectl'  
                sh './kubectl get nodes'
                sh './kubectl apply -f train-schedule-kube.yml'
                sh './kubectl get pods'
            }
        }
    }
    post{
        cleanup {
                withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'sample', contextName: '', credentialsId: 'kubelogin', namespace: '', serverUrl:'https://172.18.71.69:6443']]) {
                sh 'curl -LO "https://storage.googleapis.com/kubernetes-release/release/v1.20.5/bin/linux/amd64/kubectl"'  
                sh 'chmod u+x ./kubectl'  
                sh './kubectl get nodes'
                sh './kubectl apply -f train-schedule-kube-canary.yml'
                sh './kubectl get pods'
                }   
            }      
        }
    }
}
