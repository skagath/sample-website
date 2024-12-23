failure {
    script {
        try {
            // Fetch the last 50 lines of the pipeline logs
            def last50Logs = currentBuild.rawBuild.getLog(50).join('\n')
            
            // Truncate if the Slack message exceeds the limit
            def maxSlackMessageLength = 3900
            if (last50Logs.length() > maxSlackMessageLength) {
                last50Logs = last50Logs.take(maxSlackMessageLength) + "\n... (truncated)"
            }

            // Send logs to Slack
            slackSend channel: "${SLACK_CHANNEL_NAME}",
                      color: 'danger',
                      message: "❌ Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER} failed!\nLast 50 Lines of Logs:\n${last50Logs}",
                      tokenCredentialId: 'slack-tocken'
        } catch (Exception e) {
            echo "Failed to fetch or process logs: ${e.getMessage()}"

            // Fallback Slack notification
            slackSend channel: "${SLACK_CHANNEL_NAME}",
                      color: 'danger',
                      message: "❌ Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER} failed! Error retrieving logs: ${e.getMessage()}. Check logs: ${env.JENKINS_URL}",
                      tokenCredentialId: 'slack-tocken'
        }
    }
}
