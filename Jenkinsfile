pipeline {
    agent any

    tools {
        jdk 'JDK11'
        gradle 'Gradle'
    }

    environment {
        SONARQUBE_ENV = 'SonarQube'
        EMAIL_RECIPIENTS = 'mm_bensemane@esi.dz'
        SLACK_CHANNEL = '#social'
        // Variables pour Gradle (optionnel si vous voulez utiliser Slack/Email depuis Gradle)
        SLACK_WEBHOOK_URL = credentials('slack-webhook-url')  // Si configuré
        GMAIL_USER = 'mm_bensemane@esi.dz'
        GMAIL_APP_PASSWORD = credentials('gmail-app-password')  // Si configuré
    }

    stages {

        /* ===================== TEST ===================== */
        stage('Test') {
            steps {
                echo 'Running unit tests and generating Cucumber reports...'
                script {
                    if (isUnix()) {
                        sh './gradlew clean test generateCucumberReports jacocoTestReport'
                    } else {
                        bat 'gradlew.bat clean test generateCucumberReports jacocoTestReport'
                    }
                }
            }
            post {
                always {
                    // Archive unit test results
                    junit 'build/test-results/test/*.xml'

                    // Archive JaCoCo coverage reports
                    archiveArtifacts artifacts: 'build/reports/jacoco/**', fingerprint: true

                    // Archive Cucumber reports
                    archiveArtifacts artifacts: 'build/reports/cucumber/**', fingerprint: true

                    // Publish JaCoCo coverage report
                    jacoco(
                        execPattern: 'build/jacoco/*.exec',
                        classPattern: 'build/classes/java/main',
                        sourcePattern: 'src/main/java'
                    )

                    // Publish HTML reports
                    publishHTML(target: [
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'build/reports/jacoco/test/html',
                        reportFiles: 'index.html',
                        reportName: 'JaCoCo Coverage Report'
                    ])

                    // Publish Cucumber reports
                    publishHTML(target: [
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'build/reports/cucumber/cucumber-html-reports',
                        reportFiles: 'overview-features.html',
                        reportName: 'Cucumber Report'
                    ])
                }
            }
        }

        /* ================= CODE ANALYSIS ================= */
        stage('Code Analysis - SonarQube') {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    script {
                        if (isUnix()) {
                            sh './gradlew sonar'
                        } else {
                            bat 'gradlew.bat sonar'
                        }
                    }
                }
            }
        }

        /* ================= QUALITY GATE ================= */
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
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
                    archiveArtifacts artifacts: 'build/libs/*.jar', fingerprint: true
                    archiveArtifacts artifacts: 'build/docs/javadoc/**', fingerprint: true
                }
            }
        }

        /* ===================== DEPLOY ===================== */
        stage('Deploy') {
            steps {
                echo 'Deploying artifact to Maven repository...'
                withCredentials([usernamePassword(
                    credentialsId: 'maven-repo-creds',
                    usernameVariable: 'MAVEN_USER',
                    passwordVariable: 'MAVEN_PASS'
                )]) {
                    script {
                        if (isUnix()) {
                            sh "./gradlew publish -PmavenUser=\$MAVEN_USER -PmavenPassword=\$MAVEN_PASS"
                        } else {
                            bat "gradlew.bat publish -PmavenUser=%MAVEN_USER% -PmavenPassword=%MAVEN_PASS%"
                        }
                    }
                }
            }
        }
    }

    /* ================= NOTIFICATIONS ================= */
    post {
        success {
            echo 'Pipeline succeeded'
            emailext (
                to: "${EMAIL_RECIPIENTS}",
                subject: "SUCCESS - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    <h2>Build Successful!</h2>
                    <p><b>Project:</b> ${env.JOB_NAME}</p>
                    <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                    <p><b>Build URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    <p>Build and deployment completed successfully.</p>
                """,
                mimeType: 'text/html'
            )

            // Slack notification (optionnel - nécessite le plugin Slack)
            // slackSend channel: "${SLACK_CHANNEL}",
            //           color: 'good',
            //           message: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER} - API deployed successfully. (<${env.BUILD_URL}|Open>)"
        }

        failure {
            echo 'Pipeline failed'
            emailext (
                to: "${EMAIL_RECIPIENTS}",
                subject: "FAILURE - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    <h2>Build Failed!</h2>
                    <p><b>Project:</b> ${env.JOB_NAME}</p>
                    <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                    <p><b>Build URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    <p><b>Console Output:</b> <a href="${env.BUILD_URL}console">${env.BUILD_URL}console</a></p>
                    <p>Pipeline failed. Please check the console output for details.</p>
                """,
                mimeType: 'text/html'
            )

            // Slack notification (optionnel)
            slackSend channel: "${SLACK_CHANNEL}",
                      color: 'danger',
                      message: "FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER} - Pipeline failed. (<${env.BUILD_URL}|Open>)"
        }
    }
}