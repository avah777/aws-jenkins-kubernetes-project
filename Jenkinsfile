
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'avah777/aws-jenkins-kubernetes'
        K8S_NAMESPACE = 'devops-demo'
        K8S_DEPLOYMENT = 'aws-java-app'
        EKS_CLUSTER = 'devops-platform-eks'
        AWS_DEFAULT_REGION = 'us-east-1'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Check Java Environment') {
            steps {
                sh '''
                    echo "===== JAVA VERSION ====="
                    java -version

                    echo "===== JAVAC VERSION ====="
                    javac -version

                    echo "===== MAVEN VERSION ====="
                    mvn -version

                    echo "===== JAVA HOME ====="
                    echo "$JAVA_HOME"
                '''
            }
        }

        stage('Build and Test') {
            steps {
                sh 'mvn clean verify'
            }
        }

        stage('Check Files') {
            steps {
                sh '''
                    echo "===== CURRENT DIRECTORY ====="
                    pwd

                    echo "===== PROJECT FILES ====="
                    ls -la

                    echo "===== TARGET DIRECTORY ====="
                    ls -la target/

                    echo "===== DOCKERFILE ====="
                    ls -la Dockerfile
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                    docker build \
                    -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                    -t ${DOCKER_IMAGE}:latest .
                """
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                        -u "$DOCKER_USERNAME" \
                        --password-stdin

                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        docker push ${DOCKER_IMAGE}:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'jenkins-eks-policy2']
                ]) {
                    withKubeConfig([
                        credentialsId: 'kubeconfig'
                    ]) {
                        sh """
                            echo "Checking EKS cluster..."
                            aws eks describe-cluster \
                            --name ${EKS_CLUSTER} \
                            --region ${AWS_DEFAULT_REGION}

                            echo "Updating Kubernetes deployment..."

                            kubectl set image \
                            deployment/${K8S_DEPLOYMENT} \
                            ${K8S_DEPLOYMENT}=${DOCKER_IMAGE}:${BUILD_NUMBER} \
                            -n ${K8S_NAMESPACE}

                            echo "Checking deployment rollout..."

                            kubectl rollout status \
                            deployment/${K8S_DEPLOYMENT} \
                            -n ${K8S_NAMESPACE}
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD deployment successful!'
        }

        failure {
            echo 'CI/CD pipeline failed.'
        }
    }
}
