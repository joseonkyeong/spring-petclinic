pipeline {
    agent any

    tools {
        jdk "jdk17"
        maven "M3"
    }
    environment {
        AWS_CREDENTIAL_NAME = "AWSCredentials"
        REGION = "ap-northeast-2"
        DOCKER_IMAGE_NAME="aws11-spring-petclinic"
        ECR_REPOSITORY = "257307634175.dkr.ecr.ap-northeast-2.amazonaws.com"
        ECR_DOCKER_IMAGE = "${ECR_REPOSITORY}/${DOCKER_IMAGE_NAME}"
    }
    
    stages {
        stage('Git Clone') {
            steps {
                echo 'Git Clone'
                git url: 'https://github.com/joseonkyeong/spring-petclinic.git',
                branch: 'wavefront'
            }
            post {
                success {
                    echo 'success clone project'
                }
                failure {
                    error 'fail clone project' // exit pipeline
                }
            }
        }        
        stage ('mvn Build') {
            steps {
                sh 'mvn -Dmaven.test.failure.ignore=true install' 
            }
            post {
                success {
                    junit '**/target/surefire-reports/TEST-*.xml' 
                }
            }
        }        
        stage ('Docker Build') {
            steps {
                dir("${env.WORKSPACE}") {
                    sh """
                      docker build -t $ECR_DOCKER_IMAGE:$BUILD_NUMBER .
                      docker tag $ECR_DOCKER_IMAGE:$BUILD_NUMBER $ECR_DOCKER_IMAGE:latest
                    """
                }
            }
         }
         stage('Push Docker Image') {
             steps {
                 echo "Push Docker Image to ECR"
                 script{
                     // cleanup current user docker credentials
                     sh 'rm -f ~/.dockercfg ~/.docker/config.json || true' 
                     docker.withRegistry("https://${ECR_REPOSITORY}", "ecr:${REGION}:${AWS_CREDENTIAL_NAME}") {
                         docker.image("${ECR_DOCKER_IMAGE}:${BUILD_NUMBER}").push()
                         docker.image("${ECR_DOCKER_IMAGE}:latest").push()
                     }
                    
                 }
             }
             post {
                 success {
                     echo "Push Docker Image success!"
                 }
             }
         }
         stage('Clean Up Docker Images on Jenkins Server') {
             steps {
                 echo 'Cleaning up unused Docker images on Jenkins server'

                 // Clean up unused Docker images, including those created within the last hour
                 sh "docker image prune -f --all --filter \"until=1h\""
             }
         }
         stage('Upload to S3') {
             steps {
                 echo "Upload to S3"
                 dir("${env.WORKSPACE}") {
                     sh 'zip -r deploy.zip ./deploy appspec.yml'
                     withAWS(region:"${REGION}", credentials:"${AWS_CREDENTIAL_NAME}"){
                       s3Upload(file:"deploy.zip", bucket:"aws11-codedeploy-bucket")
                     } 
                     sh 'rm -rf ./deploy.zip'                 
                 }        
             }
         }
         stage('Codedeploy Workload') {
             steps {
                echo "create Codedeploy group"   
                 sh '''
                     aws deploy create-deployment-group \
                     --application-name aws11-code-deploy \
                     --auto-scaling-groups aws11-asg \
                     --deployment-group-name aws00-code-deploy-${BUILD_NUMBER} \
                     --deployment-config-name CodeDeployDefault.OneAtATime \
                     --service-role-arn arn:aws:iam::257307634175:role/aws11-codedeploy-service-role
                     '''
                 echo "Codedeploy Workload"   
                 sh '''
                     aws deploy create-deployment --application-name aws00-code-deploy \
                     --deployment-config-name CodeDeployDefault.OneAtATime \
                     --deployment-group-name aws00-code-deploy-${BUILD_NUMBER} \
                     --s3-location bucket=aws00-codedeploy-bucket,bundleType=zip,key=deploy.zip
                     '''
                     sleep(10) // sleep 10s
             }
         }            
    }
}