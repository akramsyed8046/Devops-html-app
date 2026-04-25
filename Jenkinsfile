pipeline {
    agent any
    tools {
        nodejs 'Node18'
      
    }
    environment {
        DOCKER_IMAGE         = "akramsyed8046/devops-html-app:latest"
        DOCKER_CREDENTIALS   = "docker-hub"
        SONARQUBE_ENV        = "sonarqube"
        NEXUS_REPO           = "http://35.154.185.85:8081/repository/raw-repo/"
        PATH                 = "${tool 'Node18'}/bin:${env.PATH}"
        KUBECONFIG           = "/var/lib/jenkins/.kube/config"
        AWS_REGION           = "ap-south-1"
        EKS_CLUSTER          = "Mangoes"
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

        stage('Test') {
            steps {
                sh '''
                if npm run | grep -q "test"; then
                    npm test
                else
                    echo "No test script found, skipping"
                fi
                '''
            }
            post {
                always {
                    echo "Test stage completed"
                }
                failure {
                    echo "Tests failed! Check the logs above."
                }
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
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-credentials',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                    VERSION=$(node -p "require('./package.json').version")
                    ARTIFACT="devops-html-app-${VERSION}.zip"

                    zip -r "$ARTIFACT" . -x "*.git*" "node_modules/*"

                    curl -u "$NEXUS_USER:$NEXUS_PASS" \
                         --upload-file "$ARTIFACT" \
                         "${NEXUS_REPO}${ARTIFACT}"

                    echo "Artifact $ARTIFACT uploaded to Nexus successfully"
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

        stage('Run Docker Container') {
            steps {
                sh '''
                docker stop devopscont25 || true
                docker rm devopscont25 || true
                docker run -itd --name devopscont25 -p 8025:80 akramsyed8046/devops-html-app:latest
                echo "Container is running at http://<your-server-ip>:8025"
                '''
            }
        }

        stage('Configure AWS EKS') {
            steps {
                sh """
                aws eks --region ${AWS_REGION} update-kubeconfig --name ${EKS_CLUSTER} --kubeconfig ${KUBECONFIG}
                kubectl --kubeconfig=${KUBECONFIG} get nodes
                """
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                # Check if deployment already exists
                if kubectl get deployment devops-html-app --kubeconfig=${KUBECONFIG} > /dev/null 2>&1; then
                    echo "Deployment exists — applying changes and forcing rollout..."
                    kubectl apply -f deployment.yaml --kubeconfig=${KUBECONFIG}

                    # Force re-pull of latest image (important when using :latest tag)
                    kubectl rollout restart deployment/devops-html-app --kubeconfig=${KUBECONFIG}
                else
                    echo "Fresh deployment — applying for the first time..."
                    kubectl apply -f deployment.yaml --kubeconfig=${KUBECONFIG}
                fi

                # Wait for rollout to finish (timeout 3 mins)
                kubectl rollout status deployment/devops-html-app --kubeconfig=${KUBECONFIG} --timeout=180s

                echo "Deployment successful!"
                """
            }
            post {
                failure {
                    echo "Deployment failed! Rolling back to previous version..."
                    sh "kubectl rollout undo deployment/devops-html-app --kubeconfig=${KUBECONFIG}"
                }
            }
        }

    } // end stages
} // end pipeline
