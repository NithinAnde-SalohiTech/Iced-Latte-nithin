pipeline {
    agent any

    environment {
        APP_EC2_IP = '172.31.10.254'
    }

    tools {
        jdk 'JAVA-25'
        maven 'MAVEN'
    }

    stages {

        stage('Git Checkout') {
            steps {
                git url: 'https://github.com/NithinAnde-SalohiTech/Iced-Latte-nithin.git',
                    branch: 'development'
            }
        }
        stage('Build') {
            steps {
                sh 'mvn spotless:apply'
                sh 'mvn clean package -DskipTests'
            }
        }
        stage('Sonar Scan') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'SONAR_ID',
                        variable: 'SONAR_TOKEN'
                    )
                ]) {
                    withSonarQubeEnv('sonarqube') {
                        sh '''
                            mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.6.0.6792:sonar \
                                -Dsonar.projectKey=nithinande-salohitech \
                                -Dsonar.organization=nithinande-salohitech \
                                -Dsonar.host.url=https://sonarcloud.io \
                                -Dsonar.token=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build \
                        -t nithinandedocker/iced-latte:latest .
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'DOCKER_ID',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "nithinandedocker" \
                            --password-stdin

                        docker push nithinandedocker/iced-latte:latest
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'APP_EC2_SSH',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no \
                            -i "$SSH_KEY" \
                            "$SSH_USER@$APP_EC2_IP" \
                            "
                                docker pull nithinandedocker/iced-latte:latest
                                docker network create iced-network 2>/dev/null || true
                                docker rm -f redis 2>/dev/null || true

                                docker run -d \
                                    --name redis \
                                    --network iced-network \
                                    redis:7

                                docker rm -f iced-latte 2>/dev/null || true
                                docker run -d \
                                  --name iced-latte \
                                  --network iced-network \
                                  -p 8080:8080 \
                                  -e REDIS_HOST=redis \
                                  -e REDIS_PORT=6379 \
                                  nithinandedocker/iced-latte:latest
                            "
                    '''
                }
            }
        }
    }
}
