pipeline {
    agent any
    
    environment {
        PROJECT_ID = credentials('GCP_PROJECT_ID')
        CLUSTER_NAME = credentials('GKE_CLUSTER_NAME')
        LOCATION = credentials('GKE_CLUSTER_LOCATION')
        CREDENTIALS_ID = 'GCP_CREDENTIALS_ID'
        REGISTRY_NAME = credentials('ARTIFACT_REGISTRY_NAME')
        REGION = credentials('GCP_REGION')
        APP_NAME = credentials('APP_NAME')
        REPO_NAME = credentials('REPO_NAME')
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
                    try {
                        echo "Iniciando autenticação no GCP..."
                        echo "Usando credencial com ID: ${CREDENTIALS_ID}"
                        withCredentials([file(credentialsId: CREDENTIALS_ID, variable: 'GC_KEY')]) {
                            echo "Credenciais carregadas, ativando conta de serviço..."
                            sh "gcloud auth activate-service-account --key-file=\"${GC_KEY}\""
                            echo "Configurando projeto GCP..."
                            sh "gcloud config set project ${PROJECT_ID}"
                            echo "Autenticação concluída com sucesso"
                        }
                    } catch (Exception e) {
                        echo "Erro durante a autenticação: ${e.message}"
                        currentBuild.result = 'FAILURE'
                        throw e
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    def imageTag = "${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_NAME}/${APP_NAME}:${BUILD_NUMBER}".toLowerCase()
                    sh """
                        gcloud auth configure-docker ${REGION}-docker.pkg.dev --quiet
                        docker build -t ${imageTag} .
                    """
                }
            }
        }
        
        stage('Push to Artifact Registry') {
            steps {
                script {
                    def imageTag = "${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_NAME}/${APP_NAME}:${BUILD_NUMBER}".toLowerCase()
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
                        kubectl set image deployment/${APP_NAME} ${APP_NAME}="${REGION}-docker.pkg.dev/${PROJECT_ID}/${REGISTRY_NAME}/${APP_NAME}:${BUILD_NUMBER}".toLowerCase()
                    """
                }
            }
        }
    }
    
    post {
        success {
            script {
                if (currentBuild.getBuildCauses().toString().contains('Authenticate to GCP') && currentBuild.currentResult == 'SUCCESS') {
                    sh 'gcloud auth revoke --all'
                    cleanWs()
                }
            }
        }
    }
}