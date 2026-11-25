pipeline {
  agent any

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Build images') {
      steps { bat 'docker compose build' }
    }
    stage('Deploy') {
      steps {
        bat 'docker compose down || exit 0'
        bat 'docker compose up -d'
      }
    }
    stage('Test') {
      steps {
        bat 'ping -n 16 127.0.0.1 > nul'
        bat 'docker exec server wget -qO- http://localhost:9000/api/health'
      }
    }
  }

  post {
    failure { bat 'docker compose logs || exit 0' }
  }
}