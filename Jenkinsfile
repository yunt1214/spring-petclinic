pipeline {
    agent any
    // 도구 설정
    tools {
        jdk "jdk17"
        maven "M3"
    }
    // 환경 설정
    environment {
        AWS_CREDENTIAL_NAME = "AWSCredentials"
        REGION = "ap-northeast-2"
        DOCKER_IMAGE_NAME = "std03-spring-petclinic"
        ECR_REPOSITORY = "257307634175.dkr.ecr.ap-northeast-2.amazonaws.com"
        ECR_DOCKER_IMAGE = "${ECR_REPOSITORY}/${DOCKER_IMAGE_NAME}"
    }
    
    stages {
        stage('Git clone') {
            steps {
                echo 'Git clone'
                git url: 'https://github.com/yunt1214/spring-petclinic.git',
                    branch: 'efficient-webjars', credentialsId: 'GitCredentials'
            }
            post {
                success {
                    echo 'Git Clone Success!!'
                }
                failure {
                    echo 'Git Clone Fail'
                }
            }
        }
    
        stage('Maven Build') {
            steps {
                echo 'Maven Build'
                sh 'mvn -Dmaven.test.failure.ignore=true clean package'
            }        
            post {
                success {
                    junit 'target/surefire-reports/**/*.xml'
                }
            }
        }

        stage ('Docker Image Build') {
            steps {
                echo 'Docker Image Build'
                dir("${env.WORKSPACE}") {
                    sh """
                        docker build -t $ECR_DOCKER_IMAGE:$BUILD_NUMBER .
                        docker tag $ECR_DOCKER_IMAGE:$BUILD_NUMBER $ECR_DOCKER_IMAGE:latest
                    """
                }
            }
        }

        stage ('Push Docker Image') {
            steps {
                echo "Push Docker Image to ECR"
                script {
                    sh 'rm -f ~/.dockercfg ~/.docker/config.json || true'
                    docker.withRegistry("https://${ECR_REPOSITORY}", "ecr:${REGION}:${AWS_CREDENTIAL_NAME}") {
                        docker.image("${ECR_DOCKER_IMAGE}:${BUILD_NUMBER}").push()
                        docker.image("${ECR_DOCKER_IMAGE}:latest").push()
                    }
                }
            }
        }

        stage ('Clean Up Docker Images on Jenkins Server') {
            steps {
                echo 'Cleaning up unused Docker images on Jenkins server'
                sh "docker image prune -a -f"
            }
        }

        stage ('Upload to S3') {
            steps {
                echo 'Upload to S3'
                dir("${env.WORKSPACE}") {
                   sh 'zip -r deploy.zip ./deploy appspec.yml'
                   withAWS(region:"${REGION}", credentials:"${AWS_CREDENTIAL_NAME}") {
                       s3Upload(file:"deploy.zip", bucket:"std03-codedeploy-bucket")
                   }
                   sh 'rm -rf ./deploy.zip'
                }
            }
        }

        stage ('Codedeploy Workload') {
            steps {
                echo "create Codedeploy group"
                sh '''
                    aws deploy create-deployment-group \
                    --application-name std03-code-deploy \
                    --auto-scaling-groups std03-asg-target \
                    --deployment-group-name std03-code-deploy-${BUILD_NUMBER} \
                    --deployment-config-name CodeDeployDefault.OneAtATime \
                    --service-role-arn arn:aws:iam::257307634175:role/std03-codedeploy-service-role
                    '''
                echo "Codedeploy Workload"
                sh '''
                    aws deploy create-deployment --application-name std03-spring-petclinic \
                    --deployment-config-name CodeDeployDefault.OneAtATime \
                    --deployment-group-name std03-code-deploy-${BUILD_NUMBER} \
                    --s3-location bucket=std03-codedeploy-bucket,bundleType=zip,key=deploy.zip
                    '''
                    sleep(10) // sleep 10s
            }
        }
        
    }
}
