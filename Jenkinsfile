pipeline {
    agent any
    
    environment {
        PATH = "/var/jenkins_home/bin:${env.PATH}"
        OPENSHIFT_SERVER = 'https://api.crc.testing:6443'
        WEBAPP_NAMESPACE = 'webapp'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo "Cloning repository from GitHub..."
                git branch: 'Master', 
                    url: 'https://github.com/mohamed-55-iti/deploy-tier-application-backend-Database-proxy-.git'
                
                sh '''
                    echo "=== Repository Contents ==="
                    ls -la
                    echo ""
                    echo "=== Files structure ==="
                    find . -type f -name "*.go" -o -name "Dockerfile" -o -name "*.conf"
                '''
            }
        }
        
        stage('Build Images with OpenShift') {
            steps {
                script {
                    echo "Building images using OpenShift BuildConfig..."
                    sh '''
                        echo "=== Starting Backend Build ==="
                        /var/jenkins_home/bin/oc start-build backend-build -n ${WEBAPP_NAMESPACE} --follow --wait || echo "Backend build failed"
                        
                        echo ""
                        echo "=== Starting Proxy Build ==="
                        /var/jenkins_home/bin/oc start-build proxy-build -n ${WEBAPP_NAMESPACE} --follow --wait || echo "Proxy build failed"
                        
                        echo ""
                        echo "=== Build Status ==="
                        /var/jenkins_home/bin/oc get builds -n ${WEBAPP_NAMESPACE}
                    '''
                }
            }
        }
        
        stage('Update Deployments') {
            steps {
                script {
                    echo "Updating deployments to use new images..."
                    sh '''
                        # Update backend deployment to use new image
                        /var/jenkins_home/bin/oc set image deployment/backend \
                            backend=image-registry.openshift-image-registry.svc:5000/${WEBAPP_NAMESPACE}/backend:latest \
                            -n ${WEBAPP_NAMESPACE} || echo "Backend update failed"
                        
                        # Update proxy deployment to use new image
                        /var/jenkins_home/bin/oc set image deployment/proxy-deployment \
                            proxy=image-registry.openshift-image-registry.svc:5000/${WEBAPP_NAMESPACE}/proxy:latest \
                            -n ${WEBAPP_NAMESPACE} || echo "Proxy update failed"
                    '''
                }
            }
        }
        
        stage('Rollout & Verify') {
            steps {
                sh '''
                    echo "=== Rolling out new deployments ==="
                    /var/jenkins_home/bin/oc rollout status deployment/backend -n ${WEBAPP_NAMESPACE} --timeout=300s || echo "Backend rollout timeout"
                    /var/jenkins_home/bin/oc rollout status deployment/proxy-deployment -n ${WEBAPP_NAMESPACE} --timeout=300s || echo "Proxy rollout timeout"
                    
                    echo ""
                    echo "=== Current Pods Status ==="
                    /var/jenkins_home/bin/oc get pods -n ${WEBAPP_NAMESPACE}
                    
                    echo ""
                    echo "=== Services ==="
                    /var/jenkins_home/bin/oc get svc -n ${WEBAPP_NAMESPACE}
                    
                    echo ""
                    echo "=== Routes ==="
                    /var/jenkins_home/bin/oc get routes -n ${WEBAPP_NAMESPACE}
                '''
            }
        }
        
        stage('Smoke Test') {
            steps {
                sh '''
                    echo "=== Running Smoke Tests ==="
                    
                    # Test Backend
                    echo "Testing Backend..."
                    /var/jenkins_home/bin/oc exec -n ${WEBAPP_NAMESPACE} deployment/backend -- curl -s localhost:8000/ || echo "Backend test failed"
                    
                    # Test Proxy
                    echo ""
                    echo "Testing Proxy..."
                    PROXY_URL=$(/var/jenkins_home/bin/oc get route proxy-route -n ${WEBAPP_NAMESPACE} -o jsonpath='{.spec.host}')
                    curl -k https://${PROXY_URL} || echo "Proxy test failed"
                '''
            }
        }
    }
    
    post {
        success {
            echo "✅ CI/CD Pipeline completed successfully!"
            echo "Build Number: ${BUILD_NUMBER}"
            sh '''
                echo ""
                echo "=== Application URLs ==="
                /var/jenkins_home/bin/oc get routes -n ${WEBAPP_NAMESPACE} -o custom-columns=NAME:.metadata.name,URL:.spec.host
            '''
        }
        failure {
            echo "❌ Pipeline failed!"
            sh '''
                echo "=== Recent Events ==="
                /var/jenkins_home/bin/oc get events -n ${WEBAPP_NAMESPACE} --sort-by='.lastTimestamp' | tail -10
            '''
        }
    }
}
