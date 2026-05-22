pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.9-eclipse-temurin-17
    command: ['sleep', '99d']
  - name: docker
    image: docker:24-dind
    securityContext:
      privileged: true
  - name: sonar-scanner
    image: sonarsource/sonar-scanner-cli:latest
    command: ['sleep', '99d']
  - name: trivy
    image: aquasec/trivy:latest
    command: ['sleep', '99d']
"""
        }
    }

    // ─── environment ────────────────────────────────────────────────
    environment {
        APP_NAME        = 'my-app'
        REGISTRY        = 'your-account.dkr.ecr.us-east-1.amazonaws.com'
        IMAGE_TAG       = "${env.GIT_COMMIT[0..7]}-${env.BUILD_NUMBER}"
        IMAGE_FULL      = "${REGISTRY}/${APP_NAME}:${IMAGE_TAG}"
        SONAR_HOST      = 'http://sonarqube:9000'
        SONAR_TOKEN     = credentials('sonarqube-token')
        AWS_CREDS       = credentials('aws-ecr-creds')
        KUBECONFIG_STG  = credentials('kubeconfig-staging')
        KUBECONFIG_PROD = credentials('kubeconfig-production')
        SLACK_CHANNEL   = '#ci-alerts'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
        disableConcurrentBuilds()
        skipStagesAfterUnstable()
        ansiColor('xterm')
    }

    triggers {
        // auto-trigger on every push via webhook
        githubPush()
        // also poll as a fallback (every 5 min)
        pollSCM('H/5 * * * *')
    }

    // ─── stages ─────────────────────────────────────────────────────
    stages {

        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${env.BRANCH_NAME}"]],
                    extensions: [
                        [$class: 'CleanBeforeCheckout'],
                        [$class: 'CloneOption', depth: 1, shallow: true]
                    ],
                    userRemoteConfigs: [[
                        credentialsId: 'github-ssh-key',
                        url: 'git@github.com:your-org/my-app.git'
                    ]]
                ])
                script {
                    env.GIT_COMMIT_MSG = sh(
                        returnStdout: true,
                        script: 'git log -1 --pretty=%B'
                    ).trim()
                }
            }
        }

        stage('Build') {
            steps {
                container('maven') {
                    sh '''
                        mvn clean package \
                            -DskipTests \
                            -Dmaven.repo.local=.m2/repository \
                            --no-transfer-progress
                    '''
                }
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }

        stage('Unit Tests') {
            steps {
                container('maven') {
                    sh 'mvn test --no-transfer-progress'
                }
            }
            post {
                always {
                    junit 'target/surefire-reports/**/*.xml'
                    jacoco(
                        execPattern: 'target/jacoco.exec',
                        classPattern: 'target/classes',
                        sourcePattern: 'src/main/java',
                        minimumInstructionCoverage: '80'
                    )
                }
            }
        }

        stage('Parallel Quality Gates') {
            parallel {

                stage('SonarQube Analysis') {
                    steps {
                        container('sonar-scanner') {
                            withSonarQubeEnv('SonarQube') {
                                sh """
                                    sonar-scanner \
                                      -Dsonar.projectKey=${APP_NAME} \
                                      -Dsonar.sources=src/main \
                                      -Dsonar.tests=src/test \
                                      -Dsonar.java.binaries=target/classes \
                                      -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                                      -Dsonar.host.url=${SONAR_HOST} \
                                      -Dsonar.login=${SONAR_TOKEN}
                                """
                            }
                        }
                        timeout(time: 5, unit: 'MINUTES') {
                            waitForQualityGate abortPipeline: true
                        }
                    }
                }

                stage('Integration Tests') {
                    steps {
                        container('maven') {
                            sh '''
                                docker-compose -f docker-compose.test.yml up -d
                                mvn verify -Pfailsafe --no-transfer-progress
                                docker-compose -f docker-compose.test.yml down -v
                            '''
                        }
                    }
                    post {
                        always {
                            junit 'target/failsafe-reports/**/*.xml'
                        }
                    }
                }

                stage('Security Scan — OWASP') {
                    steps {
                        container('maven') {
                            sh '''
                                mvn org.owasp:dependency-check-maven:check \
                                    -DfailBuildOnCVSS=7 \
                                    --no-transfer-progress
                            '''
                        }
                    }
                    post {
                        always {
                            dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
                        }
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                container('docker') {
                    sh """
                        docker build \
                          --build-arg BUILD_DATE=\$(date -u +%Y-%m-%dT%H:%M:%SZ) \
                          --build-arg VCS_REF=${env.GIT_COMMIT} \
                          --build-arg VERSION=${IMAGE_TAG} \
                          --label "git.commit=${env.GIT_COMMIT}" \
                          --label "build.number=${env.BUILD_NUMBER}" \
                          -t ${IMAGE_FULL} \
                          -t ${REGISTRY}/${APP_NAME}:latest \
                          .
                    """
                }
            }
        }

        stage('Container Security Scan — Trivy') {
            steps {
                container('trivy') {
                    sh """
                        trivy image \
                          --severity HIGH,CRITICAL \
                          --exit-code 1 \
                          --no-progress \
                          --format json \
                          -o trivy-report.json \
                          ${IMAGE_FULL}
                    """
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true
                }
            }
        }

        stage('Push to Registry') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(
                        credentialsId: 'aws-ecr-creds',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )]) {
                        sh """
                            aws ecr get-login-password --region us-east-1 \
                              | docker login --username AWS --password-stdin ${REGISTRY}
                            docker push ${IMAGE_FULL}
                            docker push ${REGISTRY}/${APP_NAME}:latest
                        """
                    }
                }
            }
        }

        stage('Deploy → Staging') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'main'
                }
            }
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-staging']) {
                    sh """
                        helm upgrade --install ${APP_NAME} ./helm/${APP_NAME} \
                          --namespace staging \
                          --set image.repository=${REGISTRY}/${APP_NAME} \
                          --set image.tag=${IMAGE_TAG} \
                          --set replicaCount=2 \
                          --wait \
                          --timeout 5m \
                          --atomic
                    """
                }
            }
            post {
                success {
                    // Run smoke tests against staging
                    sh "curl -f https://staging.myapp.com/health || exit 1"
                }
            }
        }

        stage('Approval Gate') {
            when { branch 'main' }
            steps {
                script {
                    def approver = input(
                        message: "Deploy ${IMAGE_TAG} to PRODUCTION?",
                        ok: 'Approve',
                        submitter: 'lead-engineers,devops-team',
                        parameters: [
                            choice(
                                name: 'DEPLOY_STRATEGY',
                                choices: ['rolling', 'blue-green', 'canary'],
                                description: 'Deployment strategy'
                            ),
                            string(
                                name: 'CANARY_WEIGHT',
                                defaultValue: '10',
                                description: '% traffic to canary (ignored unless canary strategy)'
                            )
                        ]
                    )
                    env.DEPLOY_STRATEGY = approver['DEPLOY_STRATEGY']
                    env.CANARY_WEIGHT   = approver['CANARY_WEIGHT']
                }
            }
        }

        stage('Deploy → Production') {
            when { branch 'main' }
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-production']) {
                    sh """
                        helm upgrade --install ${APP_NAME} ./helm/${APP_NAME} \
                          --namespace production \
                          --set image.repository=${REGISTRY}/${APP_NAME} \
                          --set image.tag=${IMAGE_TAG} \
                          --set deploymentStrategy=${env.DEPLOY_STRATEGY} \
                          --set canary.weight=${env.CANARY_WEIGHT} \
                          --wait \
                          --timeout 10m \
                          --atomic
                    """
                }
            }
        }
    }

    // ─── post actions ─────────────────────────────────────────────
    post {
        always {
            cleanWs()
        }
        success {
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'good',
                message: """
✅ *${APP_NAME}* `${IMAGE_TAG}` deployed successfully
Branch: `${env.BRANCH_NAME}` | Build: <${env.BUILD_URL}|#${env.BUILD_NUMBER}>
Commit: _${env.GIT_COMMIT_MSG}_
"""
            )
        }
        failure {
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'danger',
                message: """
❌ *${APP_NAME}* pipeline FAILED at stage `${env.STAGE_NAME}`
Branch: `${env.BRANCH_NAME}` | Build: <${env.BUILD_URL}|#${env.BUILD_NUMBER}>
"""
            )
            emailext(
                subject: "[FAILED] ${APP_NAME} #${env.BUILD_NUMBER}",
                body: '${JELLY_SCRIPT,template="html"}',
                to: 'devops@yourcompany.com',
                attachLog: true
            )
        }
        unstable {
            slackSend(channel: env.SLACK_CHANNEL, color: 'warning',
                message: "⚠️ *${APP_NAME}* build UNSTABLE — check test results")
        }
    }
}
