pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'your-docker-registry'
        AWS_REGION = 'us-east-1'
        EKS_CLUSTER_NAME = 'emartapp-cluster'
        ARGOCD_SERVER = 'argocd-server'
        ARGOCD_NAMESPACE = 'argocd'
        APP_NAME = "devops"
        RELEASE = "1.0.0"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Security Scan') {
            steps {
                script {
                    // Scan Kubernetes manifests
                    sh 'kubesec scan k8s/base/*.yaml > kubesec-report.json'
                    
                    // Scan Docker images
                    sh """
                        trivy image ${DOCKER_REGISTRY}/emartapp-backend:${BUILD_NUMBER} --format json > trivy-backend-report.json
                        trivy image ${DOCKER_REGISTRY}/emartapp-frontend:${BUILD_NUMBER} --format json > trivy-frontend-report.json
                    """
                    
                    // Check for critical vulnerabilities
                    sh """
                        if grep -q '"Severity": "CRITICAL"' trivy-backend-report.json trivy-frontend-report.json; then
                            echo "Critical vulnerabilities found!"
                            exit 1
                        fi
                    """
                }
            }
        }
        
        stage('Update Kustomization') {
            steps {
                script {
                    sh """
                        cd k8s/overlays/development
                        kustomize edit set image backend=${DOCKER_REGISTRY}/emartapp-backend:${BUILD_NUMBER}
                        kustomize edit set image frontend=${DOCKER_REGISTRY}/emartapp-frontend:${BUILD_NUMBER}
                    """
                }
            }
        }
        
        stage('Commit Changes') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        sh """
                            git config user.name "${GIT_USERNAME}"
                            git config user.email "${GIT_USERNAME}@example.com"
                            git add .
                            git commit -m "Update image tags to ${BUILD_NUMBER}"
                            git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/your-username/devops-gitops.git HEAD:main
                        """
                    }
                }
            }
        }
        
        stage('Verify ArgoCD Sync') {
            steps {
                script {
                    withAWS(credentials: 'aws-credentials', region: env.AWS_REGION) {
                        sh """
                            aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}
                            kubectl wait --for=condition=healthy --timeout=300s application/emartapp -n ${ARGOCD_NAMESPACE}
                        """
                    }
                }
            }
        }
        
        stage('Verify Security Policies') {
            steps {
                script {
                    withAWS(credentials: 'aws-credentials', region: env.AWS_REGION) {
                        sh """
                            aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}
                            
                            # Verify Network Policies
                            kubectl get networkpolicy -n emartapp
                            
                            # Verify Pod Security Policies
                            kubectl get psp emartapp-psp
                            
                            # Verify Security Contexts
                            kubectl get pods -n emartapp -o json | jq '.items[].spec.securityContext'
                        """
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    withAWS(credentials: 'aws-credentials', region: env.AWS_REGION) {
                        sh """
                            aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_REGION}
                            
                            # Verify deployments
                            kubectl rollout status deployment/backend -n emartapp
                            kubectl rollout status deployment/frontend -n emartapp
                            
                            # Verify services
                            kubectl get svc -n emartapp
                            
                            # Verify ingress
                            kubectl get ingress -n emartapp
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            emailext (
                subject: "CD Pipeline Successful: ${currentBuild.fullDisplayName}",
                body: "Your CD pipeline has completed successfully. Check console output at ${env.BUILD_URL}",
                to: '${DEFAULT_RECIPIENTS}'
            )
        }
        failure {
            emailext (
                subject: "CD Pipeline Failed: ${currentBuild.fullDisplayName}",
                body: "Your CD pipeline has failed. Check console output at ${env.BUILD_URL}",
                to: '${DEFAULT_RECIPIENTS}'
            )
        }
    }
}
