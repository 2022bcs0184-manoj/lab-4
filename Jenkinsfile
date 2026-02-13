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

        stage('Setup Python Environment') {
            steps {
                sh '''
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

        stage('Read Metrics') {
            steps {
                script {
                    env.NEW_R2 = sh(
                        script: "jq .r2 metrics.json",
                        returnStdout: true
                    ).trim()

                    echo "New R2 = ${env.NEW_R2}"
                }
            }
        }

        stage('Compare With Best') {
            steps {
                script {

            withCredentials([string(credentialsId: 'BEST_R2', variable: 'BEST_R2_VAL')]) {

                def best = BEST_R2_VAL.toFloat()
                def current = env.NEW_R2.toFloat()

                echo "Best R2 = ${best}"
                echo "Current R2 = ${current}"

                if (current > best) {
                    env.BUILD_DOCKER = "true"
                    echo "Model improved ✅"
                } else {
                    env.BUILD_DOCKER = "false"
                    echo "Model not improved ❌"
                }
            }
        }
            }
        }

        stage('Build Docker Image') {
            when {
                environment name: 'BUILD_DOCKER', value: 'true'
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
                environment name: 'BUILD_DOCKER', value: 'true'
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS')]) {

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
            archiveArtifacts artifacts: 'metrics.json, model.pkl', fingerprint: true
        }
    }
}
