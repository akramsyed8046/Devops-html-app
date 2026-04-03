pipeline {
    agent any

    tools {
        nodejs 'Node18'           // Jenkins NodeJS tool
        jdk 'JDK25'               // Only if needed for SonarQube
    }

    environment {
        DOCKER_IMAGE = "akramsyed8046/devops-html-app:latest"
        DOCKER_CREDENTIALS = "docker-hub"
        SONARQUBE_ENV = "sonarqube"
        NEXUS_TOKEN = credentials('nexus-token')
        NEXUS_REPO = "http://43.205.195.167:8081/repository/raw-repo/"
        PATH = "${tool 'Node18'}/bin:${env.PATH}"
        KUBECONFIG = "/home/jenkins/.kube/config"  // Path to your kubeconfig
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/akramsyed8046/Devops-html-app.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                if [ -f package.json ]; then
                    npm install
                else
                    echo "No package.json found, skipping install"
                fi
                '''
            }
        }
	

        stage('Build Project') {
            steps {
                sh '''
                if npm run | grep -q "build"; then
                    npm run build
                else
                    echo "No build script found, skipping"
                fi
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh "${tool 'SonarQube-Scanner'}/bin/sonar-scanner \
                       -Dsonar.projectKey=devops-html-app \
                       -Dsonar.sources=src"
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

        stage('Publish to Nexus (Raw)') {
            steps {
                withCredentials([string(credentialsId: 'nexus-token', variable: 'NEXUS_TOKEN')]) {
                    sh '''
                    VERSION=$(node -p "require('./package.json').version")
                    if [[ "$VERSION" == *"-SNAPSHOT"* ]]; then
                        REPO=http://43.205.195.167:8081/repository/npm-snapshots/
                    else
                        REPO=http://43.205.195.167:8081/repository/npm-releases/
                    fi

                    echo "registry=$REPO" > .npmrc
                    echo "//$REPO:_authToken=$NEXUS_TOKEN" >> .npmrc
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

		stage('Update kubeconfig') {
    steps {
        sh '''
        /usr/local/bin/aws eks update-kubeconfig \
        --region ap-south-1 \
        --name rehcluster
        '''
		  }
}

        stage('Deploy to Kubernetes') {
            steps {
                echo "Deploying to Kubernetes cluster..."
                sh '''
                # Apply Deployment manifest (make sure k8s/deployment.yaml exists)
                kubectl apply -f deployment.yaml

                # Apply Service manifest (optional)
                kubectl apply -f service.yaml

                # Rollout status to ensure deployment succeeded
                kubectl rollout status deployment/devops-html-app
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
