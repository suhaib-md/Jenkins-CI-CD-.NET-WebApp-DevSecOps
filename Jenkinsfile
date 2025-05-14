pipeline{
    agent any
    tools{
        jdk 'jdk17'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
        BUILD_NUMBER=sh(script: 'echo ${BUILD_NUMBER}', returnStdout: true).trim()
        IMAGE_NAME = "suhaibmdv/dotnet-monitoring"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout From Git'){
            steps{
                git branch: 'main', url: 'https://github.com/suhaib-md/Jenkins-CI-CD-.NET-WebApp-DevSecOps'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Dotnet-Webapp \
                    -Dsonar.projectKey=Dotnet-Webapp '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage("TRIVY File scan"){
            steps{
                sh "trivy fs . > trivy-fs_report.txt"
            }
        }
        stage("OWASP Dependency Check"){
            steps{
                dependencyCheck additionalArguments: '--scan ./ --format XML ', odcInstallation: 'OWASP-Dependency-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage("Docker Build & tag"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       // Use the Makefile with the environment variable
                       sh "IMAGE_TAG=${IMAGE_TAG} make image"
                       // Also tag as latest
                       sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest"
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image ${IMAGE_NAME}:${IMAGE_TAG} > trivy.txt"
            }
        }
        stage("Docker Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       // Use the make push command
                       sh "IMAGE_TAG=${IMAGE_TAG} make push"
                       // Also push the latest tag
                       sh "docker push ${IMAGE_NAME}:latest"
                    }
                }
            }
        }
        stage("Deploy to container"){
            steps{
                script {
                    // Remove existing container if it exists
                    sh 'docker rm -f dotnet || true'
                    // Run new container with the new image
                    sh "docker run -d --name dotnet -p 5000:5000 ${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }
        stage('Deploy to k8s'){
            steps{
                dir('K8S') {
                    script {
                        // Update deployment.yaml with new image tag
                        sh "sed -i 's|${IMAGE_NAME}:latest|${IMAGE_NAME}:${IMAGE_TAG}|g' deployment.yaml"
                        
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'kubernetes', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                            sh 'kubectl apply -f deployment.yaml'
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            // Clean up local Docker images to prevent disk space issues
            sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
            sh "docker rmi ${IMAGE_NAME}:latest || true"
        }
    }
}