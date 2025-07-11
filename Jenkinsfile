pipeline {
  agent { label 'infra-build-node' }

  environment {
    AWS_DEFAULT_REGION = 'us-east-1'
  }

  stages {
    stage('Install dependencies on slave') {
      steps {
        sh '''
          # Update package lists and install python3-venv
          sudo apt-get update
          sudo apt-get install -y python3-venv unzip curl sudo

          # Install AWS CLI v2 (with --update to avoid reinstall)
          curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
          unzip -q awscliv2.zip
          sudo ./aws/install --update
          rm -rf awscliv2.zip aws

          # Set up Python virtual environment
          python3 -m venv venv
          . venv/bin/activate

          # Upgrade pip and install Python packages in venv
          pip install --upgrade pip
          pip install boto3 botocore

          # Install Ansible AWS collection
          ansible-galaxy collection install amazon.aws --force

          # Install Node.js 18
          curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
          sudo apt-get install -y nodejs
        '''
      }
    }

    stage('Provision Infrastructure') {
      steps {
        sshagent(credentials: ['ssh-agent-key']) {
          withCredentials([
            usernamePassword(
              credentialsId: 'jenkins-ec2-access',
              usernameVariable: 'AWS_ACCESS_KEY_ID',
              passwordVariable: 'AWS_SECRET_ACCESS_KEY'
            )
          ]) {
            sh '''
              . venv/bin/activate
              ansible-playbook -i inventory/prod/aws_ec2.yml playbooks/site.yml
            '''
          }
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'inventory.json', allowEmptyArchive: true
      cleanWs()
    }
    success {
      echo 'Infrastructure provisioned successfully.'
    }
    failure {
      echo 'Build failed. Check archived inventory.json for details.'
    }
  }
}
