pipeline {
    agent any

    environment {
        // SonarQube Scanner tool name (defined in Global Tool Configuration)
        SCANNER_HOME = tool 'sonar-scanner'

        // Nexus settings
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
            steps { sh 'ls -R' }
        }

        // ========== SECURITY SCAN ==========
        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        // ========== SONARQUBE ==========
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh """
                            $SCANNER_HOME/bin/sonar-scanner \
                              -Dsonar.projectKey=doctor-website \
                              -Dsonar.projectName=doctor-website \
                              -Dsonar.sources=. \
                              -Dsonar.sourceEncoding=UTF-8 \
                              -Dsonar.login=$SONAR_TOKEN
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    def qg = waitForQualityGate abortPipeline: false
                    echo "Quality Gate: ${qg.status}"
                }
            }
        }

        // ========== PACKAGE ARTIFACT ==========
        stage('Package Artifact') {
            steps {
                script {
                    sh """
                        rm -f ${APP_NAME}-${BUILD_NUMBER}.zip
                        zip -r ${APP_NAME}-${BUILD_NUMBER}.zip ./*
                    """
                    archiveArtifacts artifacts: "${APP_NAME}-${BUILD_NUMBER}.zip", fingerprint: true
                }
            }
        }

        // ========== UPLOAD TO NEXUS ==========
        stage('Publish to Nexus') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'nexus-cred',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )]) {

                        sh """
                            curl -v -u "$NEXUS_USER:$NEXUS_PASS" \
                              --upload-file ${APP_NAME}-${BUILD_NUMBER}.zip \
                              "${NEXUS_URL}/repository/${NEXUS_REPO}/${APP_NAME}/${BUILD_NUMBER}/${APP_NAME}-${BUILD_NUMBER}.zip"
                        """
                    }
                }
            }
        }
        
        // =============== DEPLOY TO AMAZON EKS ===============
        stage('Deploy to EKS') {
            steps {
                echo "Deploying to Amazon EKS..."

                // kubeconfig stored as a Jenkins "Secret file" credential
                withCredentials([file(credentialsId: 'k8-cred', variable: 'KUBECONFIG')]) {

                    sh """
                        # Show cluster info
                        kubectl --kubeconfig=$KUBECONFIG cluster-info

                        # Apply deployment & service YAML
                        kubectl --kubeconfig=$KUBECONFIG apply -f deployment-service.yaml -n webapps

                        # Optional: update ENV to point to Nexus artifact URL
                        kubectl --kubeconfig=$KUBECONFIG set env deployment/doctor-website \
                            ARTIFACT_URL=${NEXUS_URL}/repository/${NEXUS_REPO}/${APP_NAME}/${BUILD_NUMBER}/${APP_NAME}-${BUILD_NUMBER}.zip \
                            -n webapps
                    """
                }
            }
        }

        stage('Verify Deployment on EKS') {
            steps {
                withCredentials([file(credentialsId: 'k8-cred', variable: 'KUBECONFIG')]) {
                    sh """
                        echo "Checking pods..."
                        kubectl --kubeconfig=$KUBECONFIG get pods -n webapps

                        echo "Checking services..."
                        kubectl --kubeconfig=$KUBECONFIG get svc -n webapps
                    """
                }
            }
        }
    }

    // ========== EMAIL NOTIFICATION ==========
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
                        <h3 style="color: white;">Status: ${pipelineStatus.toUpperCase()}</h3>
                    </div>
                    <p>Check the <a href="${BUILD_URL}">Console Output</a>.</p>
                </div>
                </body>
                </html>
                """

                emailext(
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus}",
                    body: body,
                    to: 'sanjaysadineni7209@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-fs-report.html'
                )
            }
        }
    }
}

