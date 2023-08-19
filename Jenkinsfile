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
    APPLICATION_NAME = "project04-production-in-place"
    DEPLOYMENT_GROUP_NAME = "project04-production-in-place"
    AUTO_SCALING_GROUP_NAME = "project04-target-group"
    SERVICE_ROLE_ARN = "arn:aws:iam::257307634175:role/project04-code-deploy-service-role"
    DEPLOYMENT_CONFIG_NAME = "CodeDeployDefault.OneAtATime"
    S3_BUCKET = "project04-terraform-state"
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
          sh 'aws s3 cp --region ap-northeast-2 ./deploy-1.0.zip s3://project04-terraform-state'
          sh 'rm -rf ./deploy-1.0.zip'
        }
      }
    }

    stage('CodeDeploy Application') {
      steps {
        script {    
          sh 'aws deploy delete-application --application-name "${APPLICATION_NAME}"'
          sh 'aws deploy create-application --application-name "${APPLICATION_NAME}" --compute-platform Server'
          }
       }
     }

    stage('CodeDeploy Deployment Group') {
      steps {
        script {    
          sh 'aws deploy create-deployment-group --application-name "${APPLICATION_NAME}" \
          --deployment-group-name "${DEPLOYMENT_GROUP_NAME}" \
          --auto-scaling-groups "${AUTO_SCALING_GROUP_NAME}" \
          --service-role-arn "${SERVICE_ROLE_ARN}" \
          --deployment-config-name "${DEPLOYMENT_CONFIG_NAME}"'
          }
       }
     }
    
    stage('Deploy to CodeDeploy') {
      steps {
        script {
          sh 'aws deploy create-deployment \
             --application-name "${APPLICATION_NAME}" \
             --s3-location bucket=project04-terraform-state,bundleType=zip,key=deploy-1.0 \
             --deployment-group-name "${DEPLOYMENT_GROUP_NAME}" \
             --deployment-config-name "${DEPLOYMENT_CONFIG_NAME}" \
             --target-instances autoScalingGroups="${AUTO_SCALING_GROUP_NAME}"'
        }
      }
    }


    
  }
}
