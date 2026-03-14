pipeline {
    agent any

    environment {
        IMAGE_NAME = "2022bcs0184manoj/2022bcs0184-lab6"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Python & Setup Virtual Environment') {
            steps {
                sh '''
                apt-get update
                apt-get install -y python3 python3-venv python3-pip

                python3 -m venv venv
                . venv/bin/activate

                pip install --upgrade pip
                pip install -r requirements.txt
                '''
            }
        }

        stage('Train Model') {
            steps {
                sh '''
                . venv/bin/activate
                python scripts/train.py
                '''
            }
        }

        stage('Read Accuracy') {
            steps {
                script {

                    def metrics = readJSON file: 'app/artifacts/metrics.json'

                    env.CURRENT_ACCURACY = metrics.accuracy.toString()

                    echo "Current Accuracy = ${env.CURRENT_ACCURACY}"
                }
            }
        }

        stage('Compare Accuracy') {
            steps {
                script {

                    withCredentials([string(credentialsId: 'best-accuracy', variable: 'BEST_ACC')]) {

                        def best = BEST_ACC.toFloat()
                        def current = env.CURRENT_ACCURACY.toFloat()

                        echo "Best Accuracy = ${best}"
                        echo "Current Accuracy = ${current}"

                        if (current > best) {
                            env.BUILD_DOCKER = "true"
                            echo "Model improved ✅"
                        } else {
                            env.BUILD_DOCKER = "false"
                            echo "Model did not improve ❌"
                        }
                    }
                }
            }
        }

        stage('Build Docker Image') {
            when {
                expression { env.BUILD_DOCKER == "true" }
            }

            steps {
                sh '''
                docker build -t $IMAGE_NAME:${BUILD_NUMBER} .
                docker tag $IMAGE_NAME:${BUILD_NUMBER} $IMAGE_NAME:latest
                '''
            }
        }

        stage('Push Docker Image') {
            when {
                expression { env.BUILD_DOCKER == "true" }
            }

            steps {

                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {

                    sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    docker push $IMAGE_NAME:${BUILD_NUMBER}
                    docker push $IMAGE_NAME:latest
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'app/artifacts/**', fingerprint: true
        }
    }
}
