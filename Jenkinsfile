pipeline {
  agent {
    kubernetes {
      yaml '''
apiVersion: v1
kind: Pod
metadata:
  namespace: jenkins
spec:
  # Kaniko'nun Harbor'ı bulabilmesi için DNS kaydı
  hostAliases:
    - ip: "10.10.10.2"
      hostnames:
        - "harbor.itacm.site"
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:v1.23.2-debug
      command: ["/busybox/cat"]
      tty: true
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
    # Manifest güncellemesi için Git aracı
    - name: git
      image: alpine/git:v2.40.1
      command: ["/bin/sh", "-c", "cat"]
      tty: true
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

    stage('Update manifest') {
      steps {
        container('git') {
          // Jenkins'teki credential ID'si 'github' olmalı (Username with password)
          withCredentials([usernamePassword(
            credentialsId: 'github', 
            usernameVariable: 'GIT_USER', 
            passwordVariable: 'GIT_TOKEN')]) {
            sh """
              echo "Updating k8s/02-itacm.yaml with new tag: ${IMAGE_TAG}"
              
              # 1. Eski etiketi silip yeni etiketi (IMAGE_TAG) yazıyoruz
              sed -i "s|itacm/itacm:.*|itacm/itacm:${IMAGE_TAG}|" k8s/02-itacm.yaml
              
              # 2. Git kimlik ayarları (Commit'te Jenkins olarak görünecek)
              git config user.email "jenkins@itacm.site"
              git config user.name "jenkins"
              
              # 3. Değişikliği commit'le ve dinamik olarak çalışılan branch'e pushla
              git commit -am "deploy: ${IMAGE_TAG}"
              git push https://${GIT_USER}:${GIT_TOKEN}@github.com/enesyaks/prod--tacm.git HEAD:${BRANCH_NAME}
            """
          }
        }
      }
    }
  }
}
