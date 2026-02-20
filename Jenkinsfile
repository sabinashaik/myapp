pipeline {
    agent any

    tools {
        maven 'MAVEN3'
        jdk 'JDK21'
    }

    environment {
        APP_NAME  = "myapp"
    NOTIFY_TO = "sabinashaik228@gmail.com"

    }

    stages {

        stage('Auto Detect Branch and  Map Environment') {
            steps {
                script {

                    // Detect branch dynamically
                    env.BRANCH_NAME = env.GIT_BRANCH.tokenize('/').last()
                    echo "Detected Branch: ${env.BRANCH_NAME}"

                    // Load properties file
                    def props = readProperties file: 'jenkins/build.properties'

                    // Get environment name from branch mapping
                    env.ENV_NAME = props["${env.BRANCH_NAME}.env"]

                    if (!env.ENV_NAME) {
                        error "No environment mapping found for branch ${env.BRANCH_NAME}"
                    }

                    echo "Mapped Environment: ${env.ENV_NAME}"

                    // Load environment specific configurations
                    env.TOMCAT_URL   = props["${env.ENV_NAME}.tomcat.url"]
                    env.TOMCAT_CREDS = props["${env.ENV_NAME}.tomcat.creds"]

                    if (!env.TOMCAT_URL || !env.TOMCAT_CREDS) {
                        error "Tomcat configuration missing for environment ${env.ENV_NAME}"
                    }

                    echo "Tomcat URL: ${env.TOMCAT_URL}"
                }
            }
        }

        stage('Build + Test + Package') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONAR_SERVER}") {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Upload Artifact to Nexus') {
            steps {
                sh 'mvn deploy'
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                deploy adapters: [
                    tomcat9(
                        credentialsId: "${env.TOMCAT_CREDS}",
                        path: '',
                        url: "${env.TOMCAT_URL}"
                    )
                ],
                contextPath: 'myapp',
                war: '**/target/*.war'
            }
        }
    }

  post {

    success {
      emailext(
        subject: "Pipeline",
        body: """SUCCESS

Branch: ${env.BRANCH_NAME}
Environment: ${env.ENV_NAME}
Build Number: ${env.BUILD_NUMBER}
URL: ${env.BUILD_URL}
""",
        to: "${env.NOTIFY_TO}"
      )
    }

    failure {
      emailext(
        subject: "Pipeline",
        body: """FAILURE

Branch: ${env.BRANCH_NAME}
Environment: ${env.ENV_NAME}
Build Number: ${env.BUILD_NUMBER}
URL: ${env.BUILD_URL}
""",
        to: "${env.NOTIFY_TO}"
      )
    }

    always {
      cleanWs()
    }
  }
}
