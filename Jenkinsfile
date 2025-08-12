pipeline {
  agent any

  environment {
    ECR_REPO = ''
    ECS_CLUSTER = 'my-ecs-cluster'
    SERVICE_X = 'my-ecs-service-x'
    SERVICE_Y = 'my-ecs-service-y'
    TASK_DEF_TEMPLATE = 'ecs-task-def.json'
    TASK_DEF_FILE = 'ecs-task-def-updated.json'
    IMAGE_TAG = ''
    SECRET_ID = 'my/secret/id'            // <-- update to your secret name or ARN
  }

  parameters {
    choice(name: 'AWS_ACCOUNT', choices: ['dev', 'stage', 'prod'], description: 'Select AWS Account')
    choice(name: 'AWS_REGION', choices: ['us-east-1', 'us-west-2', 'eu-west-1'], description: 'Select AWS Region')
    choice(name: 'ENVIRONMENT', choices: ['x', 'y'], description: 'Select environment')
    string(name: 'AWS_CREDS', defaultValue: '', description: 'Optional Jenkins AWS credentialId (leave blank to use instance profile)')
  }

  stages {
    stage('Set AWS Config & ECR Repo') {
      steps {
        script {
          if (params.AWS_ACCOUNT == 'dev') {
            env.ECR_REPO = '12345.dkr.ecr.us-east-1.amazonaws.com/my-app'
          } else if (params.AWS_ACCOUNT == 'stage') {
            env.ECR_REPO = '23456.dkr.ecr.us-east-1.amazonaws.com/my-app'
          } else if (params.AWS_ACCOUNT == 'prod') {
            env.ECR_REPO = '55555.dkr.ecr.us-east-1.amazonaws.com/my-app'
          }
          echo "ECR Repo set to ${env.ECR_REPO}"
        }
      }
    }

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build & Push Docker Image') {
      steps {
        script {
          env.IMAGE_TAG = "${env.BUILD_NUMBER}"
          def loginAndPush = """
            aws ecr get-login-password --region ${params.AWS_REGION} | docker login --username AWS --password-stdin ${env.ECR_REPO}
            docker build -t ${env.ECR_REPO}:${env.IMAGE_TAG} .
            docker push ${env.ECR_REPO}:${env.IMAGE_TAG}
          """
          if (params.AWS_CREDS?.trim()) {
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: params.AWS_CREDS]]) {
              sh loginAndPush
            }
          } else {
            sh loginAndPush
          }
        }
      }
    }

    stage('Fetch RDS Secret (Secrets Manager)') {
      steps {
        script {
          // fetch secret (SecretString expected as JSON)
          def getSecretCmd = "aws secretsmanager get-secret-value --region ${params.AWS_REGION} --secret-id ${env.SECRET_ID} --query SecretString --output text"
          def secretJson
          if (params.AWS_CREDS?.trim()) {
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: params.AWS_CREDS]]) {
              secretJson = sh(script: getSecretCmd, returnStdout: true).trim()
            }
          } else {
            secretJson = sh(script: getSecretCmd, returnStdout: true).trim()
          }

          if (!secretJson) {
            error("Failed to read secret ${env.SECRET_ID}")
          }

          def db = readJSON text: secretJson
          // minimal validation
          if (!db.username || !db.password) {
            error("Secret ${env.SECRET_ID} must include 'username' and 'password' fields")
          }

          // write temp env file (short-lived)
          writeFile file: 'rds_secrets.env', text: """DB_USERNAME=${db.username}
DB_PASSWORD=${db.password}
DB_HOST=${db.host ?: ''}
DB_PORT=${db.port ?: ''}
DB_NAME=${db.dbname ?: ''}
"""
        }
      }
    }

    stage('Update ECS Task Definition') {
      steps {
        script {
          // load env list and pass to withEnv as array of VAR=VALUE strings
          def envList = readFile('rds_secrets.env').split('\n').findAll { it?.trim() }
          // JQ command: ensure containerDefinitions[0].environment exists, update image, then append env vars
          def jqCmd = """jq '
    .containerDefinitions[0] |= ( . + { environment: ( .environment // [] ) } ) |
    .containerDefinitions[0].image = "${env.ECR_REPO}:${env.IMAGE_TAG}" |
    .containerDefinitions[0].environment += [
      {"name":"DB_USERNAME","value":"'"'$DB_USERNAME'"'"},
      {"name":"DB_PASSWORD","value":"'"'$DB_PASSWORD'"'"}
    ]
    ' ${TASK_DEF_TEMPLATE} > ${TASK_DEF_FILE}"""
          withEnv(envList) {
            sh(script: jqCmd)
            // register task definition and capture the returned taskDefinitionArn
            def registerCmd = "aws ecs register-task-definition --region ${params.AWS_REGION} --cli-input-json file://${TASK_DEF_FILE} --output json"
            def regOut
            if (params.AWS_CREDS?.trim()) {
              withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: params.AWS_CREDS]]) {
                regOut = sh(script: registerCmd, returnStdout: true).trim()
              }
            } else {
              regOut = sh(script: registerCmd, returnStdout: true).trim()
            }
            def regJson = readJSON text: regOut
            def newTaskDefArn = regJson.taskDefinition.taskDefinitionArn
            def newFamily = regJson.taskDefinition.family
            def newRevision = regJson.taskDefinition.revision
            echo "Registered task-definition ${newFamily}:${newRevision} -> ${newTaskDefArn}"
            // Save these for later stages
            writeFile file: 'registered_task_info.txt', text: "${newTaskDefArn}\n${newFamily}\n${newRevision}\n"
          }
        }
      }
      post {
        always {
          // delete temp env file
          sh 'rm -f rds_secrets.env || true'
        }
      }
    }

    stage('Approval Before Deploy') {
      steps {
        input message: "Approve deployment to ${params.AWS_ACCOUNT} (${params.AWS_REGION})? Image: ${env.ECR_REPO}:${env.IMAGE_TAG}", ok: "Deploy"
      }
    }

    stage('Deploy/Update ECS Service') {
      steps {
        script {
          def serviceName = params.ENVIRONMENT == 'x' ? env.SERVICE_X : env.SERVICE_Y
          def registeredInfo = readFile('registered_task_info.txt').split('\n')
          def newTaskArn = registeredInfo[0].trim()

          def updateCmd = "aws ecs update-service --region ${params.AWS_REGION} --cluster ${env.ECS_CLUSTER} --service ${serviceName} --task-definition ${newTaskArn}"
          if (params.AWS_CREDS?.trim()) {
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: params.AWS_CREDS]]) {
              sh updateCmd
            }
          } else {
            sh updateCmd
          }
        }
      }
    }

    stage('Rollback (on failure)') {
      when { expression { currentBuild.currentResult == 'FAILURE' } }
      steps {
        script {
          def serviceName = params.ENVIRONMENT == 'x' ? env.SERVICE_X : env.SERVICE_Y
          // describe current running service to get current taskDefinition ARN
          def descCmd = "aws ecs describe-services --region ${params.AWS_REGION} --cluster ${env.ECS_CLUSTER} --services ${serviceName} --query 'services[0].taskDefinition' --output text"
          def currTaskDefArn = ''
          if (params.AWS_CREDS?.trim()) {
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: params.AWS_CREDS]]) {
              currTaskDefArn = sh(script: descCmd, returnStdout: true).trim()
            }
          } else {
            currTaskDefArn = sh(script: descCmd, returnStdout: true).trim()
          }
          if (!currTaskDefArn || currTaskDefArn == 'None') {
            echo "Cannot determine current taskDefinition ARN for rollback."
            return
          }

          // get family name
          def family = currTaskDefArn.tokenize('/').last().tokenize(':')[0]
          // list last 2 task-definitions for the family (sorted desc)
          def listCmd = "aws ecs list-task-definitions --region ${params.AWS_REGION} --family-prefix ${family} --sort DESC --query 'taskDefinitionArns[:2]' --output json"
          def listOut
          if (params.AWS_CREDS?.trim()) {
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: params.AWS_CREDS]]) {
              listOut = sh(script: listCmd, returnStdout: true).trim()
            }
          } else {
            listOut = sh(script: listCmd, returnStdout: true).trim()
          }
          def arr = readJSON text: listOut
          if (arr.size() < 2) {
            echo "No previous task definition found to rollback to."
          } else {
            def prev = arr[1]
            echo "Rolling back ${serviceName} to ${prev}"
            def rollbackCmd = "aws ecs update-service --region ${params.AWS_REGION} --cluster ${env.ECS_CLUSTER} --service ${serviceName} --task-definition ${prev}"
            if (params.AWS_CREDS?.trim()) {
              withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: params.AWS_CREDS]]) {
                sh rollbackCmd
              }
            } else {
              sh rollbackCmd
            }
          }
        }
      }
    }
  }

  post {
    always {
      cleanWs()
    }
  }
}