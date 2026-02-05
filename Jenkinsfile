pipeline {
    agent any

    environment {
        SONARQUBE_ENV = 'sonar_integration'
        SONAR_PROJECT_KEY = 'Management-Carpooling-Services'
        SONAR_PROJECT_NAME = 'Management-Carpooling-Services'
        MAVEN_HOME = tool 'Maven'
        DOCKER_IMAGE = 'yassiramraoui/management-carpooling-services'
        DOCKER_TAG = 'latest'
    }

    stages {

        stage('Clone Repository') {
            steps {
                node { // Ajout du node ici
                    echo "📥 Cloning repository..."
                    checkout scm
                }
            }
        }

        stage('Compile Project') {
            steps {
                node {
                    echo "🏗️ Compiling the code..."
                    sh "${MAVEN_HOME}/bin/mvn clean compile"
                }
            }
        }

        stage('Run Unit Tests') {
            steps {
                node {
                    echo "🧪 Running tests..."
                    sh "${MAVEN_HOME}/bin/mvn test"
                }
            }
            post {
                always {
                    node {
                        echo "📊 Publishing test results..."
                        junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                    }
                }
                success {
                    echo "✅ All tests passed!"
                }
                failure {
                    echo "❌ Tests failed! Check the test reports."
                }
            }
        }

        stage('Package Application') {
            steps {
                node {
                    echo "📦 Packaging the application..."
                    sh "${MAVEN_HOME}/bin/mvn package -DskipTests"
                }
            }
            post {
                success {
                    node {
                        archiveArtifacts artifacts: 'target/*.jar, target/*.war', fingerprint: true
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                node {
                    echo "🔍 Running SonarQube analysis..."
                    withSonarQubeEnv('sonar_integration') {
                        sh """
                            ${MAVEN_HOME}/bin/mvn sonar:sonar \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.projectName=${SONAR_PROJECT_NAME} \
                            -Dsonar.sources=src/main/java \
                            -Dsonar.tests=src/test/java \
                            -Dsonar.java.binaries=target/classes \
                            -Dsonar.java.test.binaries=target/test-classes \
                            -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                            -Dsonar.junit.reportPaths=target/surefire-reports
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                node {
                    echo "🚦 Waiting for SonarQube Quality Gate..."
                    timeout(time: 5, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: false
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                node {
                    echo "🐳 Building Docker image..."
                    sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                }
            }
        }

        stage('Docker Push') {
            steps {
                node {
                    echo "🚀 Pushing Docker image to Docker Hub..."
                    withCredentials([usernamePassword(credentialsId: 'DockerHub', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
                        sh "docker push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    }
                }
            }
        }

        stage('Docker Smoke Test') {
            steps {
                node {
                    echo "🧪 Running container smoke test..."
                    sh "docker run -d --rm -p 7070:7070 --name mcs_smoke ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    sh "sleep 5 && docker logs mcs_smoke | tail -n 50 || true"
                    sh "docker rm -f mcs_smoke || true"
                }
            }
        }
    }

    post {
        always {
            node {
                echo "🧹 Cleaning workspace..."
                cleanWs()
            }
        }
        success {
            echo "✅ Pipeline finished successfully!"
        }
        failure {
            echo "❌ Pipeline failed. Check logs for errors."
        }
    }
}
