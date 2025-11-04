pipeline {
  agent any

  environment {
    AWS_REGION  = 'us-east-1'
    AWS_ACCOUNT = '288639564583'
    ECR_REPO    = 'final-project' // lowercase only
    ECR_URI     = "${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
    INSTANCE_ID = 'i-01173d56ad0ebfc7c'
  }

  triggers {
    githubPush()
  }

  stages {

    stage('Checkout from GitHub') {
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

    stage('Build Docker Image') {
      steps {
        sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} -f Dockerfile.dev ."
      }
    }

    stage('Push to ECR') {
      steps {
        withAWS(credentials: 'access-key', region: "${AWS_REGION}") {
          sh '''
            aws ecr describe-repositories --repository-names ${ECR_REPO} --region ${AWS_REGION} \
              || aws ecr create-repository --repository-name ${ECR_REPO} --region ${AWS_REGION}

            aws ecr get-login-password --region ${AWS_REGION} | \
              docker login --username AWS --password-stdin ${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com

            docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_URI}:${IMAGE_TAG}
            docker push ${ECR_URI}:${IMAGE_TAG}
          '''
        }
      }
    }

    stage('Deploy on EC2 via SSM') {
      steps {
        withAWS(credentials: 'access-key', region: "${AWS_REGION}") {
          sh """
            aws ssm send-command \
              --instance-ids ${INSTANCE_ID} \
              --document-name "AWS-RunShellScript" \
              --comment "Deploy React App" \
              --parameters commands='[
                "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_URI}",
                "docker pull ${ECR_URI}:${IMAGE_TAG}",
                "docker rm -f react-container || true",
                "docker run -d --name react-container -p 3000:80 ${ECR_URI}:${IMAGE_TAG}",
                "docker ps -a"
              ]'
          """
        }
      }
    }
  }

  post {
    success {
      echo "✅ Successfully deployed ${ECR_URI}:${IMAGE_TAG} → EC2 Port 3000"
    }
    failure {
      echo "❌ Deployment failed — check logs"
    }
  }
}
