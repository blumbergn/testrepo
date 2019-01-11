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

            #Get the temp creds for prod account

            temp_role=$(aws sts assume-role --role-arn "arn:aws:iam::371534284618:role/Jenkins-ECS-UpdateService-UpdateTask" --role-session-name "ProdBuild")
            export AWS_ACCESS_KEY_ID=$(echo $temp_role | jq .Credentials.AccessKeyId | xargs)
            export AWS_SECRET_ACCESS_KEY=$(echo $temp_role | jq .Credentials.SecretAccessKey | xargs)
            export AWS_SESSION_TOKEN=$(echo $temp_role | jq .Credentials.SessionToken | xargs)
            export AWS_DEFAULT_REGION=us-east-1

            #Get the ids for the currently deployed tasks

            num_deploys=$(aws ecs describe-services --cluster=${ecsClusterName} --services=${ecsTaskFamily} --region ${region} | jq '.services[].deployments | length')
            echo "Number of deploys: $num_deploys"
            origArns=()
            x=0
            numOrigArns=$(aws ecs list-tasks --cluster=${ecsClusterName} --service-name ${ecsTaskFamily} --region ${region} | jq '.taskArns | length')
            echo "Number of orig arns: $numOrigArns"

            echo "Current TaskIDs"

            while [ $x -lt $numOrigArns ]
              do
                origArns[$x]=`aws ecs list-tasks --cluster=${ecsClusterName} --service-name ${ecsTaskFamily} --region ${region} | jq ".taskArns["$x"]"`
                echo ${origArns[$x]}
                x=$((x+1))
              done

            echo -e '\n\n\n\n'

            # Register the new task and update the service

            envsubst < taskdef.prod.json > taskdef.prod.${APP_VERSION}.json
            aws ecs register-task-definition --family ${ecsTaskFamily} --cli-input-json file://taskdef.prod.${APP_VERSION}.json --region us-east-1 | tee -a ecs_registration_prod.json
            export PROD_TASK_REVISION=$(cat ecs_registration_prod.json | jq \'.taskDefinition.revision\')
            #update the service with the new task definition and desired count
            aws ecs update-service --cluster ${ecsClusterName} --service ${ecsService} --task-definition ${ecsTaskFamily}:${PROD_TASK_REVISION} --region us-east-1 --desired-count 2          

            # Check deployment until complete or fail

            deploy="active"
            while [ $deploy == "active" ]
            do
              sleep 2
              num_deploys=`aws ecs describe-services --cluster=${ecsClusterName} --services=${ecsTaskFamily} --region ${region} | jq '.services[].deployments | length'`
              echo "Number of deployments: $num_deploys"

              # For each loop through get the count of task ids in the service
              y=0
              numNewArns=`aws ecs list-tasks --cluster=${ecsClusterName} --service-name ${ecsTaskFamily} --region ${region} | jq '.taskArns | length'`
              echo "Number of arns: $numNewArns"

              while [ $y -lt $numNewArns ]
              do
                # put the task ids in the service in an array
                newArns=()
                newArns[$y]=`aws ecs list-tasks --cluster=${ecsClusterName} --service-name ${ecsTaskFamily} --region ${region} | jq ".taskArns["$y"]"`
                echo ${newArns[$y]}
                y=$((y+1))
              done

              if [ $num_deploys -eq 1 ]
              then
                # comfirm original task ids are no longer in the service
                # might want to add code to make sure number of task ids is greater than 0

                echo "Time to check if the deployment is ok"
                echo "Number of arns: $numNewArns"
                for origarn in "${origArns[@]}"
                  do
                    if [[ ${newArns[*]} =~  $origarn ]]
                    then
                      echo "old arn still exists"
                      echo "build failed"
                      deploy="failed"
                      exit 1
                    else
                      echo "deployment sucessful"
                      deploy="complete"
                    fi
                  done

              elif [ $num_deploys -gt 1 ]
              then
                echo "Current number of deployments: $num_deploys"
                echo "Still deploying"
              else
                echo "Current number of deployments: 0"
                echo "Build failed"
                exit 1
              fi
            done     
          '''
        //tell slack the promotion went well
        sh '''
          export PROD_TASK_REVISION=$(cat ecs_registration_prod.json | jq \'.taskDefinition.revision\')
          envsubst < slack.prod.json > slackpayload.json
          curl --request POST --url https://hooks.slack.com/services/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 --header 'content-type: application/json' --data @slackpayload.json
        '''
        script {
          archiveArtifacts('*taskdef.*prod.*.json')
          }
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
    npmEpmToken = credentials('NPM_EPM_TOKEN')
    npmToken = credentials('NPM_TOKEN')
    }
  }