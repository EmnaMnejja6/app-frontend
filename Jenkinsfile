pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
metadata:
  name: kaniko
spec:
  serviceAccountName: jenkins
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command: [sleep]
    args: [infinity]
    volumeMounts:
    - name: kaniko-secret
      mountPath: /kaniko/.docker
  - name: kubectl
    image: bitnami/kubectl:latest
    command: [sleep]
    args: [infinity]
    securityContext:
      runAsUser: 0
  volumes:
  - name: kaniko-secret
    secret:
      secretName: kaniko-nexus-secret
      items:
      - key: config.json
        path: config.json
"""
    }
  }

  environment {
    NEXUS_REGISTRY = '192.168.58.13:8082'
    IMAGE_NAME     = "${NEXUS_REGISTRY}/app-frontend"
    IMAGE_TAG      = "${GIT_COMMIT}"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Push with Kaniko') {
      steps {
        container('kaniko') {
          sh """
            /kaniko/executor \
              --dockerfile=Dockerfile \
              --context=dir://\${WORKSPACE} \
              --destination=${IMAGE_NAME}:${IMAGE_TAG} \
              --destination=${IMAGE_NAME}:latest \
              --insecure \
              --skip-tls-verify
          """
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        container('kubectl') {
          sh """
            sed -i 's|192.168.58.13:8082/app-frontend:latest|${IMAGE_NAME}:${IMAGE_TAG}|g' \
              k8s/frontend-deployment.yaml

            kubectl apply -f k8s/frontend-deployment.yaml

            kubectl rollout status deployment/frontend \
              -n default \
              --timeout=120s
          """
        }
      }
    }
  }

  post {
    failure { echo 'Pipeline failed.' }
    success { echo "Deployed ${IMAGE_NAME}:${IMAGE_TAG}" }
  }
}
