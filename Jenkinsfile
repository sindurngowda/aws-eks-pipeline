pipeline {
    agent any

    environment {
        AWS_REGION    = "ap-south-2"
        CLUSTER_NAME  = "my-eks-cluster"
        TF_PLUGIN_CACHE_DIR = "${HOME}/.terraform.d/plugin-cache"
    }

    stages {

        /* -------------------------
           1. CHECKOUT
        -------------------------- */
        stage('Checkout') {
            steps { checkout scm }
        }

        /* -------------------------
           2. AWS CREDENTIALS
        -------------------------- */
        stage('Configure AWS Credentials') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {

                    sh '''
                        echo "Configuring AWS CLI..."
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                        aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        aws configure set region ${AWS_REGION}
                        aws sts get-caller-identity
                    '''
                }
            }
        }

        /* -------------------------
           3. TERRAFORM (FAST MODE)
        -------------------------- */
        stage('Terraform Provision Infra') {
            steps {
                dir('terraform') {

                    sh '''
                      echo "Enabling Terraform Plugin Cache..."
                      mkdir -p $TF_PLUGIN_CACHE_DIR
                    '''

                    timeout(time: 40, unit: 'MINUTES') {
                        sh '''
                            set -e

                            echo "Terraform Init (cached)..."
                            terraform init -input=false -no-color

                            echo "Terraform Plan..."
                            terraform plan -out=tfplan -input=false -no-color -parallelism=10

                            echo "Terraform Apply..."
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

        /* -------------------------
           4. DOCKER BUILD (CACHED)
        -------------------------- */
        stage('Build & Push Docker Image') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {

                    sh '''
                        echo "Logging into ECR..."
                        aws ecr get-login-password --region ${AWS_REGION} \
                          | docker login --username AWS --password-stdin ${ECR_URL}

                        echo "Building Docker image using cache..."
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

        /* -------------------------
           5. HELM DEPLOY
        -------------------------- */
        stage('Deploy to EKS using Helm') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {

                    sh '''
                        echo "Updating kubeconfig..."
                        aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}
                    '''

                    dir('helm/my-app') {
                        sh '''
                            echo "Deploying app using Helm..."
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
        success { echo "🚀 Pipeline Completed Successfully!" }
        failure { echo "❌ Pipeline Failed. Check logs." }
    }
}
