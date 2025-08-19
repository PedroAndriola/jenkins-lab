pipeline {
  agent any
  options { timestamps() }

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
        // marca o started_at do deploy
        script { env.DEPLOY_START = sh(script: 'date +%s', returnStdout: true).trim() }
        sh 'echo deploying... && sleep 2 && echo deploy done'   // seu deploy real aqui
        // para testar falha, descomente:
        // sh 'exit 1'
      }
      post {
        success {
          withCredentials([string(credentialsId: 'DD_API_KEY', variable: 'DD_API_KEY')]) {
            sh '''
              set -e
              RAW_URL=${GIT_URL:-$(git config --get remote.origin.url || echo "")}
              # normaliza repo para HTTPS e remove .git
              REPO_URL=$(echo "$RAW_URL" | sed -E 's#git@([^:]+):#https://\\1/#; s#\\.git$##')

              START=${DEPLOY_START}
              FINISH=$(date +%s)
              # SHA curto POSIX (sem bash)
              SHORT_SHA=$(printf "%s" "${GIT_COMMIT}" | cut -c1-7)

              code=$(curl -sS -o /tmp/dd.out -w "%{http_code}" \
                -X POST "https://api.${DD_SITE}/api/v2/dora/deployment" \
                -H "DD-API-KEY: $DD_API_KEY" -H "Content-Type: application/json" \
                -d "{
                  \\"data\\":{\\"attributes\\":{
                    \\"service\\":\\"${SERVICE}\\",
                    \\"env\\":\\"${ENV}\\",
                    \\"version\\":\\"${SHORT_SHA}\\",
                    \\"started_at\\": ${START},
                    \\"finished_at\\": ${FINISH},
                    \\"git\\":{\\"commit_sha\\":\\"${GIT_COMMIT}\\", \\"repository_url\\": \\"${REPO_URL}\\"}
                  }}
                }")

              echo "DORA deploy status=$code"
              if [ "$code" -ge 200 ] && [ "$code" -lt 300 ]; then
                echo "Deploy event OK"
              else
                echo "Datadog API error (deployment):"; cat /tmp/dd.out || true
                exit 1
              fi
            '''
          }
        }
        failure {
          withCredentials([string(credentialsId: 'DD_API_KEY', variable: 'DD_API_KEY')]) {
            sh '''
              set -e
              RAW_URL=${GIT_URL:-$(git config --get remote.origin.url || echo "")}
              REPO_URL=$(echo "$RAW_URL" | sed -E 's#git@([^:]+):#https://\\1/#; s#\\.git$##')

              START=${DEPLOY_START:-$(date +%s)}
              FINISH=$(date +%s)
              SHORT_SHA=$(printf "%s" "${GIT_COMMIT}" | cut -c1-7)

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
              if [ "$code" -ge 200 ] && [ "$code" -lt 300 ]; then
                echo "Failure event OK"
              else
                echo "Datadog API error (failure):"; cat /tmp/dd.out || true
                exit 1
              fi
            '''
          }
        }
      }
    }
  }
}
