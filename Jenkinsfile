pipeline {
  agent any

  environment {
    // 🔧 Jenkins credentials ID for AWS ECR (Access Key + Secret)
    AWS_CREDS = credentials('ecr-creds')

    // 🪣 AWS region & repo details
    AWS_REGION = "ap-south-1"
    ECR_REPO = "674008111561.dkr.ecr.ap-south-1.amazonaws.com/siddharth-test"
  }

  options {
    timestamps()
  }

  stages {

    stage('Checkout') {
      steps {
        echo "📦 Cloning repository..."
        git branch: 'dev',
            url: 'https://github.com/siddharth0203/merge-demo.git',
            credentialsId: 'Github-ID'
        sh 'git branch -v'
      }
    }

    stage('Git Merge Ops') {
      steps {
        script {
          echo "🔄 Merging dev → main..."
          try {
            sh '''
                git config user.email "jenkins@local"
                git config user.name "Jenkins CI"
                git fetch origin main
                git checkout main
                git merge dev --no-ff -m "Auto merge by Jenkins"
            '''
            echo "✅ Merge successful."
          } catch (err) {
            sh 'git merge --abort || true'
            echo "❌ Merge Error: ${err.getMessage()}"
            sh 'git status'
            currentBuild.result = 'ABORTED'
            error("Merge failed. Aborting pipeline.")
          }
        }
      }
    }

    stage('Build Docker Image') {
      when {
        expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
      }
      steps {
        script {
          echo "🐳 Building Docker image..."
          sh """
            docker build -t generic-app:${BUILD_NUMBER} .
            docker tag generic-app:${BUILD_NUMBER} ${ECR_REPO}:${BUILD_NUMBER}
          """
        }
      }
    }

    stage('Scan Docker Image (Trivy)') {
      when {
        expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
      }
      steps {
        script {
          echo "🔍 Scanning Docker image..."
          sh '''
            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
              aquasec/trivy:latest image \
              --exit-code 0 --no-progress --format json generic-app:${BUILD_NUMBER} > scan.json || true
          '''
          def vulnCount = sh(script: "cat scan.json | jq '.Results[0].Vulnerabilities | length' 2>/dev/null || echo 0", returnStdout: true).trim()
          echo "🧮 Vulnerabilities found: ${vulnCount}"
          env.VULN_COUNT = vulnCount
        }
      }
    }

    stage('Evaluate Scan Results') {
      when {
        expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
      }
      steps {
        script {
          def score = (env.VULN_COUNT ?: '0').toInteger()
          if (score >= 5) {
            echo "❌ Too many vulnerabilities (${score} >= 5). Aborting push."
            currentBuild.result = 'ABORTED'
            error("Build stopped due to high vulnerability count.")
          } else {
            echo "✅ Vulnerability score acceptable (${score} < 5). Proceeding to push."
          }
        }
      }
    }

    stage('Push to AWS ECR') {
      when {
        expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
      }
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'ecr-creds']]) {
          script {
            echo "🔐 Logging in to AWS ECR..."
            sh """
              aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}
              docker push ${ECR_REPO}:${BUILD_NUMBER}
            """
          }
        }
      }
    }

  }

  post {
    always {
      echo "Pipeline completed with status: ${currentBuild.result ?: 'SUCCESS'}"
      archiveArtifacts artifacts: 'scan.json', allowEmptyArchive: true
      sh 'docker image rm generic-app:${BUILD_NUMBER} || true'
      sh "docker image rm ${ECR_REPO}:${BUILD_NUMBER} || true"
    }
    success {
      echo "🎉 Image pushed successfully to AWS ECR!"
    }
    aborted {
      echo "⚠️ Pipeline aborted due to merge or vulnerability issues."
    }
    failure {
      echo "❌ Pipeline failed."
    }
  }
}
