pipeline {
  agent any

  environment {
    TASK_DEF_TEMPLATE = 'definitions/ecs-task-def.json'
    TASK_DEF_FILE     = 'definitions/ecs-task-def-updated.json'
    CLUSTER_PREFIX    = 'my-ecscluster-'    // infra-created cluster name prefix
    SERVICE_NAME      = 'my-app-service'    // ECS service name (same across x/y clusters)
    DDB_TABLE         = 'DeployState'       // DynamoDB table to record lastKnownGood
  }

  parameters {
    choice(name: 'AWS_ACCOUNT', choices: ['dev','prod'], description: 'env (infra account)')
    choice(name: 'AWS_REGION', choices: ['ap-south-1','us-east-1'], description: 'AWS region')
    choice(name: 'ACTION', choices: ['create','update'], description: 'create or update service')
    string(name: 'AWS_CREDS', defaultValue: '', description: 'Optional Jenkins AWS credentialsId; leave blank to use instance profile')
    string(name: 'ECR_REPO_URI', defaultValue: '', description: 'Optional: full ECR repo URI override')
    string(name: 'HEALTH_URL', defaultValue: '', description: 'ALB/health URL for smoke test (if empty, smoke test is skipped)')
    booleanParam(name: 'WAIT_FOR_STEADY', defaultValue: true, description: 'Wait for ECS service to stabilize before smoke test')
  }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Discover ECS Cluster') {
      steps {
        script {
          def cmd = "aws ecs list-clusters --region ${params.AWS_REGION} --query \"clusterArns[?contains(@,'${env.CLUSTER_PREFIX}')]\" --output text"
          def clusters = runAws(cmd)
          if (!clusters) { error "No cluster found with prefix ${env.CLUSTER_PREFIX} in ${params.AWS_REGION}" }
          // choose last (latest) cluster when multiple exist
          def clusterArn = clusters.tokenize().last()
          env.CLUSTER_ARN = clusterArn
          echo "Selected cluster: ${env.CLUSTER_ARN}"
        }
      }
    }

    stage('Build & Push Image') {
      steps {
        script {
          env.IMAGE_TAG = "${env.BUILD_NUMBER}"
          if (!params.ECR_REPO_URI?.trim()) {
            // default repo mapping; replace account IDs with your values or pass ECR_REPO_URI
            env.ECR_REPO = (params.AWS_ACCOUNT == 'prod') ? "55555.dkr.ecr.${params.AWS_REGION}.amazonaws.com/my-app-repo" : "12345.dkr.ecr.${params.AWS_REGION}.amazonaws.com/my-app-repo"
          } else {
            env.ECR_REPO = params.ECR_REPO_URI
          }

          def buildPush = """
            set -e
            aws ecr get-login-password --region ${params.AWS_REGION} | docker login --username AWS --password-stdin ${env.ECR_REPO}
            docker build -t ${env.ECR_REPO}:${env.IMAGE_TAG} .
            docker push ${env.ECR_REPO}:${env.IMAGE_TAG}
          """
          if (params.AWS_CREDS?.trim()) {
            withCredentials([[$class:'AmazonWebServicesCredentialsBinding', credentialsId: params.AWS_CREDS]]) {
              sh buildPush
            }
          } else {
            sh buildPush
          }
          echo "Pushed ${env.ECR_REPO}:${env.IMAGE_TAG}"
        }
      }
    }

    stage('Register Task Definition (image only)') {
      steps {
        script {
          sh "jq '.containerDefinitions[0].image = \"${env.ECR_REPO}:${env.IMAGE_TAG}\"' ${TASK_DEF_TEMPLATE} > ${TASK_DEF_FILE}"

          def registerCmd = "aws ecs register-task-definition --region ${params.AWS_REGION} --cli-input-json file://${TASK_DEF_FILE} --output json"
          def regOut = runAws(registerCmd)
          def regJson = readJSON text: regOut
          env.NEW_TASK_ARN = regJson.taskDefinition.taskDefinitionArn
          echo "Registered: ${env.NEW_TASK_ARN}"
        }
      }
    }

    stage('Create or Update Service') {
      steps {
        script {
          // For 'create' action, create service (requires network config)
          if (params.ACTION == 'create') {
            // NOTE: update subnets/security groups to match your infra or pass via pipeline params
            def createCmd = """
            aws ecs create-service \
              --region ${params.AWS_REGION} \
              --cluster ${env.CLUSTER_ARN} \
              --service-name ${env.SERVICE_NAME} \
              --task-definition ${env.NEW_TASK_ARN} \
              --desired-count 1 \
              --launch-type FARGATE \
              --network-configuration 'awsvpcConfiguration={subnets=[subnet-xxxx],securityGroups=[sg-xxxx],assignPublicIp=ENABLED}'
            """
            runAws(createCmd)
          } else {
            // update path
            def currentTaskDef = runAws("aws ecs describe-services --region ${params.AWS_REGION} --cluster ${env.CLUSTER_ARN} --services ${env.SERVICE_NAME} --query 'services[0].taskDefinition' --output text") ?: ''
            env.OLD_TASK_ARN = currentTaskDef
            echo "Current taskDefinition: ${env.OLD_TASK_ARN}"
            def updateCmd = "aws ecs update-service --region ${params.AWS_REGION} --cluster ${env.CLUSTER_ARN} --service ${env.SERVICE_NAME} --task-definition ${env.NEW_TASK_ARN}"
            runAws(updateCmd)
          }
        }
      }
    }

    stage('Wait / Smoke Test / Persist') {
      steps {
        script {
          if (params.WAIT_FOR_STEADY) {
            echo "Waiting for service to become stable..."
            runAws("aws ecs wait services-stable --region ${params.AWS_REGION} --cluster ${env.CLUSTER_ARN} --services ${env.SERVICE_NAME}")
            echo "Service stable. Running smoke test (if HEALTH_URL provided)..."

            if (params.HEALTH_URL?.trim()) {
              // run smoke test script against ALB/endpoint
              try {
                sh "HEALTH_URL='${params.HEALTH_URL}' scripts/smoke_test.sh"
                echo "Smoke test passed — persisting lastKnownGood"
                putLastKnownGood(params.AWS_REGION, env.SERVICE_NAME, env.NEW_TASK_ARN)
              } catch (err) {
                echo "Smoke test failed — attempting rollback"
                def lastKnown = getLastKnownGood(params.AWS_REGION, env.SERVICE_NAME)
                if (!lastKnown) {
                  error "No lastKnownGood found to rollback to. Manual intervention required."
                } else {
                  echo "Rolling back to ${lastKnown}"
                  runAws("aws ecs update-service --region ${params.AWS_REGION} --cluster ${env.CLUSTER_ARN} --service ${env.SERVICE_NAME} --task-definition ${lastKnown}")
                  runAws("aws ecs wait services-stable --region ${params.AWS_REGION} --cluster ${env.CLUSTER_ARN} --services ${env.SERVICE_NAME}")
                  error "Rolled back to lastKnownGood (${lastKnown})."
                }
              }
            } else {
              echo "No HEALTH_URL set — skipping smoke test. Persists lastKnownGood optimistically."
              putLastKnownGood(params.AWS_REGION, env.SERVICE_NAME, env.NEW_TASK_ARN)
            }
          } else {
            echo "Skipping wait & smoke test. Persisting new task as lastKnownGood (optimistic)."
            putLastKnownGood(params.AWS_REGION, env.SERVICE_NAME, env.NEW_TASK_ARN)
          }
        }
      }
    }
  }

  post {
    always { cleanWs() }
  }
}

// ------------------- helper functions -------------------
def runAws(cmd) {
  if (params.AWS_CREDS?.trim()) {
    withCredentials([[$class:'AmazonWebServicesCredentialsBinding', credentialsId: params.AWS_CREDS]]) {
      return sh(script: cmd, returnStdout: true).trim()
    }
  } else {
    return sh(script: cmd, returnStdout: true).trim()
  }
}

def putLastKnownGood(region, service, arn) {
  def item = "{\"ServiceName\":{\"S\":\"${service}\"},\"LastKnownGood\":{\"S\":\"${arn}\"}}"
  runAws("aws dynamodb put-item --region ${region} --table-name ${env.DDB_TABLE} --item '${item}'")
  echo "Updated lastKnownGood: ${arn}"
}

def getLastKnownGood(region, service) {
  try {
    def cmd = "aws dynamodb get-item --region ${region} --table-name ${env.DDB_TABLE} --key '{\"ServiceName\":{\"S\":\"${service}\"}}' --query 'Item.LastKnownGood.S' --output text"
    def out = runAws(cmd)
    if (!out || out == "None") { return null }
    return out.trim()
  } catch (err) {
    return null
  }
}