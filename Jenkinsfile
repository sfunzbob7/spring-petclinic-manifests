pipeline {
    agent any

    parameters {
        string(name: 'IMAGE_TAG', defaultValue: "build-${env.BUILD_NUMBER}", description: 'Docker image tag for version A')
    }

    environment {
        GIT_REPO_APP = "https://github.com/sfunzbob7/spring-petclinic.git"
        GIT_REPO_MANIFESTS = "https://github.com/sfunzbob7/sample-app-manifests.git"
        GIT_BRANCH = "main"
        GIT_BRANCH_MANIFESTS = "master"
        ARGOCD_APP = "spring-app"
        ARGOCD_SERVER = "https://argocd-server-openshift-operators.apps.kyobo.mtp.local"
        ARGOCD_CLI_TOKEN = credentials('admin')
        GIT_CREDENTIAL_ID = 'spring-test-token'
        VIRTUALSERVICE_FILE = "base/virtualservice.yaml"
        OPENSHIFT_PROJECT = "spring-test"
        OPENSHIFT_REGISTRY = "default-route-openshift-image-registry.apps.kyobo.mtp.local"
    }

    stages {
        stage('Checkout App Source') {
            steps {
                git branch: "${GIT_BRANCH}", url: "${GIT_REPO_APP}", credentialsId: "${GIT_CREDENTIAL_ID}"
            }
        }

        stage('Build & Push OpenShift Image (A version)') {
            steps {
                script {
                    def imageFull = "${OPENSHIFT_REGISTRY}/${OPENSHIFT_PROJECT}/spring-app:${params.IMAGE_TAG}"
                    sh """
                        oc project ${OPENSHIFT_PROJECT}
                        if ! oc get bc spring-app >/dev/null 2>&1; then
                            echo "BuildConfig spring-app not found. Creating..."
                            oc new-build --name=spring-app --binary --image-stream=java:latest -n ${OPENSHIFT_PROJECT} --image-stream-namespace=openshift
                            oc patch bc spring-app -p '{"spec":{"output":{"to":{"name":"${imageFull}"}}}}' -n ${OPENSHIFT_PROJECT}
                        else
                            echo "BuildConfig spring-app already exists."
                        fi
                        oc start-build spring-app --from-dir=. --follow \
                            --env APP_VERSION=A
                    """
                }
            }
        }

        stage('Checkout Manifests Repo') {
            steps {
                sh "git clone ${GIT_REPO_MANIFESTS}"
            }
        }

        stage('Update Version A Deployment') {
            steps {
                dir('spring-petclinic-manifests/base') {
                    script {
                        sh """
                            sed -i 's|image: .*|image: ${OPENSHIFT_REGISTRY}/${OPENSHIFT_PROJECT}/spring-app:${params.IMAGE_TAG}|' deployment.yaml
                            git config user.email "sfunzbob7@gmail.com"
                            git config user.name "sfunzbob7"
                            git add deployment.yaml
                            git commit -m "Update A version image to ${params.IMAGE_TAG}"
                            git push origin ${GIT_BRANCH_MANIFESTS}
                        """
                    }
                }
            }
        }

        stage('Ensure VirtualService 100% A') {
            steps {
                dir('spring-petclinic-manifests/base') {
                    script {
                        sh """
                            sed -i 's/weight: [0-9]\\+/weight: 100/' ${VIRTUALSERVICE_FILE}
                            git add ${VIRTUALSERVICE_FILE}
                            git commit -m "Ensure 100% traffic to version A"
                            git push origin ${GIT_BRANCH} || echo "No changes to push"
                        """
                    }
                }
            }
        }

        stage('Sync ArgoCD Application') {
            steps {
                sh """
                    argocd login ${ARGOCD_SERVER} --insecure --username admin --password ${ARGOCD_CLI_TOKEN}
                    argocd app sync ${ARGOCD_APP} --grpc-web
                """
            }
        }
    }

    post {
        success {
            echo "✅ Version A deployed successfully: Image Tag ${params.IMAGE_TAG}, Traffic 100% A"
        }
        failure {
            echo "❌ Pipeline failed. Check logs."
        }
    }
}

