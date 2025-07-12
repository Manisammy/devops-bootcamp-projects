pipeline {
  agent { label 'infra-build-node' }

  environment {
    AWS_DEFAULT_REGION = 'us-east-1'
  }

  stages {
    stage('Install dependencies on slave') {
      steps {
        sh '''
          sudo apt-get update
          sudo apt-get install -y python3-venv unzip curl sudo

          curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
          unzip -q awscliv2.zip
          sudo ./aws/install --update
          rm -rf awscliv2.zip aws

          python3 -m venv venv
          . venv/bin/activate

          pip install --upgrade pip
          pip install boto3 botocore ansible

          ansible-galaxy collection install amazon.aws --force

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
              ./venv/bin/ansible-playbook -i inventory/prod/aws_ec2.yml playbooks/site.yml -u ubuntu
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
