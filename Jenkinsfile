
stage 'Build' 
    node {
        deleteDir()
        git 'https://github.com/wakaleo/game-of-life.git' 
        withEnv(["PATH+MAVEN=${tool 'm3'}/bin"]) {
            sh '''
            mvn -B -Dmaven.test.failure.ignore=true clean package
            '''
        }
        stash excludes: 'target/', includes: '**', name: 'source'
    }
stage 'Test'
        node {
            unstash 'source' 
            withEnv(["PATH+MAVEN=${tool 'm3'}/bin"]) {
                sh "mvn verify" 
            }
            unstash 'source' 
            withEnv(["PATH+MAVEN=${tool 'm3'}/bin"]) {
                sh '''
                set -x
                pwd
                mvn sonar:sonar'''
            }
        }
        
stage 'Approve'
    timeout(time: 7, unit: 'DAYS') {
    input('Do you want to proceed?')
}

stage 'Build'
    node{
        sh'''
        set -e
        AWS="/usr/local/bin/aws"
        DOCKER_LOGIN=`$AWS ecr get-login --no-include-email --region us-east-1`
        IMAGE_NAME="jenkins-pipeline-demo"
        ${DOCKER_LOGIN}
        
        echo "Building docker image..."
        cd $WORKSPACE/gameoflife-web
        git_tag=$(git rev-parse --short HEAD)
        
        docker_image=$IMAGE_NAME:$git_tag
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
        NEW_TASK_DEF=$(echo $OLD_TASK_DEF | jq --arg NDI $NEW_DOCKER_IMAGE '.taskDefinition.containerDefinitions[0].image=$NDI')
        FINAL_TASK=$(echo $NEW_TASK_DEF | jq '.taskDefinition|{family: .family, networkMode: .networkMode, volumes: .volumes, containerDefinitions: .containerDefinitions, placementConstraints: .placementConstraints}')
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
}

