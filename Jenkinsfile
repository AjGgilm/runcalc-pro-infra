pipeline {
    agent any
    parameters {
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Docker image tag to deploy (e.g. v1.0.4)')
    }
    environment {
        DOCKERHUB_REPO = "gilma02/runcalc-pro"
    }
    stages {
        stage('Provision EC2') {
            steps {
                sh '''
                    echo "18.208.212.118" > prod_ip.txt
                    echo "Using existing prod instance"
                '''
            }
        }
        stage('Configure & Deploy') {
            steps {
                withCredentials([string(credentialsId: 'ansible-vault-password', variable: 'VAULT_PASS')]) {
                    sh '''
                        PROD_IP=$(cat prod_ip.txt)
                        echo "[prod]" > inventory.ini
                        echo "prod_server ansible_host=$PROD_IP ansible_user=ec2-user ansible_ssh_private_key_file=/var/lib/jenkins/.ssh/prod_key.pem" >> inventory.ini
                        echo "$VAULT_PASS" > /tmp/vault_pass.txt
                        ansible-playbook -i inventory.ini deploy.yml \
                            --vault-password-file /tmp/vault_pass.txt \
                            --extra-vars "image_tag=gilma02/runcalc-pro:${IMAGE_TAG}" \
                            -e "ansible_ssh_common_args='-o StrictHostKeyChecking=no'"
                        rm /tmp/vault_pass.txt
                    '''
                }
            }
        }
        stage('Verify Deployment') {
            steps {
                sh '''
                    PROD_IP=$(cat prod_ip.txt)
                    sleep 10
                    curl -f http://$PROD_IP || exit 1
                    echo "Application is live at http://$PROD_IP"
                '''
            }
        }
    }
    post {
        success {
            sh 'echo "Deployment successful!"'
        }
        failure {
            sh 'echo "Deployment failed. Check logs above."'
        }
    }
}
