pipeline {
    agent any
    
    environment {
        PROJECT_ID = credentials('GCP_PROJECT_ID')
        CLUSTER_NAME = credentials('GKE_CLUSTER_NAME')
        LOCATION = credentials('GKE_CLUSTER_LOCATION')
        CREDENTIALS_ID = credentials('GCP_CREDENTIALS_ID')
        REGISTRY_NAME = credentials('ARTIFACT_REGISTRY_NAME')
        REGION = credentials('GCP_REGION')
        APP_NAME = credentials('APP_NAME')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Authenticate to GCP') {
            steps {
                script {
                    withCredentials([file(credentialsId: env.CREDENTIALS_ID, variable: 'GC_KEY')]) {
                        sh 'gcloud auth activate-service-account --key-file=${GC_KEY}'
                        sh "gcloud config set project ${PROJECT_ID}"
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    def imageTag = "eu.${REGION}.artifactregistry.googleapis.com/${PROJECT_ID}/${REGISTRY_NAME}/${APP_NAME}:${BUILD_NUMBER}"
                    sh """
                        gcloud auth configure-docker ${REGION}-docker.pkg.dev
                        docker build -t ${imageTag} .
                    """
                }
            }
        }
        
        stage('Push to Artifact Registry') {
            steps {
                script {
                    def imageTag = "eu.${REGION}.artifactregistry.googleapis.com/${PROJECT_ID}/${REGISTRY_NAME}/${APP_NAME}:${BUILD_NUMBER}"
                    sh "docker push ${imageTag}"
                }
            }
        }
        
        stage('Deploy to GKE') {
            steps {
                script {
                    sh """
                        gcloud container clusters get-credentials ${CLUSTER_NAME} --region ${LOCATION} --project ${PROJECT_ID}
                        
                        # Update kubernetes deployment with new image
                        kubectl set image deployment/${APP_NAME} ${APP_NAME}=eu.${REGION}.artifactregistry.googleapis.com/${PROJECT_ID}/${REGISTRY_NAME}/${APP_NAME}:${BUILD_NUMBER}
                    """
                }
            }
        }
    }
    
    post {
        always {
            // Clean up
            sh 'gcloud auth revoke --all'
            cleanWs()
        }
    }
}