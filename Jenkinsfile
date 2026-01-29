pipeline {
    agent any

    tools {
        jdk 'java17'
        maven 'maven3'
    }

    environment {
        SONAR_SCANNER_HOME = tool 'sonar7'

        IMAGE_NAME = "java-app"
        IMAGE_TAG = "${BUILD_NUMBER}"

        GCP_PROJECT_ID = "focal-dock-440200-u5"
        ARTIFACT_REPO = "java-app-repo-02"
        REGION = "us-central1"

        FULL_IMAGE_NAME = "us-docker.pkg.dev/${GCP_PROJECT_ID}/${ARTIFACT_REPO}/${IMAGE_NAME}:${IMAGE_TAG}"
        SERVICE_NAME = "java-app-service"
    }

    stages {

        stage('Initialize') {
            steps {
                echo 'Initializing pipeline...'
                sh 'java -version'
                sh 'mvn -version'
                sh 'docker --version'
                sh 'gcloud --version'
            }
        }

        stage('Checkout Source') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        credentialsId: 'jenkins-gcp',
                        url: 'https://github.com/iQuantC/Jenkins_GCP_CloudRun.git'
                    ]]
                )
            }
        }

        stage('Build with Maven') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('JUnit Tests') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonartoken', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('sonar') {
                        sh """
                        ${SONAR_SCANNER_HOME}/bin/sonar-scanner \
                          -Dsonar.projectKey=jenkinsgcp \
                          -Dsonar.sources=. \
                          -Dsonar.java.binaries=target/classes \
                          -Dsonar.token=$SONAR_TOKEN
                        """
                    }
                }
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh 'trivy fs . --severity HIGH,CRITICAL --format table -o fs-scan.txt'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh """
                trivy image ${IMAGE_NAME}:${IMAGE_TAG} \
                  --severity HIGH,CRITICAL \
                  --no-progress \
                  --format table \
                  -o image-scan.txt
                """
            }
        }

        stage('Authenticate & Push to Artifact Registry') {
            steps {
                withCredentials([file(credentialsId: 'gcpjmsa', variable: 'GCP_KEY')]) {
                    sh """
                    gcloud auth activate-service-account --key-file=$GCP_KEY
                    gcloud config set project $GCP_PROJECT_ID
                    gcloud auth configure-docker us-docker.pkg.dev --quiet

                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${FULL_IMAGE_NAME}
                    docker push ${FULL_IMAGE_NAME}
                    """
                }
            }
        }

        stage('Deploy to Cloud Run') {
            steps {
                withCredentials([file(credentialsId: 'gcpjmsa', variable: 'GCP_KEY')]) {
                    sh """
                    gcloud run deploy $SERVICE_NAME \
                      --image=$FULL_IMAGE_NAME \
                      --region=$REGION \
                      --platform=managed \
                      --allow-unauthenticated \
                      --port=8090 \
                      --memory=512Mi \
                      --quiet
                    """
                }
            }
        }

        stage('Get Service URL') {
            steps {
                sh """
                gcloud run services describe $SERVICE_NAME \
                  --region $REGION \
                  --format='value(status.url)'
                """
            }
        }
    }
}
