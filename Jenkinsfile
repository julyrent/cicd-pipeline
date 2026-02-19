pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    NODE_TOOL = "node18"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        echo "Branch: ${env.BRANCH_NAME}"
      }
    }

    stage('Set branch config') {
      steps {
        script {
          if (env.BRANCH_NAME == 'main') {
            env.IMAGE_NAME = 'nodemain:v1.0'
            env.HOST_PORT  = '3000'
            env.CTR_PORT   = '3000'
            env.CONTAINER_NAME = 'nodemain'
          } else if (env.BRANCH_NAME == 'dev') {
            env.IMAGE_NAME = 'nodedev:v1.0'
            env.HOST_PORT  = '3001'
            env.CTR_PORT   = '3000'
            env.CONTAINER_NAME = 'nodedev'
          } else {
            error("Unsupported branch: ${env.BRANCH_NAME}. Only main/dev allowed.")
          }

          echo "IMAGE=${env.IMAGE_NAME}"
          echo "PORT_MAP=${env.HOST_PORT}:${env.CTR_PORT}"
          echo "CONTAINER=${env.CONTAINER_NAME}"
        }
      }
    }

    stage('Build') {
      steps {
        nodejs(nodeJSInstallationName: "${env.NODE_TOOL}") {
          sh '''
            set -euxo pipefail
            node -v
            npm -v
            npm install
          '''
        }
      }
    }

    stage('Test') {
      steps {
        nodejs(nodeJSInstallationName: "${env.NODE_TOOL}") {
          sh '''
            set -euxo pipefail
            npm test
          '''
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''
          set -euxo pipefail
          docker build -t "$IMAGE_NAME" .
        '''
      }
    }

    stage('Deploy') {
      steps {
        sh '''
          set -euxo pipefail
          docker rm -f "$CONTAINER_NAME" || true
          docker run -d --name "$CONTAINER_NAME" \
            --expose "$HOST_PORT" \
            -p "$HOST_PORT":"$CTR_PORT" \
            "$IMAGE_NAME"
          docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}"
          echo "App should be on: http://localhost:$HOST_PORT"
        '''
      }
    }
  }

  post {
    always {
      sh 'docker ps -a || true'
    }
  }
}
