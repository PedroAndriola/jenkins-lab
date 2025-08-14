pipeline {
  agent any
  environment {
    DD_SITE = 'datadoghq.com'   // troque se for .eu, us3, us5, ap1
    SERVICE = 'storefront'
    ENV     = 'prod'
  }
  stages {
    stage('Checkout'){ steps { checkout scm } }
    stage('Build'){ steps { sh 'echo build ok' } }

    stage('Deploy to prod') {
      steps {
        // ⬇️ necessário para o started_at do DORA
        script { env.DEPLOY_START = sh(script: 'date +%s', returnStdout: true).trim() }
        echo 'deploy executado'   // seu deploy real aqui
      }
      post {
        success {
          withCredentials([string(credentialsId: 'DD_API_KEY', variable: 'DD_API_KEY')]) {
            sh '''
              set -e
              REPO_URL=${GIT_URL:-$(git config --get remote.origin.url || echo "")}
              START=${DEPLOY_START}
              FINISH=$(date +%s)

              # 👉 evento DORA oficial: DEPLOYMENT (success)
              code=$(curl -sS -o /tmp/dd.out -w "%{http_code}" \
                -X POST "https://api.${DD_SITE}/api/v2/dora/deployment" \
                -H "DD-API-KEY: $DD_API_KEY" -H "Content-Type: application/json" \
                -d "{
                  \\"data\\":{\\"attributes\\":{
                    \\"service\\":\\"${SERVICE}\\",
                    \\"env\\":\\"${ENV}\\",
                    \\"version\\":\\"${GIT_COMMIT:0:7}\\",
                    \\"started_at\\": ${START},
                    \\"finished_at\\": ${FINISH},
                    \\"git\\":{\\"commit_sha\\":\\"${GIT_COMMIT}\\", \\"repository_url\\": \\"${REPO_URL}\\"}
                  }}
                }")
              echo "DORA deploy status=$code"
              test "$code" -ge 200 -a "$code" -lt 300
            '''
          }
        }
        failure {
          withCredentials([string(credentialsId: 'DD_API_KEY', variable: 'DD_API_KEY')]) {
            sh '''
              set -e
              REPO_URL=${GIT_URL:-$(git config --get remote.origin.url || echo "")}
              START=${DEPLOY_START:-$(date +%s)}
              FINISH=$(date +%s)

              # 👉 evento DORA oficial: FAILURE (mudança com falha)
              code=$(curl -sS -o /tmp/dd.out -w "%{http_code}" \
                -X POST "https://api.${DD_SITE}/api/v2/dora/failure" \
                -H "DD-API-KEY: $DD_API_KEY" -H "Content-Type: application/json" \
                -d "{
                  \\"data\\":{\\"attributes\\":{
                    \\"name\\": \\"deploy failed\\",
                    \\"services\\":[\\"${SERVICE}\\"],
                    \\"env\\": \\"${ENV}\\",
                    \\"severity\\": \\"high\\",
                    \\"started_at\\": ${START},
                    \\"finished_at\\": ${FINISH},
                    \\"git\\":{\\"commit_sha\\":\\"${GIT_COMMIT}\\", \\"repository_url\\": \\"${REPO_URL}\\"}
                  }}
                }")
              echo "DORA failure status=$code"
              test "$code" -ge 200 -a "$code" -lt 300
            '''
          }
        }
      }
    }
  }
}
