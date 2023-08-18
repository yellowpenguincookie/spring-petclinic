pipeline {
  agent any 
  tools {
    maven 'M3'
    jdk 'JDK11'
  }
  environment {
    AWS_CREDENTIALS_NAME = "AWSCredentials"
    REGION = "ap-northeast-2"
    DOCKER_IMAGE_NAME = "project04-spring-petclinic"
    DOCKER_TAG = "1.0"
    ECR_REPOSITORY = "257307634175.dkr.ecr.ap-northeast-2.amazonaws.com"
    ECR_DOCKER_IMAGE = "${ECR_REPOSITORY}/${DOCKER_IMAGE_NAME}"
    ECR_DOCKER_TAG = "${DOCKER_TAG}" 
  }
  
  stages {
    stage('Git Clone') {
      steps {
        git url: 'https://github.com/yellowpenguincookie/spring-petclinic.git', branch: 'efficient-webjars', credentialsId: 'gitCredentials'
      }
    }
    stage('mvn build') {
      steps {
        sh 'mvn -Dmaven.test.failure.ignore=true install'
      }
      post {
        success {
          junit '**/target/surefire-reports/TEST-*.xml'
        }
      }
    }
    stage('Docker Image Build') {
      steps {
        dir("${env.WORKSPACE}") {
          sh 'docker build -t ${DOCKER_IMAGE_NAME}:${DOCKER_TAG} .'
        }
      }
    }
    stage('Push Docker Image') {
      steps {
        script {
          sh 'rm -f ~/.dockercfg ~/.docker/config.json || true' 

          docker.withRegistry("https://${ECR_REPOSITORY}", "ecr:${REGION}:${AWS_CREDENTIALS_NAME}") {
            docker.image("${DOCKER_IMAGE_NAME}:${DOCKER_TAG}").push()
          }
        }
      }
    }
    stage('Upload to S3') {
      steps {
        dir("${env.WORKSPACE}") {
          sh 'zip -r deploy-1.0.zip ./scripts appspec.yml'
          sh 'aws s3 cp --region ap-northeast-2 --acl private ./deploy-1.0.zip s3://project04-terraform-state'
          sh 'rm -rf ./deploy-1.0.zip'
        }
      }
    }
    stage('Codedeploy') {
      steps {
        script {
          sh "aws deploy create-application --application-name project04-production-in-place --compute-platform Server"
        }
      }
    }    
    stage('Codedeploy group') {
      steps {
        script {
          sh "aws deploy create-deployment-group" + 
             "--application-name project04-production-in-place" +
             "--auto-scaling-groups project04-target-group" +
             "--deployment-group-name project04-production-in-place" +
             "--service-role-arn arn:aws:iam::257307634175:role/project04-code-deploy-service-role"
        }
      }
    }    
    stage('Deploy to CodeDeploy') {
      steps {
        script {        
          sh "aws deploy create-deployment" +
             "--application-name project04-production-in-place" +
             "--s3-location bucket=<project04-terraform-state>,bundleType=zip,key=deploy-1.0" +
             "--deployment-group-name project04-production-in-place" +
             "--deployment-config-name CodeDeployDefault.OneAtATime" +
             "--target-instances autoScalingGroups=<project04-target-group>"
          }
       }
     }

    
  }
}
