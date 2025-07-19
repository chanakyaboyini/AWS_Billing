pipeline {
  agent any

  environment {
    AWS_REGION      = 'us-east-1'
    AWS_CRED_ID     = 'jenkins-aws-start-stop'
  }

  stages {
    stage('Compute Date Range') {
      steps {
        script {
          env.START_DATE = sh(script: "date +%Y-%m-01", returnStdout: true).trim()
          env.END_DATE   = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
          echo "Reporting from ${START_DATE} to ${END_DATE}"
        }
      }
    }

    stage('Fetch AWS Cost & Usage') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: AWS_CRED_ID,
          accessKeyVariable: 'AWS_ACCESS_KEY_ID',
          secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
        ]]) {
          script {
            def report = sh(
              script: """
                aws ce get-cost-and-usage \\
                  --time-period Start=${START_DATE},End=${END_DATE} \\
                  --granularity MONTHLY \\
                  --metrics UNBLENDED_COST \\
                  --metrics USAGE_QUANTITY \\
                  --group-by Type=DIMENSION,Key=SERVICE \\
                  --region ${AWS_REGION} \\
                  --output table
              """,
              returnStdout: true
            ).trim()

            writeFile file: 'aws-cost-usage.txt', text: report
            echo "Saved cost & usage to aws-cost-usage.txt"
          }
        }
      }
    }

    stage('Slack Notification') {
      steps {
        withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_WEBHOOK')]) {
          script {
            def message = readFile('aws-cost-usage.txt').replaceAll('\\n', '\n> ') // blockquote format
            def payload = """{
              "text": "*📊 AWS Cost & Usage Report* (${START_DATE} → ${END_DATE})\\n> ${message}"
            }"""
            writeFile file: 'slack-message.json', text: payload

            sh """
              curl -X POST -H 'Content-type: application/json' \\
                   --data @slack-message.json \\
                   "$SLACK_WEBHOOK"
            """
            echo "Posted billing report to Slack"
          }
        }
      }
    }
  }

  post {
    always {
      echo "Pipeline finished. Slack notification sent."
    }
  }
}