pipeline {

    agent any

    environment {
        AWS_REGION    = "ap-south-1"
        CLUSTER_NAME  = "my-eks-cluster"
        TF_PLUGIN_CACHE_DIR = "${HOME}/.terraform.d/plugin-cache"
    }

    stages {

        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Configure AWS Credentials') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {

                    sh '''
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                        aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        aws configure set region ${AWS_REGION}
                        aws sts get-caller-identity
                    '''
                }
            }
        }

        stage('Terraform Apply Infra') {
            steps {
                dir('terraform') {

                    sh '''
                        mkdir -p $TF_PLUGIN_CACHE_DIR
                    '''

                    timeout(time: 30, unit: 'MINUTES') {
                        sh '''
                            terraform init -input=false -no-color
                            terraform plan -out=tfplan -input=false -parallelism=10 -no-color
                            terraform apply -input=false -no-color tfplan
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


        
        stage('Build & Push Docker Image') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {

                    sh '''
                        aws ecr get-login-password --region ${AWS_REGION} \
                          | docker login --username AWS --password-stdin ${ECR_URL}
                    '''

                    dir('app') {
                        sh '''
                            docker build \
                              --cache-from ${ECR_URL}:latest \
                              -t ${ECR_URL}:latest \
                              -t ${ECR_URL}:v${BUILD_NUMBER} .

                            docker push ${ECR_URL}:latest
                            docker push ${ECR_URL}:v${BUILD_NUMBER}
                        '''
                    }
                }
            }
        }

        stage('Deploy to EKS using Helm') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {

                    sh '''
                        aws eks update-kubeconfig \
                            --name ${CLUSTER_NAME} \
                            --region ${AWS_REGION}
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

    post {
        success { echo "Pipeline Completed Successfully!" }
        failure { echo "Pipeline Failed. Check logs." }
    }
}
