pipeline {
    agent any

    environment {
        // SonarQube Scanner tool name (defined in Global Tool Configuration)
        SCANNER_HOME = tool 'sonar-scanner'

        // Nexus settings (NOT secret)
        NEXUS_URL  = 'http://54.89.128.176:8081'    
        NEXUS_REPO = 'doctor-artifacts'               
        APP_NAME   = 'doctor-website'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'git-cred2',
                    url: 'https://github.com/sanjay7209/devops-pipeline.git'
            }
        }

        stage('List Source Files') {
            steps {
                sh 'ls -R'
            }
        }

        // ===== SECURITY: FILE SYSTEM SCAN =====
        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        // ===== SONARQUBE (optional – remove if not using) =====
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                          -Dsonar.projectKey=doctor-website \
                          -Dsonar.projectName=doctor-website \
                          -Dsonar.sources=. \
                          -Dsonar.sourceEncoding=UTF-8
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        // ===== PACKAGE ARTIFACT (ZIP WEBSITE) =====
        stage('Package Artifact') {
            steps {
                script {
                    sh """
                        rm -f ${APP_NAME}-${BUILD_NUMBER}.zip
                        zip -r ${APP_NAME}-${BUILD_NUMBER}.zip ./*
                    """
                    // Archive inside Jenkins as well
                    archiveArtifacts artifacts: "${APP_NAME}-${BUILD_NUMBER}.zip", fingerprint: true
                }
            }
        }

        // ===== PUBLISH ARTIFACT TO NEXUS =====
        stage('Publish to Nexus') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'nexus-cred',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )]) {
                        // Path in Nexus: /repository/<repo>/<app-name>/<build>/artifact.zip
                        sh """
                            curl -v -u "$NEXUS_USER:$NEXUS_PASS" \
                              --upload-file ${APP_NAME}-${BUILD_NUMBER}.zip \
                              "${NEXUS_URL}/repository/${NEXUS_REPO}/${APP_NAME}/${BUILD_NUMBER}/${APP_NAME}-${BUILD_NUMBER}.zip"
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                def jobName        = env.JOB_NAME
                def buildNumber    = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor    = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

                def body = """
                <html>
                <body>
                <div style="border: 4px solid ${bannerColor}; padding:10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color:${bannerColor}; padding:10px;">
                        <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                    </div>
                    <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                </div>
                </body>
                </html>
                """

                emailext(
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'sanjaysadineni7209@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-fs-report.html'
                )
            }
        }
    }
}

