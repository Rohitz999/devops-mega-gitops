pipeline {
    agent any

    parameters {
        string(
            name: 'IMAGE_TAG',
            defaultValue: 'latest',
            description: 'Docker image tag to deploy'
        )
    }

    environment {
        GITOPS_REPO = "github.com/Rohitz999/devops-mega-gitops.git"
        APP_NAME    = "devops-mega-app"
        DOCKER_HUB  = "rohitdockerhub01"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timeout(time: 10, unit: 'MINUTES')
        timestamps()
    }

    stages {
        stage('Clone GitOps Repo') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-creds',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    sh """
                        rm -rf gitops-tmp
                        git clone https://\${GIT_USER}:\${GIT_TOKEN}@${GITOPS_REPO} gitops-tmp
                    """
                }
            }
        }

        stage('Update Image Tag') {
            steps {
                sh """
                    cd gitops-tmp
                    echo "===== BEFORE ====="
                    grep 'image:' manifests/deployment.yaml
                    sed -i 's|image:.*${APP_NAME}:.*|image: ${DOCKER_HUB}/${APP_NAME}:${IMAGE_TAG}|g' manifests/deployment.yaml
                    echo "===== AFTER ====="
                    grep 'image:' manifests/deployment.yaml
                """
            }
        }

        stage('Commit and Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-creds',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    sh """
                        cd gitops-tmp
                        git config user.email "jenkins@mechnomax.co.in"
                        git config user.name "Jenkins CI"
                        git add manifests/deployment.yaml
                        git diff --staged --quiet || git commit -m "Update ${APP_NAME} image to ${IMAGE_TAG}"
                        git push origin main || echo "Nothing to push"
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ GitOps updated: ${APP_NAME} → ${IMAGE_TAG}"
        }
        failure {
            echo "❌ GitOps update failed"
        }
        always {
            cleanWs()
        }
    }
}
