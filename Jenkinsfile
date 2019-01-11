pipeline {
  agent none
  stages {
    stage('build') {
      agent any
      // Set buildname to App Version from previous step
      steps {
        // add inject nonprod and prod config from environment
        sh '''
          echo "HELLO"
        '''
      }
    }
    stage('nonprod-slack') {
      agent any
        steps {    
          sh '''
            echo "deployment succeeded"
          '''   
          }
        } 
    stage('unstash-again'){
      agent any
        steps {
          sh '''
            echo "unstashing again"
          '''
        }
    }
    stage('goprod?') {
        agent none
        when {
          expression { env.BRANCH_NAME == 'master'}
        }
        steps {
          script {
            env.APP_VERSION = currentBuild.displayName
            env.GO_TO_PROD = input message: 'User input required', parameters: [choice(name: 'Go to prod?', choices: 'no\nyes', description: "Choose 'yes' if you want to deploy ${APP_VERSION} to PROD")]
            if (env.GO_TO_PROD == 'yes'){
              echo "Deploying to PROD...."
            }
          }
        }
      }
    stage('prod deploy') {
        agent any
        when {
          expression { env.GO_TO_PROD == 'yes'}
        }
        steps {
          //assume the prod-deployer rule.. 
          script {
            env.APP_VERSION = currentBuild.displayName
          }
          sh '''#!/bin/bash
            echo "Prod deploy"

          '''
        }
      }
    }
  post {
    failure {
      node('master') {
        slackSend "Build Failed - ${env.JOB_NAME} (<${env.BUILD_URL}|Open>)"
      }
    }
  }
  environment {
    ecsRepo = '484081072950.dkr.ecr.us-east-1.amazonaws.com/vermilion-webapplication'
    ecsService = 'vermilion-webapplication'
    ecsTaskFamily = 'vermilion-webapplication'
    ecsClusterName = 'vermilion_ecs_cluster'
    nonprodAlbUrl = 'https://nonprod.elseviertransitiontopractice.com'
    prodAlbUrl = 'https://www.elseviertransitiontopractice.com'
    nonprodSlackColor = '#3AB26F'
    prodSlackColor = '#347EE3'
    region = 'us-east-1'
    }
  }