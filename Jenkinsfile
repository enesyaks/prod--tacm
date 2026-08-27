// ITACM teslimat hatti.
//
// Akis:  feat/*  ->  stage  ->  dev  ->  main
//
// Kod dallar arasinda tasinir, ama YAYINA CIKAN SEY her zaman
// k8s/overlays/<ortam>/kustomization.yaml icindeki newTag satiridir.
// Bu hat o satiri da yazar, dolayisiyla elle yapilan tek is onay vermek.

def podYaml = '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:v1.23.2-debug
      command: ["/busybox/cat"]
      tty: true
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
    - name: git
      image: alpine/git:2.45.2
      command: ["cat"]
      tty: true
  volumes:
    - name: docker-config
      secret:
        secretName: harbor-docker-config
        items:
          - key: .dockerconfigjson
            path: config.json
'''

pipeline {
  // Ust seviyede agent yok: onay bekleyen bir stage hicbir pod tutmasin.
  agent none

  options {
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  environment {
    IMAGE   = 'harbor.itacm.site/itacm/itacm'
    GIT_URL_HTTPS = 'https://github.com/enesyaks/prod--tacm.git'
  }

  stages {

    // ---------------------------------------------------------------
    stage('Build') {
      // Jenkins'in kendi attigi deploy commit'leri yeni bir build
      // tetiklemesin. Dongu buradan kiriliyor.
      // beforeAgent: kosul saglanmiyorsa agent pod'u hic acilmasin.
      when {
        beforeAgent true
        not { changelog '.*\\[skip ci\\].*' }
      }
      agent { kubernetes { yaml podYaml } }
      steps {
        container('kaniko') {
          script {
            // Ortam dallari Harbor'a yazar; ozellik dallari sadece derlenir.
            def push = env.BRANCH_NAME in ['stage', 'dev', 'main']
            def dest = push ? "--destination ${IMAGE}:${env.GIT_COMMIT}" : "--no-push"
            sh """
              /kaniko/executor \\
                --context "\$WORKSPACE" \\
                --dockerfile "\$WORKSPACE/Dockerfile" \\
                --build-arg FRONTEND_DIGEST="\$GIT_COMMIT" \\
                ${dest}
            """
          }
        }
      }
    }

    // ---------------------------------------------------------------
    stage('Yayina al') {
      // stage ve dev dallari kendi ortamlarinin etiketini gunceller.
      // main bunu YAPMAZ: prod'a cikis ayri bir onaydan gecer.
      when {
        beforeAgent true
        allOf {
          anyOf { branch 'stage'; branch 'dev' }
          not { changelog '.*\\[skip ci\\].*' }
        }
      }
      agent { kubernetes { yaml podYaml } }
      steps {
        container('git') {
          withCredentials([usernamePassword(credentialsId: 'github',
                                            usernameVariable: 'GIT_USER',
                                            passwordVariable: 'GIT_TOKEN')]) {
            sh '''
              set -eu
              ENV_NAME="$BRANCH_NAME"
              FILE="k8s/overlays/${ENV_NAME}/kustomization.yaml"

              git config --global user.email "jenkins@itacm.site"
              git config --global user.name  "jenkins"
              git config --global --add safe.directory "$WORKSPACE"

              # Overlay'ler main'de durur, dolayisiyla main'i ayrica cekiyoruz.
              rm -rf /tmp/gitops
              git clone --depth 1 --branch main \
                "https://${GIT_USER}:${GIT_TOKEN}@github.com/enesyaks/prod--tacm.git" /tmp/gitops
              cd /tmp/gitops

              sed -i "s|newTag: .*|newTag: ${GIT_COMMIT}|" "$FILE"

              # Etiket zaten dogruysa commit atma. Bu, ayni surumun tekrar
              # tetiklenmesinde bos commit uretmeyi ve dongüyü onler.
              if git diff --quiet -- "$FILE"; then
                echo "[deploy] ${ENV_NAME} zaten ${GIT_COMMIT} calistiriyor"
                exit 0
              fi

              git commit -am "deploy(${ENV_NAME}): ${GIT_COMMIT} [skip ci]"
              git push origin main
            '''
          }
        }
      }
    }

    // ---------------------------------------------------------------
    stage('Onay: dev') {
      // beforeInput: Jenkins varsayilan olarak input'u when'den ONCE
      // degerlendirir, yani kosul saglanmayan bir dalda bile onay sorar.
      // Bu satir sirayi tersine cevirir.
      when {
        beforeAgent true
        beforeInput true
        branch 'stage'
      }
      // input bir DIREKTIF, adim degil: onay gelene kadar pod acilmaz.
      input {
        message 'stage dogrulandi mi? Kod dev ortamina tasinsin mi?'
        ok 'Tasi'
      }
      agent { kubernetes { yaml podYaml } }
      steps {
        container('git') {
          withCredentials([usernamePassword(credentialsId: 'github',
                                            usernameVariable: 'GIT_USER',
                                            passwordVariable: 'GIT_TOKEN')]) {
            sh '''
              set -eu
              git config --global user.email "jenkins@itacm.site"
              git config --global user.name  "jenkins"

              rm -rf /tmp/promote && git clone \
                "https://${GIT_USER}:${GIT_TOKEN}@github.com/enesyaks/prod--tacm.git" /tmp/promote
              cd /tmp/promote

              git checkout dev
              # --ff-only sart: SHA degismesin ki dev, stage'de test edilen
              # image'in AYNISINI calistirsin. Mumkun degilse dur ve soyle.
              git merge --ff-only "origin/stage"
              git push origin dev
            '''
          }
        }
      }
    }

    // ---------------------------------------------------------------
    stage('Onay: main') {
      // beforeInput: Jenkins varsayilan olarak input'u when'den ONCE
      // degerlendirir, yani kosul saglanmayan bir dalda bile onay sorar.
      // Bu satir sirayi tersine cevirir.
      when {
        beforeAgent true
        beforeInput true
        branch 'dev'
      }
      input {
        message 'dev dogrulandi mi? Kod main dalina tasinsin mi?'
        ok 'Tasi'
      }
      agent { kubernetes { yaml podYaml } }
      steps {
        container('git') {
          withCredentials([usernamePassword(credentialsId: 'github',
                                            usernameVariable: 'GIT_USER',
                                            passwordVariable: 'GIT_TOKEN')]) {
            sh '''
              set -eu
              git config --global user.email "jenkins@itacm.site"
              git config --global user.name  "jenkins"

              rm -rf /tmp/promote && git clone \
                "https://${GIT_USER}:${GIT_TOKEN}@github.com/enesyaks/prod--tacm.git" /tmp/promote
              cd /tmp/promote

              git checkout main
              # Burada --ff-only YOK: main deploy commit'leri yuzunden her
              # zaman dev'in onunde olur, fast-forward yapisi geregi imkansiz.
              git merge --no-edit "origin/dev"

              # prod'un etiketi dev'de dogrulanan surumle AYNI olacak.
              sed -i "s|newTag: .*|newTag: ${GIT_COMMIT}|" k8s/overlays/prod/kustomization.yaml
              git commit -am "deploy(prod): ${GIT_COMMIT} [skip ci]" || true
              git push origin main

              echo "[promote] prod etiketi yazildi. Yayina almak icin Argo CD'de Sync."
            '''
          }
        }
      }
    }
  }
}
