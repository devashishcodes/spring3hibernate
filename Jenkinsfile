pipeline {
    agent any

    tools {
        maven 'Maven3'   // Name configured in Manage Jenkins -> Tools
        jdk 'JDK11'      // Name configured in Manage Jenkins -> Tools
    }

    parameters {
        booleanParam(name: 'SKIP_UNIT_TESTS', defaultValue: false, description: 'Skip Code Stability (Unit Test) stage')
        booleanParam(name: 'SKIP_SONAR',      defaultValue: false, description: 'Skip Code Quality Analysis (SonarQube) stage')
        booleanParam(name: 'SKIP_COVERAGE',   defaultValue: false, description: 'Skip Code Coverage Analysis stage')
        booleanParam(name: 'SKIP_PUBLISH',    defaultValue: false, description: 'Skip Publish Artifacts stage entirely')
    }

    environment {
        SONARQUBE_ENV     = 'MySonarQube'                 // Name of SonarQube server config in Jenkins
        SLACK_CHANNEL     = '#ci-cd-notifications'
        EMAIL_RECIPIENTS  = 'team@example.com'
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '15'))
        disableConcurrentBuilds()
    }

    stages {

        stage('Code Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/devashishcodes/spring3hibernate.git'
            }
        }

        stage('Build & Analysis') {
            parallel {

                stage('Code Stability') {
                    when { expression { return !params.SKIP_UNIT_TESTS } }
                    steps {
                        sh 'mvn clean test'
                    }
                    post {
                        always {
                            junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                        }
                    }
                }

                stage('Code Quality Analysis') {
                    when { expression { return !params.SKIP_SONAR } }
                    steps {
                        withSonarQubeEnv("${SONARQUBE_ENV}") {
                            sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar'
                        }
                    }
                }

                stage('Code Coverage Analysis') {
                    when { expression { return !params.SKIP_COVERAGE } }
                    steps {
                        sh 'mvn clean verify'   // triggers jacoco:report bound in pom.xml
                    }
                    post {
                        always {
                            jacoco execPattern: '**/target/jacoco.exec'
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            when { expression { return !params.SKIP_SONAR } }
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Generate Report') {
            steps {
                sh 'mvn site'
                publishHTML(target: [
                    reportDir             : 'target/site',
                    reportFiles           : 'index.html',
                    reportName            : 'Code Quality & Coverage Report',
                    keepAll               : true,
                    alwaysLinkToLastBuild : true,
                    allowMissing          : true
                ])
            }
        }

        stage('Approval for Publish') {
            when { expression { return !params.SKIP_PUBLISH } }
            steps {
                script {
                    def decision = input(
                        id: 'PublishApproval',
                        message: 'Approve publishing artifacts to the repository?',
                        submitter: 'admin,release-managers',
                        parameters: [
                            choice(name: 'DECISION', choices: ['Approve', 'Reject'], description: 'Approve or Reject the publish')
                        ]
                    )
                    env.PUBLISH_DECISION = decision
                }
            }
        }

        stage('Publish Artifacts') {
            when {
                allOf {
                    expression { return !params.SKIP_PUBLISH }
                    expression { return env.PUBLISH_DECISION == 'Approve' }
                }
            }
            steps {
                sh 'mvn clean package -DskipTests'
                archiveArtifacts artifacts: '**/target/*.war', fingerprint: true
                // Uncomment once distributionManagement in pom.xml points to your Nexus/Artifactory:
                // sh 'mvn deploy -DskipTests'
            }
        }
    }

    post {
        always {
            script {
                def status  = currentBuild.currentResult
                def color   = status == 'SUCCESS' ? 'good' : (status == 'UNSTABLE' ? 'warning' : 'danger')
                def decision = env.PUBLISH_DECISION ?: 'N/A'

                slackSend(
                    channel: "${SLACK_CHANNEL}",
                    color: color,
                    message: "Job *${env.JOB_NAME}* #${env.BUILD_NUMBER} -> *${status}*\nPublish decision: ${decision}\n${env.BUILD_URL}"
                )

                emailext(
                    to: "${EMAIL_RECIPIENTS}",
                    subject: "Jenkins Build ${status}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                        Build: ${env.JOB_NAME} #${env.BUILD_NUMBER}
                        Status: ${status}
                        Publish Decision: ${decision}
                        Details: ${env.BUILD_URL}
                    """,
                    attachLog: true
                )
            }
        }
    }
}
