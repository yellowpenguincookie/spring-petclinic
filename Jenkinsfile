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
    
    stage('CodeDeploy'){
      steps {
        sh("""
          git config --global user.name "yellowpenguincookie"
          git config --global user.email "yurijjjung@gmail.com"
          git checkout -B master
        """)
  
        script{
          previousTAG = sh(script: 'echo `expr ${BUILD_NUMBER} - 1`', returnStdout: true).trim()
        }
  
        withCredentials([usernamePassword(credentialsId: 'github-signin', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
          sh("""
          #!/usr/bin/env bash
          git config --local credential.helper "!f() { echo username=\\$GIT_USERNAME; echo password=\\$GIT_PASSWORD; }; f"
          echo ${previrousTAG}
          
          git add eks/deployment.yaml
          git status
          git commit -m "update the image tag"
          git push origin HEAD:master
        """)
        }
      }
    }

  
  }
}
