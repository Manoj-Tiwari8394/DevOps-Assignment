pipeline {
    agent any

    environment {
        PROJECT_ID = "DrvOps_Project"
        IMAGE_NAME = "sync-service"
        GCR_REPO = "asia-south1-docker.pkg.dev/${PROJECT_ID}/cloudeagle"
        env = sh(returnStdout: true, script: '''
            if [[ $BRANCH_NAME == refs/tags/* ]]; then
                echo "prod"
            elif [[ $BRANCH_NAME == release/* ]]; then
                echo "staging"
            elif [[ $BRANCH_NAME == develop ]]; then
                echo "qa"
            else
                echo "pr"
            fi
        ''').trim()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    echo "Branch: ${BRANCH_NAME}"
                    echo "Env: ${env}"
                }
            }
        }

        stage('Build') {
            when {
                expression { env != "pr" }
            }
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            when {
                expression { env != "pr" }
            }
            steps {
                sh 'mvn test'
            }
        }

        stage('Code Quality') {
            steps {
                sh 'echo "Run SonarQube / Checkstyle here"'
            }
        }

        stage('Build Docker Image') {
            when {
                expression { env != "pr" }
            }
            steps {
                script {
                    def TAG = "latest"
                    if (env == "prod") {
                        TAG = sh(returnStdout: true, script: 'git describe --tags --exact-match || echo "manual"').trim()
                    } else if (env == "staging") {
                        TAG = "staging-${GIT_COMMIT.take(7)}"
                    } else if (env == "qa") {
                        TAG = "qa-${BUILD_NUMBER}"
                    }

                    sh """
                        docker build -t ${GCR_REPO}/${IMAGE_NAME}:${TAG} .
                    """
                }
            }
        }

        stage('Push Image') {
            when {
                expression { env != "pr" }
            }
            steps {
                sh '''
                    gcloud auth configure-docker asia-south1-docker.pkg.dev
                    docker push ${GCR_REPO}/${IMAGE_NAME}:${TAG}
                '''
            }
        }

        stage('Deploy to QA (PR)') {
            when {
                expression { env == "pr" && params.DEPLOY_TO_QA }
            }
            steps {
                sh 'echo "Deploy ephemeral test container / VM"'
            }
        }

        stage('Deploy to QA') {
            when {
                expression { env == "qa" }
            }
            steps {
                sh '''
                    echo "Deploy to QA VMs"
                    # Example: SSH into VMs and restart Docker container or systemd service
                '''
            }
        }

        stage('Deploy to Staging') {
            when {
                expression { env == "staging" }
            }
            steps {
                sh '''
                    echo "Deploy to staging VMs"
                '''
            }
        }

        stage('Deploy to Prod (Manual)') {
            when {
                expression { env == "prod" }
            }
            steps {
                input message: 'Deploy to PROD?', ok: 'Deploy'

                script {
                    sh '''
                        echo "Blue/Green: update GCP LB to new instance group"
                    '''
                }
            }
        }

        stage('Rollback if Prod Failed') {
            when {
                expression { env == "prod" && currentBuild.result == "FAILURE" }
            }
            steps {
                script {
                    echo "Revert GCP LB to previous healthy backend"
                }
            }
        }
    }

    post {
        success {
            script {
                echo "Build success: ${env} deploy completed."
            }
        }
        failure {
            script {
                echo "Build failed. Check logs and retry."
            }
        }
    }
}
