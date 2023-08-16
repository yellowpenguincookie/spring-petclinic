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

    stage('CodeDeploy') {
      when {
          expression {
              return env.dockerizingResult ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/
          }
      }
      steps {
          script {
              try {
                  // appspec.yaml 생성 및 s3에 업로드
                  createAppspecAndUpload()
                  
                  def cmd = """
                    aws deploy create-deployment \
                    --application-name ${JOB_NAME} \
                    --deployment-config-name CodeDeployDefault.ECSAllAtOnce \
                    --deployment-group-name ${DEPLOYMENT_GROUP} \
                    --s3-location bucket=${S3_BUCKET},key=${JOB_NAME}/${DEPLOYMENT_GROUP}/appspec.yaml,bundleType=YAML | jq '.deploymentId' -r
                  """
      
                  def deploymentId = withAWS(credentials:"project04-key", region: 'ap-northeast-2') {
                      return executeAwsCliByReturn(cmd)
                  }
                  
      
                  cmd = "aws deploy get-deployment --deployment-id ${deploymentId} | jq '.deploymentInfo.status' -r"
                  def result = ""
                  timeout(unit: 'SECONDS', time: 600) {
                      while ("${result}" != "Succeeded") {
                          if ("${result}" == "Failed") {
                              exit 1
                          }
                          result = withAWS(credentials:"project04-key", region: 'ap-northeast-2') {
                              return executeAwsCliByReturn(cmd)
                          }
                          print("${result}")
                          sleep(15)
                      }
                  }
      
              } catch(Exception e) {
                  print(e)
                  cleanWs()
                  currentBuild.result = 'FAILURE'
              } finally {
                  cleanWs()
              }
          }
      }
    }


    
  
  }
}
