pipeline {

    agent any

    environment {
        AWS_REGION    = "ap-south-1"
        CLUSTER_NAME  = "my-eks-cluster"
        TF_PLUGIN_CACHE_DIR = "${HOME}/.terraform.d/plugin-cache"
    }

    stages {

        /* -------------------------
           1. CHECKOUT
        ------------------------- */
        stage('Checkout') {
            steps { checkout scm }
        }

        /* -------------------------
           2. AWS CREDENTIALS
        ------------------------- */
        stage('Configure AWS Credentials') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {

                    sh '''
                        echo "[INFO] Configuring AWS CLI..."
                        aws configure set aws_access_key_id     $AWS_ACCESS_KEY_ID
                        aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        aws configure set region ${AWS_REGION}
                        aws sts get-caller-identity
                    '''
                }
            }
        }

        /* -------------------------
           3. TERRAFORM (FAST MODE)
        ------------------------- */
        stage('Terraform Apply Infra') {
            steps {
                dir('terraform') {

                    sh '''
                        echo "[INFO] Enabling Terraform Plugin Cache..."
                        mkdir -p $TF_PLUGIN_CACHE_DIR
                    '''

                    timeout(time: 30, unit: 'MINUTES') {
                        sh '''
                            set -e

                            echo "[INFO] Terraform Init (cached)..."
                            terraform init -input=false -no-color

                            echo "[INFO] Terraform Plan..."
                            terraform plan \
                                -out=tfplan \
                                -input=false \
                                -parallelism=10 \
                                -no-color

                            echo "[INFO] Terraform Apply..."
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
           4. DOCKER BUILD & PUSH 
              (RUN INSIDE DOCKER AGENT TO AVOID PERMISSION PROBLEMS)
        ------------------------- */
        stage('Build & Push Docker Image') {
            agent {
                docker {
                    image 'docker:27.0-dind'
                    args  '--privileged -v /var/run/docker.sock:/var/run/docker.sock'
                }
            }

            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {

                    sh '''
                        echo "[INFO] Logging into ECR..."
                        aws ecr get-login-password --region ${AWS_REGION} \
                            | docker login --username AWS --password-stdin ${ECR_URL}

                        echo "[INFO] Setting up Docker builder..."
                        docker buildx create --use || true
                    '''

                    dir('app') {
                        sh '''
                            echo "[INFO] Building Docker image with cache..."

                            docker build \
                                --cache-from ${ECR_URL}:latest \
                                -t ${ECR_URL}:latest \
                                -t ${ECR_URL}:v${BUILD_NUMBER} .

                            echo "[INFO] Pushing image..."
                            docker push ${ECR_URL}:latest
                            docker push ${ECR_URL}:v${BUILD_NUMBER}
                        '''
                    }
                }
            }
        }

        /* -------------------------
           5. DEPLOY USING HELM
        ------------------------- */
        stage('Deploy to EKS via Helm') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {

                    sh '''
                        echo "[INFO] Updating kubeconfig..."
                        aws eks update-kubeconfig \
                            --name ${CLUSTER_NAME} \
                            --region ${AWS_REGION}
                    '''

                    dir('helm/my-app') {
                        sh '''
                            echo "[INFO] Deploying using Helm..."
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
        success { echo "🚀 SUCCESS: Full CI/CD pipeline completed!" }
        failure { echo "❌ Pipeline FAILED. Check logs." }
        always  { echo "📌 Pipeline finished with status: ${currentBuild.currentResult}" }
    }
}
