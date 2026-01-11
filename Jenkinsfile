pipeline {
    agent any

    tools {
        jdk 'JDK11'
        gradle 'Gradle'
    }

    environment {
        SONARQUBE_ENV = 'SonarQube'  // Nom exact de l'installation SonarQube dans Jenkins
        EMAIL_RECIPIENTS = 'mm_bensemane@esi.dz'
        SLACK_CHANNEL = '#social'
        SLACK_WEBHOOK_URL = credentials('slack-webhook-url')
        GMAIL_USER = 'mm_bensemane@esi.dz'
        GMAIL_APP_PASSWORD = credentials('gmail-app-password')
        GRADLE_OPTS = '-Djavax.net.ssl.trustStoreType=Windows-ROOT -Djavax.net.ssl.trustStore=NONE'
    }

    stages {

        /* ================= ENVIRONMENT CHECK ================= */
        stage('Environment Check') {
            steps {
                echo 'Checking Java and Gradle versions...'
                script {
                    if (isUnix()) {
                        sh 'java -version'
                        sh './gradlew --version'
                    } else {
                        bat 'java -version'
                        bat 'gradlew.bat --version'
                    }
                }
            }
        }

        /* ===================== TEST ===================== */
        stage('Test') {
            steps {
                echo 'Running unit tests and generating Cucumber reports...'
                script {
                    if (isUnix()) {
                        sh './gradlew clean test generateCucumberReports jacocoTestReport --stacktrace'
                    } else {
                        bat 'gradlew.bat clean test generateCucumberReports jacocoTestReport --stacktrace'
                    }
                }
            }
            post {
                always {
                    // Archive unit test results
                    junit allowEmptyResults: true, testResults: 'build/test-results/test/*.xml'

                    // Archive JaCoCo reports
                    script {
                        if (fileExists('build/reports/jacoco')) {
                            archiveArtifacts artifacts: 'build/reports/jacoco/**', fingerprint: true, allowEmptyArchive: true

                            // Publish JaCoCo HTML report
                            publishHTML(target: [
                                allowMissing: true,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: 'build/reports/jacoco/test/html',
                                reportFiles: 'index.html',
                                reportName: 'JaCoCo Coverage Report'
                            ])
                        }

                        // Archive Cucumber reports
                        if (fileExists('build/reports/cucumber')) {
                            archiveArtifacts artifacts: 'build/reports/cucumber/**', fingerprint: true, allowEmptyArchive: true

                            // Publish Cucumber HTML report
                            publishHTML(target: [
                                allowMissing: true,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: 'build/reports/cucumber/cucumber-html-reports',
                                reportFiles: 'overview-features.html',
                                reportName: 'Cucumber Report'
                            ])
                        }
                    }
                }
                success {
                    echo '✅ Tests passed successfully!'
                }
            }
        }

        /* ================= CODE ANALYSIS ================= */
        stage('Code Analysis - SonarQube') {
            steps {
                script {
                    try {
                        withSonarQubeEnv("${SONARQUBE_ENV}") {
                            if (isUnix()) {
                                sh './gradlew sonar'
                            } else {
                                bat 'gradlew.bat sonar'
                            }
                        }
                        env.SONAR_ANALYSIS_COMPLETED = 'true'
                    } catch (Exception e) {
                        echo "⚠️ SonarQube analysis failed: ${e.message}"
                        echo "Continuing pipeline without SonarQube..."
                        env.SONAR_ANALYSIS_COMPLETED = 'false'
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }

        /* ================= QUALITY GATE ================= */
        stage('Quality Gate') {
            when {
                expression { env.SONAR_ANALYSIS_COMPLETED == 'true' }
            }
            steps {
                script {
                    try {
                        timeout(time: 2, unit: 'MINUTES') {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                echo "⚠️ Quality Gate status: ${qg.status}"
                                currentBuild.result = 'UNSTABLE'
                            } else {
                                echo "✅ Quality Gate passed!"
                            }
                        }
                    } catch (Exception e) {
                        echo "⚠️ Quality Gate failed or timed out: ${e.message}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }

        /* ===================== BUILD ===================== */
        stage('Build') {
            steps {
                echo 'Building JAR and Javadoc...'
                script {
                    if (isUnix()) {
                        sh './gradlew build javadoc -x test'
                    } else {
                        bat 'gradlew.bat build javadoc -x test'
                    }
                }
            }
            post {
                success {
                    archiveArtifacts artifacts: 'build/libs/*.jar', fingerprint: true, allowEmptyArchive: true
                    archiveArtifacts artifacts: 'build/docs/javadoc/**', fingerprint: true, allowEmptyArchive: true
                    echo '✅ Build artifacts created successfully!'
                }
            }
        }

        /* ===================== DEPLOY ===================== */
        stage('Deploy') {
            steps {
                echo 'Deploying artifact to Maven repository...'
                script {
                    try {
                        withCredentials([usernamePassword(
                            credentialsId: 'maven-repo-creds',
                            usernameVariable: 'MAVEN_USER',
                            passwordVariable: 'MAVEN_PASS'
                        )]) {
                            if (isUnix()) {
                                sh "./gradlew publish -PmavenUser=\$MAVEN_USER -PmavenPassword=\$MAVEN_PASS"
                            } else {
                                bat "gradlew.bat publish -PmavenUser=%MAVEN_USER% -PmavenPassword=%MAVEN_PASS%"
                            }
                        }
                        echo '✅ Deployment successful!'
                    } catch (Exception e) {
                        echo "⚠️ Deployment failed: ${e.message}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
    }

    /* ================= NOTIFICATIONS ================= */
    post {
        success {
            echo '✅ Pipeline completed successfully!'
            emailext (
                to: "${EMAIL_RECIPIENTS}",
                subject: "✅ SUCCESS - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    <h2 style="color: green;">✅ Build Successful!</h2>
                    <p><b>Project:</b> ${env.JOB_NAME}</p>
                    <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                    <p><b>Build URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    <p><b>JaCoCo Coverage Report:</b> <a href="${env.BUILD_URL}JaCoCo_20Coverage_20Report/">View Coverage</a></p>
                    <p><b>Cucumber Test Report:</b> <a href="${env.BUILD_URL}Cucumber_20Report/">View Tests</a></p>
                    <hr>
                    <p>✅ All tests passed</p>
                    <p>✅ Build artifacts created</p>
                    <p>✅ Deployment completed</p>
                """,
                mimeType: 'text/html'
            )

            script {
                try {
                    slackSend(
                        channel: "${SLACK_CHANNEL}",
                        color: 'good',
                        message: "✅ SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n• Tests passed\n• Build completed\n• Deployment successful\n<${env.BUILD_URL}|View Build>",
                        tokenCredentialId: 'slack-webhook-url'
                    )
                } catch (Exception e) {
                    echo "⚠️ Slack notification failed: ${e.message}"
                }
            }
        }

        unstable {
            echo '⚠️ Pipeline completed with warnings'
            emailext (
                to: "${EMAIL_RECIPIENTS}",
                subject: "⚠️ UNSTABLE - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    <h2 style="color: orange;">⚠️ Build Unstable</h2>
                    <p><b>Project:</b> ${env.JOB_NAME}</p>
                    <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                    <p><b>Build URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    <p><b>Console Output:</b> <a href="${env.BUILD_URL}console">${env.BUILD_URL}console</a></p>
                    <hr>
                    <p>The build completed but some non-critical steps failed (SonarQube or Deployment).</p>
                    <p>Please check the console output for details.</p>
                """,
                mimeType: 'text/html'
            )

            script {
                try {
                    slackSend(
                        channel: "${SLACK_CHANNEL}",
                        color: 'warning',
                        message: "⚠️ UNSTABLE: ${env.JOB_NAME} #${env.BUILD_NUMBER}\nBuild completed with warnings.\n<${env.BUILD_URL}console|View Console>",
                        tokenCredentialId: 'slack-webhook-url'
                    )
                } catch (Exception e) {
                    echo "⚠️ Slack notification failed: ${e.message}"
                }
            }
        }

        failure {
            echo '❌ Pipeline failed'
            emailext (
                to: "${EMAIL_RECIPIENTS}",
                subject: "❌ FAILURE - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    <h2 style="color: red;">❌ Build Failed!</h2>
                    <p><b>Project:</b> ${env.JOB_NAME}</p>
                    <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                    <p><b>Build URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    <p><b>Console Output:</b> <a href="${env.BUILD_URL}console">${env.BUILD_URL}console</a></p>
                    <hr>
                    <p style="color: red;">Pipeline failed. Please check the console output for details.</p>
                """,
                mimeType: 'text/html'
            )

            script {
                try {
                    slackSend(
                        channel: "${SLACK_CHANNEL}",
                        color: 'danger',
                        message: "❌ FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER}\nPipeline failed!\n<${env.BUILD_URL}console|View Console>",
                        tokenCredentialId: 'slack-webhook-url'
                    )
                } catch (Exception e) {
                    echo "⚠️ Slack notification failed: ${e.message}"
                }
            }
        }
    }
}