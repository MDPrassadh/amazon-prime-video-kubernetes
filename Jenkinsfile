pipeline{
    agent any
    tools{
        jdk 'jdk'
        nodejs 'node17'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('CLEAN WORKSPACE'){
            steps{
                cleanWs()
            }
        }
        stage('1-CHECK OUT FROM GIT'){
            steps{
                git branch: 'main', url: 'https://github.com/MDPrassadh/amazon-prime-video-kubernetes.git'
            }
        }
        stage("SONARQUBE ANALYSIS"){
            steps{
                withSonarQubeEnv('SonarQube') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=amazon-prime-video \
                    -Dsonar.projectKey=amazon-prime-video '''
                }
            }
        }
        stage("QUALITY GATES"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage('INSTALL DEPENDENCIES') {
            steps {
                sh "npm install"
            }
        }        
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("DOCKER BUILD & PUSH"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build -t amazon-prime-video ."
                       sh "docker tag amazon-prime-video aseemakram19/amazon-prime-video:latest "
                       sh "docker push aseemakram19/amazon-prime-video:latest "
                    }
                }
            }
        }
		stage('DOCKER SCOUT IMAGE') {
            steps {
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh 'docker-scout quickview aseemakram19/amazon-prime-video:latest'
                       sh 'docker-scout cves aseemakram19/amazon-prime-video:latest'
                       sh 'docker-scout recommendations aseemakram19/amazon-prime-video:latest'
                   }
                }
            }
        }

        stage("TRIVY-DOCKER-IMAGES"){
            steps{
                sh "trivy image aseemakram19/amazon-prime-video:latest > trivyimage.txt" 
            }
        }
        stage('APP DEPLOY TO DOCKER CONTAINER'){
            steps{
                sh 'docker run -d --name amazon-prime-video -p 3000:3000 aseemakram19/amazon-prime-video:latest'
            }
        }

    }
    post {
    always {
        script {
            def buildStatus = currentBuild.currentResult
            def buildUser = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')[0]?.userId ?: 'Github User'
            
            emailext (
                subject: "Pipeline ${buildStatus}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    <p>This is a Jenkins amazon-prime-video CICD pipeline status.</p>
                    <p>Project: ${env.JOB_NAME}</p>
                    <p>Build Number: ${env.BUILD_NUMBER}</p>
                    <p>Build Status: ${buildStatus}</p>
                    <p>Started by: ${buildUser}</p>
                    <p>Build URL: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                """,
                to: 'mohdaseemakram19@gmail.com',
                from: 'mohdaseemakram19@gmail.com',
                replyTo: 'mohdaseemakram19@gmail.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
            )
           }
       }

    }

}
