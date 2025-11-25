pipeline {
    agent any

    environment {
        AWS_REGION  = 'ap-south-2'
        CLUSTER_NAME = 'my-eks-cluster'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Terraform Provision Infra') {
            steps {
                withAWS(credentials:'aws-creds', region:"${AWS_REGION}") {
                    dir('terraform') {
                        sh 'terraform init'
                        sh 'terraform apply -auto-approve'
                        script {
                            env.ECR_URL = sh(
                                returnStdout: true,
                                script: "terraform output -raw ecr_repository_url"
                            ).trim()
                        }
                    }
                }
            }
        }

        stage('Build & Push Image to ECR') {
            steps {
                withAWS(credentials:'aws-creds', region:"${AWS_REGION}") {
                    script {
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_URL}"
                    }
                    dir('app') {
                        sh "docker build -t ${ECR_URL}:v${BUILD_NUMBER} ."
                        sh "docker push ${ECR_URL}:v${BUILD_NUMBER}"
                    }
                }
            }
        }

        stage('Deploy to EKS with Helm') {
            steps {
                withAWS(credentials:'aws-creds', region:"${AWS_REGION}") {
                    sh "aws eks --region ${AWS_REGION} update-kubeconfig --name ${CLUSTER_NAME}"

                    dir('helm/my-app') {
                        sh """
                        helm upgrade --install my-app . \
                          --set image.repository=${ECR_URL} \
                          --set image.tag=v${BUILD_NUMBER}
                        """
                    }
                }
            }
        }
    }
}
