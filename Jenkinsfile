pipeline {
  agent any
  environment {
    DD_SITE = 'datadoghq.com'
    SERVICE = 'storefront'
    ENV     = 'prod'
  }
  stages {
    stage('Checkout'){ steps { checkout scm } }
    stage('Build'){ steps { sh 'echo build ok' } }

    stage('Deploy to prod') {
      steps {
        echo 'deploy executado'     // seu deploy real aqui
      }
      post {
        success {
          withCredentials([string(credentialsId: 'DD_API_KEY', variable: 'DD_API_KEY')]) {
            sh '''
              set -e
              NOW=$(date +%s)
              SHA=${GIT_COMMIT:-$(git rev-parse --short HEAD)}
              MERGE_TS=$(git show -s --format=%ct "$GIT_COMMIT" 2>/dev/null || echo $NOW)
              LEAD_MS=$(( (NOW - MERGE_TS) * 1000 ))

              # Evento de deploy (success)
              curl -sS -X POST "https://api.${DD_SITE}/api/v2/events" \
                -H "DD-API-KEY: $DD_API_KEY" -H "Content-Type: application/json" \
                -d "{
                  \\"title\\": \\"deploy success\\",
                  \\"text\\": \\"service=${SERVICE} version=${SHA}\\",
                  \\"tags\\": [\\"service:${SERVICE}\\",\\"env:${ENV}\\",\\"version:${SHA}\\",\\"commit:${GIT_COMMIT}\\",\\"status:success\\"],
                  \\"source\\": \\"deploy\\", \\"type\\": \\"event\\", \\"timestamp\\": ${NOW}
                }" >/dev/null

              # Lead time
              curl -sS -X POST "https://api.${DD_SITE}/api/v1/series" \
                -H "DD-API-KEY: $DD_API_KEY" -H "Content-Type: application/json" \
                -d "{
                  \\"series\\":[{\\"metric\\":\\"dora.lead_time_ms\\",\\"type\\":\\"distribution\\",
                  \\"points\\":[[${NOW},${LEAD_MS}]],
                  \\"tags\\":[\\"service:${SERVICE}\\",\\"env:${ENV}\\",\\"version:${SHA}\\",\\"commit:${GIT_COMMIT}\\"]}]}" >/dev/null
            '''
          }
          
          withCredentials([string(credentialsId: 'DD_API_KEY', variable: 'DD_API_KEY')]) {
            sh '''
              NOW=$(date +%s); DUR_MS=$((15*60*1000))
              curl -sS -X POST "https://api.${DD_SITE}/api/v1/series" \
                -H "DD-API-KEY: $DD_API_KEY" -H "Content-Type: application/json" \
                -d "{
                  \\"series\\":[{\\"metric\\":\\"dora.incident.duration_ms\\",\\"type\\":\\"distribution\\",
                  \\"points\\":[[${NOW},${DUR_MS}]],
                  \\"tags\\":[\\"service:${SERVICE}\\",\\"env:${ENV}\\"]}]}"
            '''
          }
        }
        failure {
          withCredentials([string(credentialsId: 'DD_API_KEY', variable: 'DD_API_KEY')]) {
            sh '''
              NOW=$(date +%s)
              SHA=${GIT_COMMIT:-$(git rev-parse --short HEAD 2>/dev/null || echo "b${BUILD_NUMBER}")}

              curl -sS -X POST "https://api.${DD_SITE}/api/v2/events" \
                -H "DD-API-KEY: $DD_API_KEY" -H "Content-Type: application/json" \
                -d "{
                  \\"title\\": \\"deploy failed\\",
                  \\"text\\": \\"service=${SERVICE} version=${SHA}\\",
                  \\"tags\\": [\\"service:${SERVICE}\\",\\"env:${ENV}\\",\\"version:${SHA}\\",\\"commit:${GIT_COMMIT}\\",\\"status:failed\\"],
                  \\"source\\": \\"deploy\\", \\"type\\": \\"event\\", \\"timestamp\\": ${NOW}
                }" >/dev/null
            '''
          }
        }
      }
    }
  }
}
