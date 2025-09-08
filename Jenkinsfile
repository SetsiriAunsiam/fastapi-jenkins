pipeline {
    agent {
        docker {
            image 'python:3.11'
            args '-v /var/run/docker.sock:/var/run/docker.sock --user root'
        }
    }
    environment {
        SONARQUBE = credentials('sonar-token')
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/SetsiriAunsiam/fastapi-jenkins.git'
            }
        }
        stage('System deps for SonarScanner') {
            steps {
                sh '''
                apt-get update
                apt-get install -y --no-install-recommends default-jre-headless ca-certificates
                java -version
                '''
            }
        }
        stage('Setup venv') {
            steps {
                sh '''
                python3 -m venv venv
                . venv/bin/activate
                pip install --upgrade pip
                pip install -r requirements.txt
                pip install pytest-cov coverage || true
                '''
            }
        }
        stage('Run Tests & Coverage') {
            steps {
                sh '''
                venv/bin/pytest -q --maxfail=1 --disable-warnings --cov=app --cov-report=xml:coverage.xml || true
                test -f coverage.xml && echo "Coverage XML generated." || echo "No coverage.xml"
                '''
            }
        }
        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool name: 'SonarScanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
                    withSonarQubeEnv('SonarQube') {
                        sh """
                        sonar-scanner \
                        -Dsonar.projectKey=fastapi-app \
                        -Dsonar.projectName='fastapi app' \
                        -Dsonar.sources=./app \
                        -Dsonar.token=$SONARQUBE
                        """
                    }
                }
            }    
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Install Docker CLI') {
            steps {
                sh '''
                apt-get update
                apt-get install -y --no-install-recommends docker-cli
                which docker && docker --version
                '''
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t fastapi-app:latest .'
            }
        }
        stage('Deploy Container') {
            steps {
                sh 'docker run -d -p 8000:8000 fastapi-app:latest'
            }
        }
    }
    post {
        always {
            echo "Pipeline finished"
        }
    }
}
