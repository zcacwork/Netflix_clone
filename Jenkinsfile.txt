pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'nodejs'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Workspace Cleaning') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'master', url: 'https://github.com/sunilkumar0633/Netflix-Clone.git'
            }
        }

        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=DevOpsProject \
                        -Dsonar.projectKey=DevOpsProject \
                        -Dsonar.java.binaries=.
                    '''
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('OWASP DP SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'owasp-dp-check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        
        stage('TRIVY FS SCAN') {
            steps {
               sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        stage("Docker Image Build") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t $JOB_NAME:$BUILD_ID --build-arg TMDB_V3_API_KEY=647f91193510922347e7200c56bd3c83 ."
                        sh "docker tag $JOB_NAME:$BUILD_ID jacksneel/$JOB_NAME:$BUILD_ID"
                        sh "docker tag $JOB_NAME:$BUILD_ID jacksneel/$JOB_NAME:latest"
                    }
                }
            }
        }

        stage("Docker Image Pushing") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push jacksneel/$JOB_NAME:$BUILD_ID"
                        sh "docker push jacksneel/$JOB_NAME:latest"
                    }
                }
            }
        }

        stage("TRIVY Image Scan") {
            steps {
                sh "trivy image --format table -o trivy-image-report.html jacksneel/$JOB_NAME:latest "
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    dir('Kubernetes') {
                        withKubeConfig(credentialsId: 'k8s-cred', namespace: 'webapps', serverUrl: 'https://172.31.25.156:6443') {
                            sh 'kubectl apply -f deployment.yml'
                            sh 'kubectl apply -f service.yml'
                        }
                    }
                }
            }
        }
    }

    // Post-build Email Notification
    post {
        always {
            emailext(
                subject: "Jenkins Build #${BUILD_NUMBER} - ${JOB_NAME}",
                body: """<p>Build Status: ${currentBuild.currentResult}</p>
                         <p>Check the build details: <a href="${BUILD_URL}">${BUILD_URL}</a></p>""",
                to: 'sunielmahla@gmail.com',
                replyTo: 'sunielmahla@gmail.com',
                mimeType: 'text/html'
            )
        }
    }
}
