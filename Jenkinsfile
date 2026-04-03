pipeline {
    agent any

    tools {
        maven 'maven3'       // Jenkins Maven tool
        jdk 'JDK25'          // Jenkins JDK
        nodejs 'Node16'      // Jenkins NodeJS tool
    }

    environment {
        DOCKER_IMAGE = "akramsyed8046/devops-html-app:latest"
        DOCKER_CREDENTIALS = "docker-hub"       // Docker Hub credentials ID
        SONARQUBE_ENV = "SonarQube"             // Jenkins SonarQube server
        NEXUS_TOKEN = credentials('nexus-token') // Nexus npm token from Jenkins
        NEXUS_URL = "http://3.110.170.36:8081/repository/npm-releases/"
        PATH = "${tool 'Node16'}/bin:${env.PATH}" // NodeJS path
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/akramsyed8046/Devops-html-app.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                withCredentials([string(credentialsId: 'nexus-token', variable: 'NEXUS_TOKEN')]) {
                    sh '''
                    echo "registry=${NEXUS_URL}" > .npmrc
                    echo "//3.110.170.36:8081/repository/npm-releases/:_authToken=$NEXUS_TOKEN" >> .npmrc
                    npm install || echo "No package.json found, skipping install"
                    '''
                }
            }
        }

        stage('Build Project') {
            steps {
                sh 'npm run build || echo "No build script found, skipping"'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh 'sonar-scanner -Dsonar.projectKey=devops-html-app -Dsonar.sources=src'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Prepare Artifact') {
            steps {
                sh 'mkdir -p artifact'
                sh 'zip -r artifact/devops-html-app.zip src/'
            }
        }

        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: 'artifact/devops-html-app.zip', fingerprint: true
            }
        }

        stage('Publish to Nexus') {
            steps {
                withCredentials([string(credentialsId: 'nexus-token', variable: 'NEXUS_TOKEN')]) {
                    sh '''
                    echo "registry=${NEXUS_URL}" > .npmrc
                    echo "//3.110.170.36:8081/repository/npm-releases/:_authToken=$NEXUS_TOKEN" >> .npmrc
                    npm publish || echo "Publish failed, check package.json version"
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE} ."
            }
        }

        stage('Push Docker Image') {
            steps {
                withDockerRegistry([credentialsId: "${DOCKER_CREDENTIALS}", url: 'https://index.docker.io/v1/']) {
                    sh "docker push ${DOCKER_IMAGE}"
                }
            }
        }

        stage('Deploy Container') {
            steps {
                sh '''
                docker rm -f devops-html-app || true
                docker run -d --name devops-html-app -p 8080:8080 ${DOCKER_IMAGE}
                '''
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully ✅"
        }
        failure {
            echo "Pipeline failed ❌"
        }
    }
}
