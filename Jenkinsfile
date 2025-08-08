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
    }

    parameters {
        choice(name: 'AWS_ACCOUNT', choices: ['dev', 'stage', 'prod'], description: 'Select AWS Account')
        choice(name: 'AWS_REGION', choices: ['us-east-1', 'us-west-2', 'eu-west-1'], description: 'Select AWS Region')
        choice(name: 'ENVIRONMENT', choices: ['x', 'y'], description: 'Select environment (blue/green)')
    }

    stages {
        stage('Set AWS Credentials & ECR Repo') {
            steps {
                script {
                    if (params.AWS_ACCOUNT == 'dev') {
                        env.AWS_CREDS = 'aws-dev-creds'
                        env.ECR_REPO = '12345.dkr.ecr.us-east-1.amazonaws.com/my-app'
                    } else if (params.AWS_ACCOUNT == 'stage') {
                        env.AWS_CREDS = 'aws-stage-creds'
                        env.ECR_REPO = '23456.dkr.ecr.us-east-1.amazonaws.com/my-app'
                    } else if (params.AWS_ACCOUNT == 'prod') {
                        env.AWS_CREDS = 'aws-prod-creds'
                        env.ECR_REPO = '55555.dkr.ecr.us-east-1.amazonaws.com/my-app'
                    }
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
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: env.AWS_CREDS]]) {
                        sh """
                            aws ecr get-login-password --region ${params.AWS_REGION} | docker login --username AWS --password-stdin $ECR_REPO
                            docker build -t $ECR_REPO:$IMAGE_TAG .
                            docker push $ECR_REPO:$IMAGE_TAG
                        """
                    }
                }
            }
        }

        stage('Fetch RDS Secrets') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: env.AWS_CREDS]]) {
                    script {
                        def dbSecret = sh(
                            script: "aws secretsmanager get-secret-value --region ${params.AWS_REGION} --secret-id my/secret/id --query SecretString --output text",
                            returnStdout: true
                        ).trim()
                        def db = readJSON text: dbSecret
                        writeFile file: 'rds_secrets.env', text: """
DB_USERNAME=${db.username}
DB_PASSWORD=${db.password}
"""
                    }
                }
            }
        }

        stage('Update ECS Task Definition') {
            steps {
                withEnv(readFile('rds_secrets.env').split('\n').findAll { it }) {
                    sh """
                        jq '.containerDefinitions[0].image = "$ECR_REPO:$IMAGE_TAG" | .containerDefinitions[0].environment += [{"name":"DB_USERNAME","value":"'$DB_USERNAME'"},{"name":"DB_PASSWORD","value":"'$DB_PASSWORD'"}]' $TASK_DEF_TEMPLATE > $TASK_DEF_FILE
                        aws ecs register-task-definition --region ${params.AWS_REGION} --cli-input-json file://$TASK_DEF_FILE
                    """
                }
            }
        }

        stage('Approval Before Deploy') {
            steps {
                input message: "Approve deployment to ECS in ${params.AWS_ACCOUNT} (${params.AWS_REGION})?", ok: "Deploy"
            }
        }

        stage('Deploy/Update ECS Service') {
            steps {
                script {
                    def serviceName = params.ENVIRONMENT == 'x' ? env.SERVICE_X : env.SERVICE_Y
                    catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                        sh """
                            aws ecs update-service --region ${params.AWS_REGION} --cluster $ECS_CLUSTER --service $serviceName --task-definition $(jq -r .family $TASK_DEF_FILE)
                        """
                    }
                }
            }
        }

        stage('Rollback (on failure)') {
            when { expression { currentBuild.currentResult == 'FAILURE' } }
            steps {
                script {
                    def serviceName = params.ENVIRONMENT == 'x' ? env.SERVICE_X : env.SERVICE_Y
                    def currTaskDef = sh(
                        script: "aws ecs describe-services --region ${params.AWS_REGION} --cluster $ECS_CLUSTER --services $serviceName --query 'services[0].taskDefinition' --output text",
                        returnStdout: true
                    ).trim()
                    def family = currTaskDef.tokenize('/')[1].tokenize(':')[0]
                    def prevTaskDef = sh(
                        script: "aws ecs list-task-definitions --region ${params.AWS_REGION} --family-prefix ${family} --sort DESC --query 'taskDefinitionArns[1]' --output text",
                        returnStdout: true
                    ).trim()
                    sh """
                        aws ecs update-service --region ${params.AWS_REGION} --cluster $ECS_CLUSTER --service $serviceName --task-definition $prevTaskDef
                    """
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
