pipeline {
    agent any

    environment {
        EMAIL = "akramsyed8046@gmail.com"
    }

    stages {

        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/akramsyed8046/Devops-html-app.git'
            }
            post {
                success {
                    emailext subject: "✅ Clone SUCCESS",
                    body: "Repository cloned successfully.",
                    to: "${EMAIL}"
                }
                failure {
                    emailext subject: "❌ Clone FAILED",
                    body: "Repository cloning failed.",
                    to: "${EMAIL}"
                }
            }
        }

        stage('Test K8s Connection') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                    echo "Checking Kubernetes connection..."
                    kubectl --kubeconfig=$KUBECONFIG get nodes
                    '''
                }
            }
            post {
                success {
                    emailext subject: "✅ K8s Connection SUCCESS",
                    body: "Jenkins connected to Kubernetes successfully.",
                    to: "${EMAIL}"
                }
                failure {
                    emailext subject: "❌ K8s Connection FAILED",
                    body: "Jenkins could not connect to Kubernetes.",
                    to: "${EMAIL}"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                    echo "Deploying application..."

                    kubectl --kubeconfig=$KUBECONFIG apply -f deployment.yaml
                    kubectl --kubeconfig=$KUBECONFIG apply -f service.yaml
                    '''
                }
            }
            post {
                success {
                    emailext subject: "✅ Deployment SUCCESS",
                    body: "Application deployed to Kubernetes successfully.",
                    to: "${EMAIL}"
                }
                failure {
                    emailext subject: "❌ Deployment FAILED",
                    body: "Deployment failed. Check Jenkins logs.",
                    to: "${EMAIL}"
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                    echo "Verifying deployment..."

                    kubectl --kubeconfig=$KUBECONFIG get pods -o wide
                    kubectl --kubeconfig=$KUBECONFIG get svc
                    '''
                }
            }
            post {
                success {
                    emailext subject: "✅ Verification SUCCESS",
                    body: "Pods and services verified successfully.",
                    to: "${EMAIL}"
                }
                failure {
                    emailext subject: "❌ Verification FAILED",
                    body: "Verification failed. Check Kubernetes resources.",
                    to: "${EMAIL}"
                }
            }
        }
    }
}
