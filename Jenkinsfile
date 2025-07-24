pipeline {
    agent any

    tools {
        maven 'Maven 3.8.8'
        jdk 'JDK 21'
        nodejs 'Node 20.19.0'
    }

    environment {
        REGISTRY_URL                  = 'docker.io'
        BACKEND_IMAGE                 = 'fathyafi/be-app3:latest'
        OPENSHIFT_PROJECT             = 'fathya-app'
        SONAR_QUBE_SERVER_URL         = 'https://sonar3am.42n.fun/'
        SONAR_QUBE_PROJECT_KEY        = 'be-app-sq'
        SONAR_QUBE_PROJECT_NAME       = 'Project SonarQube Backend'
    }

    stages {
        stage('Checkout') {
            steps {
                deleteDir()
                dir('backend') {
                    git branch: 'main', url: 'https://github.com/fathyafi/api-main.git'
                }
                echo "Repository checked out successfully."
            }
        }

        stage('Build Aplikasi (Maven)') {
            steps {
                dir('backend') {
                    script {
                        withMaven(maven: 'Maven 3.8.8') {
                            sh 'mvn clean install -DskipTests'
                            echo "Application built successfully."
                        }
                    }
                }
            }
        }

        stage('Unit Test') {
            steps {
                dir('backend') {
                    script {
                        sh '''
                            chmod +x ./mvnw
                            mkdir -p src/main/resources
                        '''
                        withCredentials([file(credentialsId: 'app-properties-backend', variable: 'APP_PROPS')]) {
                            sh 'cp "$APP_PROPS" src/main/resources/application.properties'
                        }
                        sh './mvnw clean package -DskipTests'
                        withMaven(maven: 'Maven 3.8.8') {
                            sh './mvnw test'
                        }
                    }
                }
            }
        }

        stage('SAST with SonarQube') {
            steps {
                dir('backend') {
                    script {
                        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                            sh """
                                mvn clean compile sonar:sonar \\
                                -Dsonar.projectKey=${SONAR_QUBE_PROJECT_KEY} \\
                                -Dsonar.projectName="${SONAR_QUBE_PROJECT_NAME}" \\
                                -Dsonar.host.url=${SONAR_QUBE_SERVER_URL} \\
                                -Dsonar.token=${SONAR_TOKEN}
                            """
                            echo "SonarQube analysis completed."
                        }
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                dir('backend'){
                    script {
                        sh "docker build --no-cache -t ${BACKEND_IMAGE} ."
                        echo "Backend image built."
                    }
                }
            }
        }

        stage('Push Docker Image to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh """
                            docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}
                            docker push ${BACKEND_IMAGE}
                        """
                        echo "Docker image pushed to Docker Hub."
                    }
                }
            }
        }

        stage('Deploy to OpenShift (GCP)') {
            steps {
                script {
                    sh "oc project ${OPENSHIFT_PROJECT}"
                    dir('backend/openshift') {
                        sh "oc apply -f deployment.yml"
                        sh "oc apply -f service.yml"
                        sh "oc apply -f route.yml"
                        sh "oc rollout restart deployment/backend-app1"
                    }
                    echo "Application deployed to OpenShift."
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline finished successfully!"
        }
        failure {
            echo "❌ Pipeline failed! Check the logs for details."
        }
    }
}