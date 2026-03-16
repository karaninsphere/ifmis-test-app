pipeline {
    agent any
    
    triggers {
        // Trigger on push to main branch
        githubPush()
    }
    
    environment {
        // GCP Configuration - UPDATED WITH YOUR VALUES
        PROJECT_ID = 'df-cg-ifmis-chhattisgarh'
        CLUSTER_NAME = 'ifmis-stg-cluster'
        CLUSTER_ZONE = 'asia-south1-b'
        APP_NAME = 'ifmis-test-app'
        NAMESPACE = 'default'
        ARTIFACT_REPO = 'asia-south1-docker.pkg.dev/df-cg-ifmis-chhattisgarh/ifmis-stg-repo'
        
        // Docker image tags
        DOCKER_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.substring(0,7)}"
        
        // GCP Service Account from Jenkins credentials - USING YOUR SA NAME
        GCP_SA_KEY = credentials('gcp-jenkins-sa')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Authenticate with GCP') {
            steps {
                script {
                    sh """
                        # Authenticate with service account
                        gcloud auth activate-service-account --key-file=${GCP_SA_KEY}
                        gcloud config set project ${PROJECT_ID}
                        
                        # Configure Docker for Artifact Registry (asia-south1 region)
                        gcloud auth configure-docker asia-south1-docker.pkg.dev --quiet
                        
                        # Get GKE credentials for asia-south1-b
                        gcloud container clusters get-credentials ${CLUSTER_NAME} \\
                            --zone ${CLUSTER_ZONE} \\
                            --project ${PROJECT_ID}
                    """
                }
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }
        
        stage('Run Tests') {
            steps {
                sh 'npm test'
            }
        }
        
        stage('Build and Push Docker Image') {
            steps {
                script {
                    // Build with multiple tags
                    def imageLatest = "${env.ARTIFACT_REPO}/${APP_NAME}:latest"
                    def imageVersion = "${env.ARTIFACT_REPO}/${APP_NAME}:${env.DOCKER_TAG}"
                    
                    sh """
                        echo "Building Docker image: ${imageLatest}"
                        docker build -t ${imageLatest} .
                        docker tag ${imageLatest} ${imageVersion}
                        
                        echo "Pushing images to Artifact Registry..."
                        docker push ${imageLatest}
                        docker push ${imageVersion}
                    """
                }
            }
        }
        
        stage('Deploy to GKE') {
            steps {
                script {
                    // Update the image in the deployment
                    def imageVersion = "${env.ARTIFACT_REPO}/${APP_NAME}:${env.DOCKER_TAG}"
                    
                    // Check if deployment exists
                    def deploymentExists = sh(
                        script: "kubectl get deployment ${APP_NAME} -n ${NAMESPACE} --ignore-not-found",
                        returnStdout: true
                    ).trim()
                    
                    if (deploymentExists) {
                        // Update existing deployment
                        sh """
                            echo "Updating existing deployment..."
                            kubectl set image deployment/${APP_NAME} \\
                                ${APP_NAME}=${imageVersion} \\
                                -n ${NAMESPACE}
                                
                            kubectl rollout status deployment/${APP_NAME} \\
                                -n ${NAMESPACE} --timeout=60s
                        """
                    } else {
                        // Create new deployment and service
                        sh """
                            echo "Creating new deployment and service..."
                            cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
  labels:
    app: ${APP_NAME}
    commit: ${env.GIT_COMMIT.substring(0,7)}
    build: ${env.BUILD_NUMBER}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ${APP_NAME}
  template:
    metadata:
      labels:
        app: ${APP_NAME}
        commit: ${env.GIT_COMMIT.substring(0,7)}
    spec:
      containers:
      - name: ${APP_NAME}
        image: ${imageVersion}
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: BUILD_NUMBER
          value: "${env.BUILD_NUMBER}"
        - name: GIT_COMMIT
          value: "${env.GIT_COMMIT}"
        - name: PROJECT_ID
          value: "${env.PROJECT_ID}"
        - name: CLUSTER_NAME
          value: "${env.CLUSTER_NAME}"
        - name: CLUSTER_ZONE
          value: "${env.CLUSTER_ZONE}"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
spec:
  selector:
    app: ${APP_NAME}
  ports:
  - port: 80
    targetPort: 3000
  type: ClusterIP
EOF
                        """
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    // Wait for pods to be ready
                    sh """
                        echo "Waiting for pods to be ready..."
                        kubectl wait --for=condition=ready pod \\
                            -l app=${APP_NAME} \\
                            -n ${NAMESPACE} \\
                            --timeout=60s || true
                        
                        echo ""
                        echo "========================================="
                        echo "=== Deployment Status ==="
                        echo "========================================="
                        kubectl get pods -n ${NAMESPACE} -l app=${APP_NAME}
                        
                        echo ""
                        echo "========================================="
                        echo "=== Service Status ==="
                        echo "========================================="
                        kubectl get svc -n ${NAMESPACE} ${APP_NAME}
                        
                        echo ""
                        echo "========================================="
                        echo "=== Recent Pod Logs ==="
                        echo "========================================="
                        kubectl logs -n ${NAMESPACE} -l app=${APP_NAME} --tail=20 || true
                    """
                }
            }
        }
    }
    
    post {
        success {
            echo '========================================='
            echo '✅🚀 PIPELINE EXECUTED SUCCESSFULLY! 🚀✅'
            echo '========================================='
            echo "Application: ${APP_NAME}"
            echo "Project: ${PROJECT_ID}"
            echo "Cluster: ${CLUSTER_NAME} (${CLUSTER_ZONE})"
            echo "Namespace: ${NAMESPACE}"
            echo "Image Tag: ${env.DOCKER_TAG}"
            echo ""
            echo "To access your application locally:"
            echo "kubectl port-forward -n ${NAMESPACE} svc/${APP_NAME} 8080:80"
            echo ""
            echo "Then open http://localhost:8080 in your browser"
            echo '========================================='
        }
        failure {
            echo '========================================='
            echo '❌❌ PIPELINE EXECUTION FAILED! ❌❌'
            echo '========================================='
            echo "Check the Jenkins console output for details"
            echo '========================================='
        }
        always {
            // Clean up Docker images if needed
            sh 'docker system prune -f || true'
        }
    }
}