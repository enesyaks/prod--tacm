pipeline {
  agent {
    kubernetes {
      yaml '''
apiVersion: v1
kind: Pod
metadata:
  namespace: jenkins
spec:
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:v1.23.2-debug
      command: ["/busybox/cat"]
      tty: true
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
  volumes:
    - name: docker-config
      secret:
        secretName: harbor-docker-config
        items:
          - key: .dockerconfigjson
            path: config.json
'''
    }
  }

  environment {
    IMAGE_BASE = "harbor.itacm.site/itacm/itacm"
  }

  stages {
    stage('Determine Environment & Tag') {
      steps {
        script {
          if (env.BRANCH_NAME == 'dev') {
            env.DEPLOY_ENV = 'dev'
            env.IMAGE_TAG = "dev-${GIT_COMMIT}"
          } else if (env.BRANCH_NAME == 'stage') {
            env.DEPLOY_ENV = 'stage'
            env.IMAGE_TAG = "stage-${GIT_COMMIT}"
          } else if (env.BRANCH_NAME == 'main' || env.BRANCH_NAME == 'master') {
            env.DEPLOY_ENV = 'prod'
            env.IMAGE_TAG = "prod-${GIT_COMMIT}"
          } else {
            env.DEPLOY_ENV = 'preview'
            env.IMAGE_TAG = "preview-${GIT_COMMIT}"
          }
          echo "Building for environment: ${DEPLOY_ENV} with tag: ${IMAGE_TAG}"
        }
      }
    }

    stage('Build and Push with Kaniko') {
      steps {
        container('kaniko') {
          sh '''
            echo "Building and pushing to ${IMAGE_BASE}:${IMAGE_TAG}"
            /kaniko/executor \
              --context "$WORKSPACE" \
              --dockerfile "$WORKSPACE/Dockerfile" \
              --destination "${IMAGE_BASE}:${IMAGE_TAG}" \
              --build-arg ENV="${DEPLOY_ENV}" \
              --build-arg FRONTEND_DIGEST="$GIT_COMMIT"
          '''
        }
      }
    }
  }
}
