pipeline{
    agent any

    tools{
        nodejs 'node17'
    }

    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }

    stages {

        stage('Clean Workspace'){
            steps{
                cleanWs()
            }
        }

        stage('Checkout from Git'){
            steps{
                git branch: 'main', credentialsId: 'github-token', url: 'https://github.com/Faiz-official/starbucks-project.git'
            }
        }

        stage("Sonarqube Analysis"){
            steps{
                withSonarQubeEnv('SonarQube') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=starbucks \
                    -Dsonar.projectKey=starbucks
                    '''
                }
            }
        }

        stage("Quality Gate"){
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        stage('Install Dependencies'){
            steps{
                sh "npm install"
            }
        }

        stage('TRIVY FS SCAN'){
            steps{
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh "docker build -t starbucks ."
                       sh "docker tag starbucks mydockerhub12/starbucks:latest"
                       sh "docker push mydockerhub12/starbucks:latest"
                    }
                }
            }
        }

        stage("TRIVY IMAGE SCAN"){
            steps{
                sh "trivy image mydockerhub12/starbucks:latest > trivyimage.txt"
            }
        }

        stage('Deploy Container'){
            steps{
                sh '''
                docker rm -f starbucks || true
                docker run -d --name starbucks -p 3000:3000 mydockerhub12/starbucks:latest
                '''
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
                    <p>This is a Jenkins Starbucks CICD pipeline status.</p>
                    <p>Project: ${env.JOB_NAME}</p>
                    <p>Build Number: ${env.BUILD_NUMBER}</p>
                    <p>Build Status: ${buildStatus}</p>
                    <p>Started by: ${buildUser}</p>
                    <p>Build URL: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    """,
                    to: 'faizshaikh.fs574@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
                )
            }
        }
    }
}