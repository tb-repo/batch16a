pipeline {
    agent any
	
    parameters {
        choice(name: 'DEPLOYED_ENV', choices: ['dev', 'prod'], description: 'Select target deployment environment')
    }	
	
    environment {
        // FIXED: Restored your correct repository path
        GIT_REPO = "https://github.com/tb-repo/batch16a.git"
        DOCKER_HUB_USER = "thiagarajanb1985"
        IMAGE_NAME = "batch16a-tb-app"
        IMAGE_TAG = "${params.DEPLOYED_ENV == 'dev' ? 'dev_' + env.BUILD_ID : env.BUILD_ID}"
        
        EC2_PUBLIC_IP = "3.226.206.0" 
        EC2_USER = "ubuntu"
        
        HOST_PATH = "${params.DEPLOYED_ENV == 'dev' ? '/home/ubuntu/dev' : '/home/ubuntu/prod'}"
        TAR_FILE = "dev_image_${env.BUILD_ID}.tar"
    }

    stages {
        stage("Git Checkout") {
            steps {
                git branch: "${params.DEPLOYED_ENV}",  
                    credentialsId: 'HV-B16A-TB-Git', 
                    url: "${env.GIT_REPO}"
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${env.DOCKER_HUB_USER}/${env.IMAGE_NAME}:${env.IMAGE_TAG} ."
                    sh "docker tag ${env.DOCKER_HUB_USER}/${env.IMAGE_NAME}:${env.IMAGE_TAG} ${env.DOCKER_HUB_USER}/${env.IMAGE_NAME}:latest"
                }
            }
        }

        stage('Push to Docker Hub') {
            when {
                expression { params.DEPLOYED_ENV == 'prod' }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'HV-B16A-TB-DockerHub', 
                                                 usernameVariable: 'DOCKER_USER', 
                                                 passwordVariable: 'DOCKER_PASS')]) {
                    sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                    sh "docker push ${env.DOCKER_HUB_USER}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"
                    sh "docker push ${env.DOCKER_HUB_USER}/${env.IMAGE_NAME}:latest"
                }
            }
        }

        stage('Install Docker and Prepare Folders on EC2') {
            steps {
                sshagent(['HV-B16A-TB-EC2-Keypair']) {
					sh "ssh -o StrictHostKeyChecking=no ${env.EC2_USER}@${env.EC2_PUBLIC_IP} 'mkdir -p ${env.HOST_PATH} && if ! command -v docker > /dev/null; then echo \"Installing Docker via convenience script...\"; curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh && sudo usermod -aG docker ${env.EC2_USER} && rm -f get-docker.sh; else echo \"Docker already installed\"; fi'"
                }
            }
        }

        stage('Transfer Dev Image via SCP') {
            when {
                expression { params.DEPLOYED_ENV == 'dev' }
            }
            steps {
                sshagent(['HV-B16A-TB-EC2-Keypair']) {
                    script {
                        sh "docker save -o ${env.TAR_FILE} ${env.DOCKER_HUB_USER}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"
                        sh "scp -o StrictHostKeyChecking=no ${env.TAR_FILE} ${env.EC2_USER}@${env.EC2_PUBLIC_IP}:${env.HOST_PATH}/"
                        sh "rm ${env.TAR_FILE}"
                    }
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(['HV-B16A-TB-EC2-Keypair']) {
                    script {
                        def targetDir = params.DEPLOYED_ENV == 'dev' ? '/home/ubuntu/dev' : '/home/ubuntu/prod'
                        
                        sh "ssh -o StrictHostKeyChecking=no ${env.EC2_USER}@${env.EC2_PUBLIC_IP} 'echo \"Connected to EC2. Deploying to ${params.DEPLOYED_ENV}...\"; cd ${targetDir} && if [ \"${params.DEPLOYED_ENV}\" = \"dev\" ]; then echo \"Loading local dev image tarball...\"; sg docker -c \"docker load -i ${targetDir}/${env.TAR_FILE}\" && rm -f ${targetDir}/${env.TAR_FILE}; else echo \"Pulling prod image from Docker Hub...\"; sg docker -c \"docker pull ${env.DOCKER_HUB_USER}/${env.IMAGE_NAME}:${env.IMAGE_TAG}\"; fi && sg docker -c \"docker stop ${env.IMAGE_NAME}-${params.DEPLOYED_ENV}-container || true\" && sg docker -c \"docker rm ${env.IMAGE_NAME}-${params.DEPLOYED_ENV}-container || true\" && echo \"--- EXECUTING PYTHON APPLICATION OUTPUT ---\" && sg docker -c \"docker run --name ${env.IMAGE_NAME}-${params.DEPLOYED_ENV}-container ${env.DOCKER_HUB_USER}/${env.IMAGE_NAME}:${env.IMAGE_TAG}\" && echo \"-----------------------------------------\" && echo \"${params.DEPLOYED_ENV} setup finished successfully!\"'"
                    }
                }
            }
        }
    }
    
        post {
        always {
            script {
                echo "Starting post-build cleanup operations..."
                
                // 1. Clean up the specific tagged images you built
                sh "docker rmi ${env.DOCKER_HUB_USER}/${env.IMAGE_NAME}:${env.IMAGE_TAG} || true"
                sh "docker rmi ${env.DOCKER_HUB_USER}/${env.IMAGE_NAME}:latest || true"
                
                // 2. Clean up the temporary local tar file if it exists
                sh "rm -f ${env.TAR_FILE}"
                
                // 3. Remove dangling build layers/cache to save Jenkins disk space
                sh "docker image prune -f || true"
                
                // 4. Remove stopped/dead containers on the Jenkins machine (if any)
                sh "docker container prune -f || true"
            }
        }
        success {
            echo "Build and Deployment completed successfully! Clearing workspace..."
            // 5. Wipes the Jenkins workspace directory completely to keep the agent disk clean
            cleanWs()
        }
        failure {
            echo "Pipeline failed. Keeping workspace files temporarily for troubleshooting/logs."
        }
    }
}