pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'node23'
    }

    environment {
        SCANNER_HOME = tool 'sq'
        IMAGE_NAME = 'satyasaia99/bms:latest'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/main']],
                    extensions: [],
                    userRemoteConfigs: [[url: 'https://github.com/SatyasaiA99/Book-My-Show.git']]
                )

                sh 'ls -la'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sq') {
                    sh """
                    ${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectName=BMS \
                    -Dsonar.projectKey=BMS
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sq'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                cd bookmyshow-app

                if [ -f package-lock.json ]; then
                    npm ci
                else
                    npm install
                fi
                '''
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh '''
                trivy fs --scanners vuln . > trivyfs.txt
                '''
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    sh '''
                    docker build -t $IMAGE_NAME \
                    -f bookmyshow-app/Dockerfile \
                    bookmyshow-app
                    '''
                }
            }
        }

        stage('Trivy Docker Image Scan') {
            steps {
                sh '''
                trivy image $IMAGE_NAME > trivyimage.txt
                '''
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'Dockerhub', toolName: 'docker') {

                        sh '''
                        docker push $IMAGE_NAME
                        '''
                    }
                }
            }
        }

        stage('Deploy Container') {
            steps {
                sh '''
                echo "Stopping old container..."
                docker stop bms || true
                docker rm bms || true

                echo "Running new container..."
                docker run -d \
                --restart=always \
                --name bms \
                -p 3000:3000 \
                $IMAGE_NAME

                echo "Running Containers:"
                docker ps -a

                echo "Application Logs:"
                sleep 10
                docker logs bms
                '''
            }
        }
    }

    post {
        always {
            emailext(
                attachLog: true,
                subject: "${currentBuild.result}: Job ${env.JOB_NAME}",
                body: """
                Project: ${env.JOB_NAME}<br/>
                Build Number: ${env.BUILD_NUMBER}<br/>
                Build URL: ${env.BUILD_URL}<br/>
                """,
                to: 'satyaankam69@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
            )
        }
    }
}
