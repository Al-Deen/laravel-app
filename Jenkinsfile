pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Al-Deen/laravel-app.git'
            }
        }
        stage('Test') {
            steps {
                echo 'Running tests...'
                // এখানে PHPUnit টেস্ট রান করানোর কমান্ড থাকবে
            }
        }
        stage('Build & Deploy') {
            steps {
                echo 'Building docker image and deploying...'
            }
        }
    }
}
