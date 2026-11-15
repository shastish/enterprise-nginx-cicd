pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '30'))
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }

    parameters {
        string(name: 'AWS_REGION', defaultValue: 'ap-south-1', description: 'AWS Region containing ECR and EKS.')
        string(name: 'ECR_REPOSITORY', defaultValue: 'enterprise-nginx-cicd', description: 'Pre-created ECR repository with immutable tags.')
        string(name: 'EKS_CLUSTER', defaultValue: 'enterprise-nginx-eks', description: 'Pre-created EKS cluster.')
        string(name: 'K8S_NAMESPACE', defaultValue: 'nginx', description: 'Namespace for the release.')
        booleanParam(name: 'PUBLISH_TO_ECR', defaultValue: false, description: 'Publish the scanned image from main to ECR.')
        booleanParam(name: 'DEPLOY_TO_EKS', defaultValue: false, description: 'Deploy the published image from main to EKS.')
    }

    environment {
        LOCAL_IMAGE = 'enterprise-nginx-cicd'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.IMAGE_TAG = sh(
                        returnStdout: true,
                        script: 'git rev-parse --short=12 HEAD'
                    ).trim()
                }
            }
        }

        stage('Validate') {
            steps {
                script {
                    if (!(params.AWS_REGION ==~ /[a-z]{2}(-[a-z0-9]+)+-[0-9]/)) {
                        error('AWS_REGION must be a valid region identifier, for example ap-south-1.')
                    }
                    if (!(params.ECR_REPOSITORY ==~ /[a-z0-9]+([._/-][a-z0-9]+)*/)) {
                        error('ECR_REPOSITORY contains invalid characters.')
                    }
                    if (!(params.EKS_CLUSTER ==~ /[A-Za-z0-9][A-Za-z0-9_-]{0,99}/)) {
                        error('EKS_CLUSTER contains invalid characters.')
                    }
                    if (!(params.K8S_NAMESPACE ==~ /[a-z0-9]([-a-z0-9]*[a-z0-9])?/)) {
                        error('K8S_NAMESPACE must be a DNS label.')
                    }
                    if (params.DEPLOY_TO_EKS && !params.PUBLISH_TO_ECR) {
                        error('DEPLOY_TO_EKS requires PUBLISH_TO_ECR.')
                    }
                }
                sh '''
                    set -eu
                    command -v helm
                    helm lint helm/nginx
                    helm template nginx-site helm/nginx --namespace "$K8S_NAMESPACE" > /tmp/nginx-helm-rendered.yaml
                    test -s /tmp/nginx-helm-rendered.yaml
                '''
            }
        }

        stage('Scan source') {
            steps {
                sh '''
                    set -eu
                    trivy fs --scanners vuln,misconfig,secret \
                        --severity CRITICAL,HIGH --exit-code 1 --ignore-unfixed .
                '''
            }
        }

        stage('Build image') {
            steps {
                sh '''
                    set -eu
                    docker build --pull --tag "$LOCAL_IMAGE:$IMAGE_TAG" .
                '''
            }
        }

        stage('Scan image') {
            steps {
                sh '''
                    set -eu
                    trivy image --severity CRITICAL,HIGH --exit-code 1 \
                        --ignore-unfixed "$LOCAL_IMAGE:$IMAGE_TAG"
                '''
            }
        }

        stage('Publish to ECR') {
            when {
                allOf {
                    branch 'main'
                    expression { params.PUBLISH_TO_ECR }
                }
            }
            steps {
                script {
                    def accountId = sh(
                        returnStdout: true,
                        script: 'aws sts get-caller-identity --query Account --output text'
                    ).trim()
                    if (!(accountId ==~ /[0-9]{12}/)) {
                        error('AWS identity lookup returned an invalid account ID.')
                    }
                    env.ECR_REGISTRY = "${accountId}.dkr.ecr.${params.AWS_REGION}.amazonaws.com"
                    env.IMAGE_URI = "${env.ECR_REGISTRY}/${params.ECR_REPOSITORY}:${env.IMAGE_TAG}"
                }
                withEnv(["AWS_REGION=${params.AWS_REGION}", "ECR_REPOSITORY=${params.ECR_REPOSITORY}"]) {
                    sh '''
                        set -eu
                        aws ecr describe-repositories --region "$AWS_REGION" \
                            --repository-names "$ECR_REPOSITORY" >/dev/null
                        aws ecr get-login-password --region "$AWS_REGION" \
                            | docker login --username AWS --password-stdin "$ECR_REGISTRY"
                        docker tag "$LOCAL_IMAGE:$IMAGE_TAG" "$IMAGE_URI"
                        docker push "$IMAGE_URI"
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            when {
                allOf {
                    branch 'main'
                    expression { params.DEPLOY_TO_EKS }
                }
            }
            steps {
                withEnv([
                    "AWS_REGION=${params.AWS_REGION}",
                    "EKS_CLUSTER=${params.EKS_CLUSTER}",
                    "K8S_NAMESPACE=${params.K8S_NAMESPACE}"
                ]) {
                    sh '''
                        set -eu
                        aws eks update-kubeconfig --region "$AWS_REGION" --name "$EKS_CLUSTER"
                        helm upgrade --install nginx-site helm/nginx \
                            --namespace "$K8S_NAMESPACE" --create-namespace \
                            --wait --atomic --timeout 5m \
                            --set-string image.repository="${IMAGE_URI%:*}" \
                            --set-string image.tag="$IMAGE_TAG"
                        kubectl rollout status deployment/nginx-site \
                            --namespace "$K8S_NAMESPACE" --timeout=180s
                        kubectl get deployment,pods,service,endpoints \
                            --namespace "$K8S_NAMESPACE" --output wide
                        kubectl exec --namespace "$K8S_NAMESPACE" deployment/nginx-site \
                            --container nginx -- wget -qO- http://127.0.0.1:8080/healthz \
                            | grep -qx ok
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker image rm "$LOCAL_IMAGE:$IMAGE_TAG" >/dev/null 2>&1 || true'
        }
    }
}
