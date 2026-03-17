pipeline {
    agent any
    
    triggers {
        githubPush()
    }
    
    environment {
        PROJECT_ID = 'df-cg-ifmis-chhattisgarh'
        CLUSTER_NAME = 'ifmis-stg-cluster'
        CLUSTER_ZONE = 'asia-south1-b'
        APP_NAME = 'ifmis-test-app'
        NAMESPACE = 'default'
        ARTIFACT_REPO = 'asia-south1-docker.pkg.dev/df-cg-ifmis-chhattisgarh/ifmis-stg-repo'
        DOCKER_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.substring(0,7)}"
        GCP_SA_KEY = credentials('gcp-jenkins-sa')
    }
    
    stages {
        // Stage 1: Matches "Declarative: Checkout SCM"
        stage('Declarative: Checkout SCM') {
            steps {
                checkout scm
            }
        }
        
        // Stage 2: Combined Auth, Install, and Docker Build into "DEV_BUILD"
        stage('DEV_BUILD') {
            steps {
                script {
                    sh """
                        gcloud auth activate-service-account --key-file=${GCP_SA_KEY}
                        gcloud config set project ${PROJECT_ID}
                        gcloud auth configure-docker asia-south1-docker.pkg.dev --quiet
                        gcloud container clusters get-credentials ${CLUSTER_NAME} --zone ${CLUSTER_ZONE} --project ${PROJECT_ID}
                        
                        npm install
                        npm test
                        
                        docker build -t ${env.ARTIFACT_REPO}/${APP_NAME}:latest .
                        docker tag ${env.ARTIFACT_REPO}/${APP_NAME}:latest ${env.ARTIFACT_REPO}/${APP_NAME}:${env.DOCKER_TAG}
                    """
                }
            }
        }
        
        // Stage 3: Matches "DEV_ECR PUSH" (Adjusted for your GCP Artifact Registry)
        stage('DEV_ECR PUSH') {
            steps {
                sh """
                    docker push ${env.ARTIFACT_REPO}/${APP_NAME}:latest
                    docker push ${env.ARTIFACT_REPO}/${APP_NAME}:${env.DOCKER_TAG}
                """
            }
        }
        
        // Stage 4: Matches "DEV_DEPLOY"
        stage('DEV_DEPLOY') {
            steps {
                script {
                    def imageVersion = "${env.ARTIFACT_REPO}/${APP_NAME}:${env.DOCKER_TAG}"
                    def deploymentExists = sh(script: "kubectl get deployment ${APP_NAME} -n ${NAMESPACE} --ignore-not-found", returnStdout: true).trim()
                    
                    if (deploymentExists) {
                        sh "kubectl set image deployment/${APP_NAME} ${APP_NAME}=${imageVersion} -n ${NAMESPACE}"
                        sh "kubectl rollout status deployment/${APP_NAME} -n ${NAMESPACE} --timeout=60s"
                    } else {
                        // Your existing HEREDOC for creating new deployment
                        sh """
                            cat <<EOF | kubectl apply -f -
                            # ... (Keep your existing Kubernetes YAML here) ...
EOF
                        """
                    }
                }
            }
        }
        
        // Stage 5: Matches "POST_CHECKS"
        stage('POST_CHECKS') {
            steps {
                sh """
                    kubectl wait --for=condition=ready pod -l app=${APP_NAME} -n ${NAMESPACE} --timeout=60s || true
                    kubectl get pods -n ${NAMESPACE} -l app=${APP_NAME}
                """
            }
        }
    }
    
    // This section automatically creates the "Declarative: Post Actions" column
    post {
        success {
            echo '✅ PIPELINE EXECUTED SUCCESSFULLY!'
        }
        failure {
            echo '❌ PIPELINE EXECUTION FAILED!'
        }
        always {
            sh 'docker system prune -f || true'
        }
    }
}
