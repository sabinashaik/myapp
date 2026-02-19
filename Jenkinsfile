pipeline {

  agent any

  triggers {
    githubPush()
  }

  tools {
    maven "MAVEN3"
    jdk "JDK21"
  }

  environment {
    APP_NAME  = "myapp"
    NOTIFY_TO = "sabinashaik228@gmail.com"
  }

  stages {
    stage('Auto Detect Branch & Map ENV') {
      steps {
        script {

          // Detect branch automatically
          env.BRANCH_NAME = env.GIT_BRANCH.tokenize('/').last()

          // Map branch to environment
          if (env.BRANCH_NAME == "br1") {
              env.ENV_NAME = "dev"
          }
          else if (env.BRANCH_NAME == "br2") {
              env.ENV_NAME = "qa"
          }
          else if (env.BRANCH_NAME == "main") {
              env.ENV_NAME = "prod"
          }
          else {
              error "Branch ${env.BRANCH_NAME} not mapped to any environment"
          }

          echo "Detected Branch: ${env.BRANCH_NAME}"
          echo "Mapped Environment: ${env.ENV_NAME}"

          // Load environment config
          def props = readProperties file: 'jenkins/build.properties'

          env.TOMCAT_URL   = props["${env.ENV_NAME}.tomcat.url"]
          env.TOMCAT_CREDS = props["${env.ENV_NAME}.tomcat.creds"]

          if (!env.TOMCAT_URL) {
              error "Environment configuration missing"
          }
        }
      }
    }

    stage('Build + Test + Package') {
      steps {
        sh 'mvn -U clean test package'
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('SonarQube') {
          sh 'mvn sonar:sonar'
        }
      }
    }

    stage('Upload to Nexus') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'nexus-creds',
          usernameVariable: 'NEXUS_USER',
          passwordVariable: 'NEXUS_PASS'
        )]) {

          sh '''
            mvn deploy -DskipTests \
              -Dnexus.username=$NEXUS_USER \
              -Dnexus.password=$NEXUS_PASS
          '''
        }
      }
    }

    stage('Deploy to Tomcat') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: "${env.TOMCAT_CREDS}",
          usernameVariable: 'TOMCAT_USER',
          passwordVariable: 'TOMCAT_PASS'
        )]) {

          sh """
            echo "Deploying to ${env.ENV_NAME}..."
            curl --fail -u \$TOMCAT_USER:\$TOMCAT_PASS \
              -T target/${APP_NAME}.war \
              "${env.TOMCAT_URL}/deploy?path=/${APP_NAME}&update=true"
          """
        }
      }
    }
  }

  post {

    success {
      emailext(
        subject: "Pipeline  - ${env.BRANCH_NAME}",
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
        subject: "Pipeline  - ${env.BRANCH_NAME}",
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
