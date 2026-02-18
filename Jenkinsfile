pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    DOCKERHUB_USER = "gigagabunia"
    IMAGE_NAME     = "cicd-pipeline-app"
    DOCKER_REGISTRY_URL = "https://index.docker.io/v1/"
    DOCKER_CREDS_ID     = "dockerhub-credentials"
    NODE_TOOL = "node18"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'git rev-parse --short HEAD > .git_commit'
        sh 'echo "Branch: ${BRANCH_NAME} | Commit: $(cat .git_commit)"'
      }
    }

    stage('Set branch config') {
      steps {
        script {
          if (env.BRANCH_NAME == 'main') {
            env.DEPLOY_PORT = '3000'
          } else if (env.BRANCH_NAME == 'dev') {
            env.DEPLOY_PORT = '3001'
          } else {
            env.DEPLOY_PORT = '0'
          }

          env.GIT_SHA = readFile('.git_commit').trim()
          env.IMAGE_TAG   = "${env.BRANCH_NAME}-${env.GIT_SHA}"
          env.LATEST_TAG  = "${env.BRANCH_NAME}-latest"

          env.FULL_IMAGE   = "${env.DOCKERHUB_USER}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"
          env.LATEST_IMAGE = "${env.DOCKERHUB_USER}/${env.IMAGE_NAME}:${env.LATEST_TAG}"

          echo "BRANCH=${env.BRANCH_NAME}"
          echo "PORT=${env.DEPLOY_PORT}"
          echo "IMAGE=${env.FULL_IMAGE}"
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
            npm run build
          '''
        }
      }
    }

    stage('Test') {
      steps {
        nodejs(nodeJSInstallationName: "${env.NODE_TOOL}") {
          sh '''
            set -euxo pipefail
            npm test -- --watchAll=false
          '''
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          env.DOCKER_BUILD_ARGS = "--build-arg BRANCH_NAME=${env.BRANCH_NAME} --build-arg PORT=${env.DEPLOY_PORT} ."
          docker.build("${env.FULL_IMAGE}", "${env.DOCKER_BUILD_ARGS}")
          sh "docker tag ${env.FULL_IMAGE} ${env.LATEST_IMAGE}"
        }
      }
    }

    stage('Push') {
      steps {
        script {
          docker.withRegistry("${env.DOCKER_REGISTRY_URL}", "${env.DOCKER_CREDS_ID}") {
            docker.image("${env.FULL_IMAGE}").push()
            docker.image("${env.LATEST_IMAGE}").push()
          }
        }
      }
    }
  }

  post {
  always {
    sh '''
      echo "CICD finished for branch=${BRANCH_NAME}"
      docker images | head -n 25 || true
    '''
  }
  cleanup {
    cleanWs(deleteDirs: true, disableDeferredWipeout: true)
    }
  }
}
