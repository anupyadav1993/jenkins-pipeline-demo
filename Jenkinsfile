#!groovy

/*
Global Variables Defined
*/

pipeline {
  agent {
    label 'master'
  }
  stages {
    stage('Checkout') {
      steps {
        deleteDir()
        checkout scm
      }
    }
    stage('Build') {
      steps {
            script{
                    def dockerImage = docker.image('maven:3.3.3-jdk-8')
                    dockerImage.pull();
                    dockerImage.inside{
                        sh '''mvn clean package'''
                    }
                }
          }
    }
    stage('Run Tests'){
      when {
        expression {
          env.BRANCH_NAME != 'master'
        }
      }
      parallel {
        stage('Integration Test') {
          steps {
            script{
                    def dockerImage = docker.image('maven:3.3.3-jdk-8')
                    dockerImage.pull();
                    dockerImage.inside{
                        sh '''mvn clean verify test package'''
                    }
                }
          }
          post {
            success {
              slackSend (color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' Build and Integration Tested(${env.BUILD_URL})")
            }
            failure {
              slackSend (color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' Quality Test Done (${env.BUILD_URL})")
            }
          }
        }
        stage('Quality Test') {
          steps{
              echo "Quality Testing..."
            }
          post {
            success {
              slackSend (color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' Build and Tested(${env.BUILD_URL})")
            }
            failure {
              slackSend (color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' Build and Test (${env.BUILD_URL})")
            }
          } 
        }
      }      
    }
    stage('Publish Test Results') {
      when {
        expression {
          env.BRANCH_NAME != 'master'
        }
      }
      steps {
        junit 'gameoflife-core/build/test-results/*.xml'
        publishHTML(target:[allowMissing: false, alwaysLinkToLastBuild: false, keepAll: true, reportDir: 'gameoflife-core/build/reports/tests', reportFiles: 'index.html', reportTitles: 'Report', reportName: 'Report'])
      }
      post {
        success {
          slackSend (color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' Report URL: (${env.BUILD_URL}Report)")
        }
        failure {
          slackSend (color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' Publish Test Result Failed (${env.BUILD_URL})")
        }
      }
    }
    stage('Approve') {
      when {
        expression {
          env.BRANCH_NAME == 'master'
        }
      }
      steps {
        timeout(time: 7, unit: 'HOURS') {
          input 'Do you want to proceed?'
        }
      }
    }
    stage('Deploy') {
      when {
        expression {
          env.CHANGE_ID == null
        }
      }
      steps {
        sh '''set -e
AWS="/usr/local/bin/aws"
DOCKER_LOGIN=`$AWS ecr get-login --no-include-email --region us-east-1`
IMAGE_NAME="jenkins-pipeline-demo"
${DOCKER_LOGIN}
echo "Creating docker image..."
cd $WORKSPACE/gameoflife-web
git fetch
git_tag=$(git tag --sort version:refname| tail -1)
docker_image=$IMAGE_NAME:$git_tag
sed -i 's/custom/'"$git_tag"'/g' Dockerfile
cat Dockerfile
docker build -t $docker_image .
if [ $? -eq 0 ]
then
   echo "Successfully image created"
else
   echo "Error in creating image"
   exit 1
fi
docker tag $docker_image 940345575562.dkr.ecr.us-east-1.amazonaws.com/$docker_image
docker push 940345575562.dkr.ecr.us-east-1.amazonaws.com/$docker_image
if [ $? -eq 0 ]
then
    echo "Successfully image tagged and pushed to repository"
    echo 940345575562.dkr.ecr.us-east-1.amazonaws.com/$docker_image > $WORKSPACE/image_id
    cat $WORKSPACE/image_id
else
    echo "Error in tagging/pushing image"
    exit 1
fi
TASK_FAMILY="jenkins-pipeline-demo-td"
SERVICE_NAME="jenkins-pipeline-demo-svc"
NEW_DOCKER_IMAGE=`cat $WORKSPACE/image_id`
CLUSTER_NAME="jenkins-pipeline-demo"
OLD_TASK_DEF=$($AWS ecs describe-task-definition --task-definition $TASK_FAMILY --output json --region us-east-1)
NEW_TASK_DEF=$(echo $OLD_TASK_DEF | jq --arg NDI $NEW_DOCKER_IMAGE \'.taskDefinition.containerDefinitions[0].image=$NDI\')
FINAL_TASK=$(echo $NEW_TASK_DEF | jq \'.taskDefinition|{family: .family, networkMode: .networkMode, volumes: .volumes, containerDefinitions: .containerDefinitions, placementConstraints: .placementConstraints}\')
$AWS ecs register-task-definition --family $TASK_FAMILY --cli-input-json "$(echo $FINAL_TASK)"  --region us-east-1
if [ $? -eq 0 ]
then
    echo "New task has been registered"
else
    echo "Error in task registration"
exit 1
fi
echo "Now deploying new version..."
$AWS ecs update-service --service $SERVICE_NAME  --desired-count 1 --task-definition $TASK_FAMILY --cluster $CLUSTER_NAME  --region us-east-1
                    '''
                    script{
                      sh (script: "git tag --sort version:refname| tail -1 > git_tag_file", returnStdout: true)
                      
                      env.git_tag_env = readFile 'git_tag_file'
                    }
      }
      post {
        success {
          slackSend (color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' Application Deployed (demo-application.tothenew.net/${env.git_tag_env})")
        }
        failure {
          slackSend (color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' Application Deployment Failed (${env.BUILD_URL})")
        }
    }

    }
  }
}
