pipeline {
    agent any

    tools {
        jdk 'JDK21'
    }

    triggers {
        // Scheduled build: runs every weekday (Mon-Fri) around 2:00 AM
        cron('H 2 * * 1-5')
        // Automatically triggers on GitHub webhook push event
        githubPush()
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

        stage('Clean') {
            steps {
                dir('backend') {
                    sh 'chmod +x mvnw'
                    sh './mvnw clean'
                }
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
                        sh './mvnw test'
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
                        // Package without re-running tests since they already passed in Test stage
                        sh './mvnw -DskipTests package'
                    }
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                archiveArtifacts artifacts: 'backend/target/backend-1.0.0.jar', fingerprint: true, onlyIfSuccessful: true
            }
        }

        // =====================================================================
        // Requirement F: Continuous Deployment (Local WSL Target)
        // Note: Spring Boot defaults to port 8080 which conflicts with Jenkins.
        // When enabled, it runs on port 8082.
        // Uncomment and activate only after target confirmation & approval.
        // =====================================================================
        /*
        stage('Deploy (Local)') {
            when {
                environment name: 'ENABLE_LOCAL_DEPLOY', value: 'true'
            }
            steps {
                echo 'Deploying backend application on port 8082...'
                sh 'nohup java -Dserver.port=8082 -jar backend/target/backend-1.0.0.jar > backend-app.log 2>&1 &'
            }
        }
        */
    }

    post {
        success {
            echo 'CI/CD Pipeline completed successfully! Backend JAR artifact archived.'
        }
        failure {
            echo 'CI/CD Pipeline failed. Check the stage logs and test reports.'
        }
    }
}
