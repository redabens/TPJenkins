pipeline {
    agent any

    tools {
        jdk 'JDK11'
        gradle 'Gradle'
    }

    environment {
        SONARQUBE_ENV = 'SonarQube'
        MAVEN_REPO_URL = 'https://mymavenrepo.com'
        EMAIL_RECIPIENTS = 'mm_bensemane@esi.dz'
        SLACK_CHANNEL = '#social'
    }

    stages {

        /* ===================== TEST ===================== */
        stage('Test') {
            steps {
                echo 'Running unit tests and generating Cucumber reports...'
                script {
                    if (isUnix()) {
                        sh './gradlew clean test cucumber jacocoTestReport'
                    } else {
                        bat 'gradlew.bat clean test cucumber jacocoTestReport'
                    }
                }
            }
            post {
                always {
                    // Archive unit test results
                    junit 'build/test-results/test/*.xml'
                    archiveArtifacts artifacts: 'build/reports/tests/**', fingerprint: true
                    // Archive JaCoCo coverage reports
                    archiveArtifacts artifacts: 'build/reports/jacoco/**', fingerprint: true
                    // Archive Cucumber reports
                    archiveArtifacts artifacts: 'build/reports/cucumber/**', fingerprint: true
                    // Publish HTML reports
                    publishHTML(target: [
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'build/reports/jacoco/test/html',
                        reportFiles: 'index.html',
                        reportName: 'JaCoCo Coverage Report'
                    ])
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
                timeout(time: 2, unit: 'MINUTES') {
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
                        sh './gradlew build javadoc'
                    } else {
                        bat 'gradlew.bat build javadoc'
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
                            sh "./gradlew publish -PmavenUser=$MAVEN_USER -PmavenPassword=$MAVEN_PASS"
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
            mail to: "${EMAIL_RECIPIENTS}",
                 subject: "SUCCESS - Jenkins Pipeline",
                 body: "Build and deployment completed successfully."

            slackSend channel: "${SLACK_CHANNEL}",
                      message: "SUCCESS: API deployed successfully."
        }

        failure {
            echo 'Pipeline failed'
            mail to: "${EMAIL_RECIPIENTS}",
                 subject: "FAILURE - Jenkins Pipeline",
                 body: "Pipeline failed. Please check Jenkins logs."

            slackSend channel: "${SLACK_CHANNEL}",
                      message: "FAILURE: Jenkins pipeline failed."
        }
    }
}
