pipeline {
    agent any

    environment {
        AWS_REGION    = "ap-south-2"
        CLUSTER_NAME  = "my-eks-cluster"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Configure AWS Credentials') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'ID',
                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
                ]) {
                    sh '''
                        echo "Configuring AWS CLI..."
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                        aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        aws configure set default.region ${AWS_REGION}
                        aws sts get-caller-identity
                    '''
                }
            }
        }

        stage('Terraform Provision Infra') {
            steps {
                dir('terraform') {
                    withCredentials([
                        [$class: 'AmazonWebServicesCredentialsBinding',
                         credentialsId: 'ID',
                         accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                         secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
                    ]) {
                        sh '''
                          terraform init
                          terraform apply -auto-approve
                        '''
                    }

                    script {
                        env.ECR_URL = sh(
                            returnStdout: true,
                            script: "terraform output -raw ecr_repository_url"
                        ).trim()
                    }
                }
            }
        }

        stage('Build & Push Docker Image to ECR') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'ID',
                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
                ]) {
                    sh '''
                        echo "Logging into ECR..."
                        aws ecr get-login-password --region ${AWS_REGION} \
                          | docker login --username AWS --password-stdin ${ECR_URL}

                        echo "Building image..."
                    '''

                    dir('app') {
                        sh '''
                          docker build -t ${ECR_URL}:v${BUILD_NUMBER} .
                          docker push ${ECR_URL}:v${BUILD_NUMBER}
                        '''
                    }
                }
            }
        }

        stage('Deploy to EKS using Helm') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'ID',
                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
                ]) {
                    sh '''
                      aws eks --region ${AWS_REGION} update-kubeconfig --name ${CLUSTER_NAME}
                    '''

                    dir('helm/my-app') {
                        sh '''
                          helm upgrade --install my-app . \
                            --set image.repository=${ECR_URL} \
                            --set image.tag=v${BUILD_NUMBER}
                        '''
                    }
                }
            }
        }
    }
}
