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
                withCredentials([
                    string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY'),
                    string(credentialsId: 'aws-session-token', variable: 'AWS_SESSION_TOKEN')
                ]) {
                    sh '''
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                        aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        aws configure set aws_session_token $AWS_SESSION_TOKEN
                        aws configure set region us-east-1

                        INSTANCE_ID=$(aws ec2 run-instances \
                            --image-id ami-0c02fb55956c7d316 \
                            --instance-type t2.micro \
                            --key-name prod-key \
                            --security-group-ids sg-0d5883df1c11bd157 \
                            --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=runcalc-prod}]" \
                            --query "Instances[0].InstanceId" \
                            --output text)

                        echo "Instance ID: $INSTANCE_ID"
                        echo $INSTANCE_ID > instance_id.txt

                        aws ec2 wait instance-running --instance-ids $INSTANCE_ID

                        PROD_IP=$(aws ec2 describe-instances \
                            --instance-ids $INSTANCE_ID \
                            --query "Reservations[0].Instances[0].PublicIpAddress" \
                            --output text)

                        echo "Prod IP: $PROD_IP"
                        echo $PROD_IP > prod_ip.txt
                    '''
                }
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

                        until ssh -o StrictHostKeyChecking=no -i /var/lib/jenkins/.ssh/prod_key.pem ec2-user@$PROD_IP 'echo ready'; do
  echo "Waiting for SSH..."
  sleep 5
done

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
