pipeline {
  agent any

  environment {
    AWS_REGION   = 'us-east-1'
    INSTANCE_ID  = 'i-01173d56ad0ebfc7c'
    REPO_URL     = 'https://github.com/barnavo05/Devops-project.git'
    APP_DIR      = '/opt/app/Devops-project'
    CONTAINER    = 'react-container'
  }

  // ⏱ Poll GitHub every 1 minute for changes
  triggers {
    pollSCM('*/1 * * * *')   // Every 1 minute
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Set Image Tag') {
      steps {
        script {
          env.IMAGE_TAG = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
          echo "IMAGE_TAG=${env.IMAGE_TAG}"
        }
      }
    }

    stage('Deploy to EC2 via SSM') {
      steps {
        withAWS(credentials: 'access-key', region: "${AWS_REGION}") {
          sh """
            aws ssm send-command \
              --instance-ids ${INSTANCE_ID} \
              --document-name "AWS-RunShellScript" \
              --comment "Deploy React App" \
              --parameters 'commands=[
                "set -e",
                "command -v git >/dev/null 2>&1 || (sudo yum -y install git)",
                "mkdir -p /opt/app",
                "cd /opt/app",
                "if [ -d Devops-project ]; then cd Devops-project && git checkout main && git pull; else git clone ${REPO_URL}; cd Devops-project; fi",
                "docker rm -f ${CONTAINER} || true",
                "docker build -t react-app:${IMAGE_TAG} -f Dockerfile.dev .",
                "docker run -d --restart unless-stopped --name ${CONTAINER} -p 3000:80 react-app:${IMAGE_TAG}",
                "docker ps -a"
              ]'
          """
        }
      }
    }
  }

  post {
    success {
      echo "✅ Successfully deployed commit ${IMAGE_TAG} on EC2 (port 3000)"
    }
    failure {
      echo "❌ Deployment failed — check Jenkins logs or SSM output"
    }
  }
}
