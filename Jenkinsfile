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

        stage('Deploy') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'mysql-backend-db',
                    usernameVariable: 'DB_USERNAME',
                    passwordVariable: 'DB_PASSWORD'
                )]) {
                    sh '''
                        DEPLOY_DIR="/var/lib/jenkins/backend-deploy"
                        PID_FILE="$DEPLOY_DIR/backend.pid"
                        LOG_FILE="$DEPLOY_DIR/backend-app.log"
                        JAR_PATH="$DEPLOY_DIR/backend-1.0.0.jar"

                        echo "=== Continuous Deployment: Local WSL Target (Port 8082) ==="

                        # 1. Stop existing backend instance if running
                        if [ -f "$PID_FILE" ]; then
                            OLD_PID=$(cat "$PID_FILE" 2>/dev/null || true)
                            if [ -n "$OLD_PID" ] && kill -0 "$OLD_PID" 2>/dev/null; then
                                echo "Stopping existing backend process (PID: $OLD_PID)..."
                                kill "$OLD_PID" || true
                                for i in $(seq 1 10); do
                                    if ! kill -0 "$OLD_PID" 2>/dev/null; then
                                        echo "Existing process $OLD_PID stopped gracefully."
                                        break
                                    fi
                                    sleep 1
                                done
                                if kill -0 "$OLD_PID" 2>/dev/null; then
                                    echo "Force-stopping process $OLD_PID..."
                                    kill -9 "$OLD_PID" 2>/dev/null || true
                                fi
                            fi
                            rm -f "$PID_FILE"
                        fi

                        # 2. Deploy fresh artifact
                        mkdir -p "$DEPLOY_DIR"
                        cp backend/target/backend-1.0.0.jar "$JAR_PATH"

                        # 3. Start application in background with JENKINS_NODE_COOKIE=dontKillMe
                        echo "Starting backend-1.0.0.jar on port 8082..."
                        export SERVER_PORT=8082
                        export SPRING_DATASOURCE_URL="$DB_URL"
                        export SPRING_DATASOURCE_USERNAME="$DB_USERNAME"
                        export SPRING_DATASOURCE_PASSWORD="$DB_PASSWORD"
                        export JENKINS_NODE_COOKIE=dontKillMe

                        nohup java -Dserver.port=8082 -jar "$JAR_PATH" > "$LOG_FILE" 2>&1 &
                        NEW_PID=$!
                        echo "$NEW_PID" > "$PID_FILE"
                        echo "Backend started with PID $NEW_PID."

                        # 4. Health verification on real endpoint /equipe/all
                        echo "Waiting for backend to initialize and connect to MySQL..."
                        READY=0
                        for i in $(seq 1 30); do
                            if ! kill -0 "$NEW_PID" 2>/dev/null; then
                                echo "ERROR: Backend process $NEW_PID died unexpectedly!"
                                tail -n 40 "$LOG_FILE"
                                exit 1
                            fi
                            HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8082/equipe/all || true)
                            if [ "$HTTP_CODE" = "200" ]; then
                                echo "SUCCESS: Backend responded HTTP 200 on http://localhost:8082/equipe/all after $((i*2))s!"
                                READY=1
                                break
                            fi
                            sleep 2
                        done

                        if [ "$READY" -ne 1 ]; then
                            echo "ERROR: Backend did not become ready within 60s. Log tail:"
                            tail -n 40 "$LOG_FILE"
                            exit 1
                        fi
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD Pipeline completed successfully! Backend JAR archived and deployed on port 8082.'
        }
        failure {
            echo 'CI/CD Pipeline failed. Check the stage logs and test reports.'
        }
    }
}
