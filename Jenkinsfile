pipeline {
    agent any
    
    tools {
        jdk 'jdk'
        maven 'maven'
    }
    
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
        APP_NAME = "boardgame-devsecops"
        RELEASE = "1.0"
        DOCKER_USER = "naman96"
        DOCKER_PASS = 'docker-cred'
    	IMAGE_NAME = "${DOCKER_USER}" + "/" + "${APP_NAME}"
    	IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    }

    stages {
        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }
        
        stage('SCM Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/naman-6/Board-Game-devsecops.git'
            }
        }
        
        stage('Compile') {
            steps {
                script {
                    sh 'mvn clean compile'
                }
            }
        }
        
        stage('Test') {
            steps {
                script {
                    sh 'mvn test'
                }
            }
        }
        
        stage('File System Scan: Trivy') {
            steps {
                script {
                    sh 'trivy fs --format table -o trivy-file-scan-report.html .'
                }
            }
        }
        
        stage('Static Code Analysis: SonarQube') {
            steps {
                script {
                    withSonarQubeEnv('sonar-server') {
                        sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Boardgame -Dsonar.java.binaries=. -Dsonar.projectKey=Boardgame'''
                    }
                }
            }
        }
        
        stage('Quality Gate Check: SonarQube') {
            steps {
                script {
                    waitForQualityGate abortPipeline:false, credentialsId: 'sonar-token'
                }
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        // stage('Publish Artifact: Nexus') {
        //     steps {
        //         script {
        //             withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk', maven: 'maven', mavenSettingsConfig: '', traceability: true) {
        //                 sh 'mvn deploy'
        //             }
        //         }
        //     }
        // }
        
        stage('Dependency Check: OWASP') {
            steps {
                script {
                    dependencyCheck additionalArguments: '--format HTML --scan target/', nvdCredentialsId: 'nvd-api', odcInstallation: 'owasp'
                }
            }
        }
        
        stage('Docker Image Build') {
            steps {
                script {
                    docker.withRegistry('', DOCKER_PASS) {
			            docker_image = docker.build "${IMAGE_NAME}"
                    }
                }
            }
        }
        
        stage('Docker Image Scan: Trivy') {
            steps {
                script {
                    sh 'trivy image --format table -o trivy-docker-scan-report.html ${IMAGE_NAME}'
                }
            }
        }
        
        stage('Docker Image Push: DockerHub') {
            steps {
                script {
                    docker.withRegistry('', DOCKER_PASS) {
			            docker_image.push("${IMAGE_TAG}")
            			docker_image.push('latest')
                    }
                }
            }
        }
        
        stage('Docker Image Cleanup') {
            steps {
                script {
                    sh 'docker rmi ${IMAGE_NAME}'
                    sh 'docker rmi ${IMAGE_NAME}:${IMAGE_TAG}'
                }
            }
        }

	stage('Deploy to Kubernetes') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8s-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.11.171:6443') {
		        sh "kubectl apply -f deployment-service.yml"
		    }
                }
            }
        }

	stage('Verify the Deployment') {
            steps {
               withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8s-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.11.171:6443') {
                        sh "kubectl get pods -n webapps"
                        sh "kubectl get svc -n webapps"
                }
            }
        }
        
        
    }
    
    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'
                def body = """
                    <html>
                        <body>
                            <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                                <h2>${jobName} - Build ${buildNumber}</h2>
                                <div style="background-color: ${bannerColor}; padding: 10px;">
                                    <h3 style="color: white;">Pipeline Status:${pipelineStatus.toUpperCase()}</h3>
                                </div>
                                <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                            </div>
                        </body>
                    </html>"""
                
                emailext (
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'itsmenaman06@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-file-scan-report.html, trivy-docker-scan-report.html, dependency-check-report.html'
                )
            }
        }
    }
}
