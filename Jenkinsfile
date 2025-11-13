pipeline {
    agent any
    
    environment {
        DOCKERHUB_USERNAME = 'mohamedkhaled55'
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        GITHUB_REPO = 'https://github.com/mohamed-55-iti/deploy-tier-application-backend-Database-proxy-.git'
        PATH = "/var/jenkins_home/bin:${env.PATH}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo "Cloning repository..."
                git branch: 'Master', 
                    url: "${GITHUB_REPO}"
                sh 'ls -la'
            }
        }
        
        stage('Build Backend Image') {
            steps {
                script {
                    echo "Building Backend image..."
                    sh '''
                        docker build -t ${DOCKERHUB_USERNAME}/backend:${BUILD_NUMBER} -f Dockerfile .
                        docker tag ${DOCKERHUB_USERNAME}/backend:${BUILD_NUMBER} ${DOCKERHUB_USERNAME}/backend:latest
                    '''
                }
            }
        }
        
        stage('Build Nginx Proxy Image') {
            steps {
                script {
                    echo "Building Nginx Proxy image..."
                    sh '''
                        # إنشاء Dockerfile للـ Nginx
                        cat > nginx/Dockerfile << 'NGEOF'
FROM nginx:alpine
COPY default.conf /etc/nginx/conf.d/default.conf
EXPOSE 80 443
CMD ["nginx", "-g", "daemon off;"]
NGEOF
                        
                        docker build -t ${DOCKERHUB_USERNAME}/proxy:${BUILD_NUMBER} nginx/
                        docker tag ${DOCKERHUB_USERNAME}/proxy:${BUILD_NUMBER} ${DOCKERHUB_USERNAME}/proxy:latest
                    '''
                }
            }
        }
        
        stage('Push Images to DockerHub') {
            steps {
                script {
                    echo "Pushing images to DockerHub..."
                    sh '''
                        echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin
                        docker push ${DOCKERHUB_USERNAME}/backend:${BUILD_NUMBER}
                        docker push ${DOCKERHUB_USERNAME}/backend:latest
                        docker push ${DOCKERHUB_USERNAME}/proxy:${BUILD_NUMBER}
                        docker push ${DOCKERHUB_USERNAME}/proxy:latest
                    '''
                }
            }
        }
        
        stage('Deploy to OpenShift') {
            steps {
                script {
                    echo "Deploying to webapp namespace..."
                    sh '''
                        # Update backend deployment
                        /var/jenkins_home/bin/oc set image deployment/backend backend=${DOCKERHUB_USERNAME}/backend:${BUILD_NUMBER} -n webapp || echo "Backend deployment not found"
                        
                        # Update proxy deployment
                        /var/jenkins_home/bin/oc set image deployment/proxy-deployment proxy=${DOCKERHUB_USERNAME}/proxy:${BUILD_NUMBER} -n webapp || echo "Proxy deployment not found"
                        
                        # Wait for rollout
                        /var/jenkins_home/bin/oc rollout status deployment/backend -n webapp --timeout=300s || echo "Backend rollout timeout"
                        /var/jenkins_home/bin/oc rollout status deployment/proxy-deployment -n webapp --timeout=300s || echo "Proxy rollout timeout"
                    '''
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "=== Checking Pods Status ==="
                    /var/jenkins_home/bin/oc get pods -n webapp
                    
                    echo ""
                    echo "=== Checking Services ==="
                    /var/jenkins_home/bin/oc get svc -n webapp
                    
                    echo ""
                    echo "=== Checking Routes ==="
                    /var/jenkins_home/bin/oc get routes -n webapp
                '''
            }
        }
    }
    
    post {
        success {
            echo "✅ CI/CD Pipeline completed successfully!"
            echo "Build Number: ${BUILD_NUMBER}"
            echo "Backend Image: ${DOCKERHUB_USERNAME}/backend:${BUILD_NUMBER}"
            echo "Proxy Image: ${DOCKERHUB_USERNAME}/proxy:${BUILD_NUMBER}"
        }
        failure {
            echo "❌ Pipeline failed!"
        }
        always {
            sh 'docker logout || true'
        }
    }
}
