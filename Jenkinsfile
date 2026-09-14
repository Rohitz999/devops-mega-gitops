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
        GITOPS_REPO_URL = "https://github.com/Rohitz999/devops-mega-gitops.git"
        APP_NAME        = "devops-mega-app"
        DOCKER_HUB      = "rohitdockerhub01"
        ARGOCD_SERVER   = "https://argocd.mechnomax.co.in"
        ARGOCD_APP      = "devops-mega-app"
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
                    sh '''
                        rm -rf gitops-tmp
                        git -c credential.helper='!f() { echo "username=$GIT_USER"; echo "password=$GIT_TOKEN"; }; f' \
                            clone ${GITOPS_REPO_URL} gitops-tmp
                    '''
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
                    sh '''
                        cd gitops-tmp
                        git config user.email "jenkins@mechnomax.co.in"
                        git config user.name "Jenkins CI"
                        git add manifests/deployment.yaml
                        git diff --staged --quiet || git commit -m "Update image to ${IMAGE_TAG}"
                        git -c credential.helper='!f() { echo "username=$GIT_USER"; echo "password=$GIT_TOKEN"; }; f' \
                            push origin main
                    '''
                }
            }
        }

        stage('Trigger ArgoCD Sync') {
            steps {
                withCredentials([string(
                    credentialsId: 'argocd-token',
                    variable: 'ARGOCD_TOKEN'
                )]) {
                    sh '''
                        echo "===== Triggering ArgoCD Sync ====="

                        # Give GitHub 2 seconds to register the push
                        sleep 2

                        # Trigger sync
                        echo "Triggering sync for ${ARGOCD_APP}..."
                        SYNC_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
                          -X POST \
                          -H "Authorization: Bearer $ARGOCD_TOKEN" \
                          -H "Content-Type: application/json" \
                          --insecure \
                          "${ARGOCD_SERVER}/api/v1/applications/${ARGOCD_APP}/sync" \
                          -d '{"revision":"HEAD","prune":true}')

                        echo "Sync HTTP: $SYNC_CODE"

                        if [ "$SYNC_CODE" = "200" ]; then
                            echo "✅ ArgoCD sync triggered successfully"
                        else
                            echo "❌ ArgoCD sync failed with HTTP $SYNC_CODE"
                            exit 1
                        fi

                        # Wait for sync to start
                        sleep 5

                        # Get app status
                        echo ""
                        echo "===== ArgoCD App Status ====="
                        curl -s \
                          -H "Authorization: Bearer $ARGOCD_TOKEN" \
                          --insecure \
                          "${ARGOCD_SERVER}/api/v1/applications/${ARGOCD_APP}" \
                          | python3 -c "
import sys, json
try:
    data = json.load(sys.stdin)
    status = data.get('status', {})
    sync = status.get('sync', {})
    health = status.get('health', {})
    print(f\\\"  Sync Status:   {sync.get('status', 'unknown')}\\\")
    print(f\\\"  Health Status: {health.get('status', 'unknown')}\\\")
    rev = sync.get('revision', 'unknown')
    print(f\\\"  Revision:      {rev[:8] if rev != 'unknown' else rev}\\\")
except Exception as e:
    print(f'  (could not parse status: {e})')
"

                        echo "===== ArgoCD Sync Completed ====="
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ GitOps updated: ${APP_NAME} → ${IMAGE_TAG}"
            echo "✅ ArgoCD synced to K3s"
            echo "🌐 Live: https://app.mechnomax.co.in"
        }
        failure {
            echo "❌ GitOps update failed for ${APP_NAME}:${IMAGE_TAG}"
        }
        always {
            cleanWs()
        }
    }
}
