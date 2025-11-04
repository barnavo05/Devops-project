pipeline {
  agent any

  environment {
    AWS_REGION  = 'us-east-1'
    AWS_ACCOUNT = '288639564583'
    ECR_REPO    = 'final-project' // must be lowercase for ECR
    ECR_URI     = "${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
    INSTANCE_ID = 'i-01173d56ad0ebfc7c'
  }

  triggers { githubPush() }

  stages {
    stage('Checkout from GitHub') {
      steps { checkout scm }
    }

    stage('Set Image Tag') {
      steps {
        script {
          env.IMAGE_TAG = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
          echo "IMAGE_TAG=${env.IMAGE_TAG}"
        }
      }
    }

    stage('Build Docker image') {
      steps {
        // ✅ CHANGE #1: use new Dockerfile
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
          sh '''
            aws ssm send-command \
              --instance-ids ${INSTANCE_ID} \
              --document-name "AWS-RunShellScript" \
              --comment "Deploy React App" \
              --parameters commands="[
                \\"aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com\\",
                \\"docker pull ${ECR_URI}:${IMAGE_TAG}\\",
                \\"docker rm -f react-container || true\\",
                
                # ✅ CHANGE #2: expose container port 80 -> EC2 port 3000
                \\"docker run -d --name react-container -p 3000:80 ${ECR_URI}:${IMAGE_TAG}\\",

                \\"docker ps -a\\"
              ]"
          '''
        }
      }
    }
  }

  post {
    success { echo "✅ Deployed ${ECR_URI}:${IMAGE_TAG} to EC2:3000" }
    failure { echo "❌ Pipeline failed — check logs" }
  }
}
