pipeline {
  agent any
  options { timestamps() }

  environment {
    DD_SITE = 'datadoghq.com'  // troque se for .eu, us3, us5, ap1
    SERVICE = 'storefront'
    ENV     = 'prod'
  }

  stages {
    stage('Checkout'){ steps { checkout scm } }

    stage('Build'){ steps { sh 'echo build ok' } }

    stage('Deploy to prod') {
      steps {
        // marca o início do deploy (started_at)
        script { env.DEPLOY_START = sh(script: 'date +%s', returnStdout: true).trim() }
        // “deploy” de mentira só pra ter duração > 0
        sh 'echo deploying... && sleep 2 && echo deploy done'
      }
    }
  }

  post {
    // ← enviamos o evento DORA de deployment no success do pipeline
    success {
      withCredentials([string(credentialsId: 'DD_API_KEY', variable: 'DD_API_KEY')]) {
        sh '''
          set -e
          # normaliza repo p/ HTTPS e remove .git
          RAW_URL=${GIT_URL:-$(git config --get remote.origin.url || echo "")}
          REPO_URL=$(echo "$RAW_URL" | sed -E 's#git@([^:]+):#https://\\1/#; s#\\.git$##')

          START=${DEPLOY_START:-$(date +%s)}
          FINISH=$(date +%s)

          # DORA: deployment (success)
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
          if [ "$code" -ge 200 ] && [ "$code" -lt 300 ]; then
            echo "Deploy event OK"
          else
            echo "Datadog API error (deployment):"; cat /tmp/dd.out || true
            exit 1
          fi
        '''
      }
    }
  }
}
