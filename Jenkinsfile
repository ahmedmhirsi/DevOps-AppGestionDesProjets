
pipeline {
    agent any

    tools {
        jdk 'JDK21'
    }

    environment {
        DB_URL = 'jdbc:mysql://127.0.0.1:3306/test_db?createDatabaseIfNotExist=true'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'mysql-backend-db',
                    usernameVariable: 'DB_USERNAME',
                    passwordVariable: 'DB_PASSWORD'
                )]) {
                    dir('backend') {
                        sh 'chmod +x mvnw'
                        sh './mvnw clean test'
                    }
                }
            }
            post {
                always {
                    junit 'backend/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Package') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'mysql-backend-db',
                    usernameVariable: 'DB_USERNAME',
                    passwordVariable: 'DB_PASSWORD'
                )]) {
                    dir('backend') {
                        sh './mvnw -DskipTests package'
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'CI Pipeline completed successfully!'
        }
        failure {
            echo 'CI Pipeline failed. Check the stage logs.'
        }
    }
}
