pipeline {
    agent any
    stages {

        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/akramsyed8046/Devops-html-app.git'
            }
        }
        

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Build Project') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Prepare Artifact') {
            steps {
                script {
                    // Make sure the artifact folder exists
                    sh 'mkdir -p artifact'
                    // Move the zip file into the artifact folder
                    sh 'mv devops-html-app.zip artifact/'
                }
            }
        }

        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: 'artifact/devops-html-app.zip', fingerprint: true
            }
        }

    }

    post {
        //always {
        //  cleanWs()
        //}
    }
}
