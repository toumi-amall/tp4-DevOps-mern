pipeline {
  agent any

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Build images') {
      steps { sh 'docker compose build' }
    }
    stage('Deploy') {
      steps {
        sh 'docker compose down || true'
        sh 'docker compose up -d'
      }
    }
    stage('Test') {
      steps {
        sh 'sleep 10'
        sh 'docker exec server wget -qO- http://localhost:9000/api/health'
      }
    }
  }

  post {
    failure { sh 'docker compose logs || true' }
  }
}