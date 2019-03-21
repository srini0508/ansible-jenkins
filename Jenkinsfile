pipeline {
    agent any
    environment {
            PROJECT_NAME = "dispatch"
            ECR_REPO_DEV = "757289378104.dkr.ecr.us-west-2.amazonaws.com/dispatch"
            ECR_REPO_NP =  "757289378104.dkr.ecr.us-west-2.amazonaws.com/dispatch"
            ECR_REPO_PROD = "619481788748.dkr.ecr.us-west-2.amazonaws.com/dispatch"
            AWS_REGION = "us-west-2"
            QA_HOSTNAME = "qa-dispatch.dma.foxinc.com"
            DV_KEY_BUCKET="dma-non-prod-us-west-2-dv-key"
            DV_KEY_BUCKET_PROD="dma-prod-us-west-2-dv-key"
    }
    stages {
      stage ('Init') {  
        steps {
           wrap([$class: 'BuildUser']) {
                   script{
                       user_name="${BUILD_USER}"
                       user_email="${BUILD_USER_EMAIL}"
                   }
               }
           }
       }
    stage('Checkout') {
           steps {
            cleanWs()
              script {
                if (env.PROJECT_ENV == 'dev') {
                    fileOperations([
                            folderCreateOperation (folderPath: "${WORKSPACE}/${PROJECT_NAME}-build"),
                            folderCreateOperation (folderPath: "${WORKSPACE}/${PROJECT_NAME}-build/source")
                          ])
            dir("${WORKSPACE}/${PROJECT_NAME}-build/source"){
            checkout([$class: 'GitSCM',
                      branches: [[name: '*/${BRANCH}']], 
                      doGenerateSubmoduleConfigurations: false,
                      extensions: [[$class: 'SubmoduleOption',
                                    disableSubmodules: false,
                                    parentCredentials: false,
                                    recursiveSubmodules: true,
                                    reference: '',
                                    trackingSubmodules: false]], 
                      submoduleCfg: [], 
                      userRemoteConfigs: [[url: 'ssh://git@lax099130vbbt01:7999/dis/dispatch.git']]])
              }
            }
            else {
                echo " This stage is not required for \${PROJECT_ENV} "
               }
            }
        }
    }
    stage('NPM'){
        steps {
          script {
             if (env.PROJECT_ENV == 'dev') {   
                dir("${WORKSPACE}/${PROJECT_NAME}-build/source"){
                 sh """
                      meteor npm install --quiet --production --no-progress --registry=https://registry.npmjs.org
                      if [ "\$(git diff package-lock.json)" != "" ]; then
                          meteor npm install --quiet --production --no-progress --registry=https://registry.npmjs.org
                      fi
                      meteor build --debug --architecture=os.linux.x86_64 --directory ${env.WORKSPACE}/${PROJECT_NAME}-build --allow-superuser
                 """
              }
           }
      else {
                echo "This stage is not for ${PROJECT_ENV} environment"
             }
          }
       }
    } 
    stage ("Docker Bundle"){
	    steps {
	      script {
          if (env.PROJECT_ENV == 'dev') {
	          dir ("${env.WORKSPACE}/${PROJECT_NAME}-build/source"){
	            fileOperations ([
                 fileCopyOperation (excludes: '', flattenFiles: false, includes: "Dockerfile", targetLocation: "${env.WORKSPACE}/${PROJECT_NAME}-build/bundle"),
                 fileCopyOperation (excludes: '', flattenFiles: false, includes: ".dockerignore", targetLocation: "${env.WORKSPACE}/${PROJECT_NAME}-build/bundle"),
	               fileCopyOperation (excludes: '', flattenFiles: false, includes: "*_settings.json", targetLocation: "${env.WORKSPACE}/${PROJECT_NAME}-build/bundle"),
		             fileCopyOperation (excludes: '', flattenFiles: false, includes: "run.sh", targetLocation: "${env.WORKSPACE}/${PROJECT_NAME}-build/bundle")
		      ])
	      }
	    }
      else {
                echo "This stage is not for ${PROJECT_ENV} environment"
          }
       }
	  }
	}
    stage("App Version") {
           steps {
             script {
            if (env.PROJECT_ENV == 'dev') {
            dir("${env.WORKSPACE}/${PROJECT_NAME}-build/bundle") {
               sh """
                 jq \'.public.appVersion="${VERSION}"\'  dev_settings.json > tmp.json
                 cat tmp.json > dev_settings.json
                 jq \'.public.appVersion="${VERSION}"\'  qa_settings.json > tmp.json
                 cat tmp.json > qa_settings.json
                 jq \'.public.appVersion="${VERSION}"\'  uat_settings.json > tmp.json
                 cat tmp.json > uat_settings.json
                 jq \'.public.appVersion="${VERSION}"\'  pr_settings.json > tmp.json
                 cat tmp.json > pr_settings.json
                 rm -rf tmp.json
                """
               }
            }
            else {
                echo "This stage is not for ${PROJECT_ENV} environment"
                }
             }
          }
       }
    stage('Build') {
        steps {
          script {
            if (env.PROJECT_ENV == 'dev') { 
               dir("${WORKSPACE}/${PROJECT_NAME}-build/bundle"){
               sh """
                 echo "Building a docker image for ${PROJECT_ENV}"
                 docker build -f Dockerfile -t ${PROJECT_NAME}_${PROJECT_ENV}_${BUILD_NUMBER}:${VERSION} .
                 """
               }
            }
            else {
                echo "This stage is not for ${PROJECT_ENV} environment"
             }
          }
        }
    }
    stage('Push') {
        steps {
          script {
            if (env.PROJECT_ENV == 'dev') {
              sh """
                echo "pushing docker image to Dev (nonprod) account"
                docker tag ${PROJECT_NAME}_${PROJECT_ENV}_${BUILD_NUMBER}:${VERSION} ${ECR_REPO_DEV}:${VERSION}_dev
                eval `aws ecr get-login --profile=nonprod --region "${AWS_REGION}" | sed 's|-e none https://||'`
                docker push ${ECR_REPO_DEV}:${VERSION}_dev
              """
          }
          else if (env.PROJECT_ENV == 'qa') { 
               dir("${WORKSPACE}/${PROJECT_NAME}-build/bundle"){
               sh """
                 echo "pushing a docker image for ${PROJECT_ENV}"
                 eval `aws ecr get-login --profile=nonprod --region "${AWS_REGION}" | sed 's|-e none https://||'`
                 docker pull ${ECR_REPO_DEV}:${VERSION}_dev
                 docker tag ${ECR_REPO_DEV}:${VERSION}_dev ${ECR_REPO_NP}:${VERSION}
                 docker tag ${ECR_REPO_DEV}:${VERSION}_dev ${ECR_REPO_NP}:latest
                 eval `aws ecr get-login --profile=nonprod --region "${AWS_REGION}" | sed 's|-e none https://||'`
                 docker push ${ECR_REPO_NP}:${VERSION}
                 docker push ${ECR_REPO_NP}:latest
                 """
               }
            }
          else if (env.PROJECT_ENV == 'prod') { 
               dir("${WORKSPACE}/${PROJECT_NAME}-build/bundle"){
               sh """
                 echo "pushing a docker image for ${PROJECT_ENV}"
                 eval `aws ecr get-login --profile=nonprod --region "${AWS_REGION}" | sed 's|-e none https://||'`
                 docker pull ${ECR_REPO_NP}:${VERSION}
                 docker tag ${ECR_REPO_NP}:${VERSION} ${ECR_REPO_PROD}:${VERSION}
                 docker tag ${ECR_REPO_NP}:${VERSION} ${ECR_REPO_PROD}:latest
                 eval `aws ecr get-login --profile=prod --region "${AWS_REGION}" | sed 's|-e none https://||'`
                 docker push ${ECR_REPO_PROD}:${VERSION}
                 docker push ${ECR_REPO_PROD}:latest
                 """
               }
            }
          else {
              echo "This stage is not for ${PROJECT_ENV} environment"
             }
          }
        }
    }
    stage('Deploy') {
        steps {
            script {
               if (env.PROJECT_ENV == 'dev') {
                   writeFile(file: "${PROJECT_NAME}${PROJECT_ENV}.sh", text:
"""
#!/bin/bash
set -e
eval `aws ecr get-login --profile=nonprod --region "${AWS_REGION}" | sed 's|-e none https://||'`
echo "Pulling docker image from dev aws ECR"
docker pull ${ECR_REPO_DEV}:${VERSION}_dev
echo "Removing ${PROJECT_NAME} containers with status created and exited "
docker ps -aq --no-trunc -f status=exited -f status=created -f name=${PROJECT_NAME}| xargs docker rm | true
echo "Stopping ${PROJECT_NAME} old container"
docker stop \$(docker ps -q -f name=${PROJECT_NAME}) | true
echo "creating ${PROJECT_NAME} new container"
docker run --env-file /home/admin/Builds/Dispatch/dev_env.list --link mongodb_3.4 -d -p 7000:7000 --name ${PROJECT_NAME}_${PROJECT_ENV}_${VERSION}_${BUILD_NUMBER} ${ECR_REPO_DEV}:${VERSION}_dev
sleep 10s
docker image prune -a -f || true         
""", encoding: "UTF-8")
             script {
                      sshPublisher(
                      continueOnError: false, failOnError: true,
                      publishers: [
                      sshPublisherDesc(
                      configName: "${PROJECT_NAME}buildserver",
                      verbose: true,
                      transfers: [
                      sshTransfer(
                      sourceFiles: "${PROJECT_NAME}${PROJECT_ENV}.sh",
                      removePrefix: "",
                      remoteDirectory: "",
                      execCommand: "bash -x ${PROJECT_NAME}${PROJECT_ENV}.sh; rm -rf ${PROJECT_NAME}${PROJECT_ENV}.sh"
                  )
                 ])
               ])
             }
            }
                else if (env.PROJECT_ENV == 'nonprod') {
                   writeFile(file: "${PROJECT_NAME}${PROJECT_ENV}.sh", text: 
"""
#!/bin/bash
set -e
aws s3 cp s3://${DV_KEY_BUCKET}/rockit_key/dispatch-keys /home/ec2-user/dispatch-keys
DISPATCH_KEYS=\$(aws kms decrypt --region us-west-2 --ciphertext-blob fileb:///home/ec2-user/dispatch-keys --output text --query Plaintext | base64 --decode)
cat << EOF > /home/ec2-user/dispatch-env
\$DISPATCH_KEYS
EOF

chown ec2-user /home/ec2-user/dispatch-env
chmod go-rwx /home/ec2-user/dispatch-env
docker ps -aq --no-trunc -f status=exited -f status=created -f name=${PROJECT_NAME} | xargs docker rm 2>/dev/null || true
docker stop \$(docker ps -q -f name=${PROJECT_NAME}) 2>/dev/null || true
\$(aws ecr get-login --no-include-email --region us-west-2)
DISPATCH_IMAGE="${ECR_REPO_NP}:${VERSION}"
echo "Pulling ${VERSION} dispatch image from \$DISPATCH_IMAGE ..."
docker pull \$DISPATCH_IMAGE

docker run \
-d -p 80:80 \
-e ENV=qa \
-e HOST=${QA_HOSTNAME} \
-e PORT=80 \
--env-file /home/ec2-user/dispatch-env \
-e "MONGO_HOST_1=internal-Mongo-1-ELB-1990770058.us-west-2.elb.amazonaws.com" \
-e "MONGO_HOST_2=internal-Mongo-2-ELB-755742902.us-west-2.elb.amazonaws.com" \
-e "MONGO_HOST_3=internal-Mongo-3-ELB-490053853.us-west-2.elb.amazonaws.com" \
--name ${PROJECT_NAME}_${PROJECT_ENV}_${VERSION}_${BUILD_NUMBER} \
\$DISPATCH_IMAGE
sleep 10s
docker image prune -a -f || true
sudo rm /home/ec2-user/dispatch-env
sudo rm /home/ec2-user/dispatch-keys        
""", encoding: "UTF-8")
             script {
                      sshPublisher(
                      continueOnError: false, failOnError: true,
                      publishers: [
                      sshPublisherDesc(
                      configName: "${PROJECT_NAME}${PROJECT_ENV}awsserver",
                      verbose: true,
                      transfers: [
                      sshTransfer(
                      sourceFiles: "${PROJECT_NAME}${PROJECT_ENV}.sh",
                      removePrefix: "",
                      remoteDirectory: "",
                      execCommand: "bash -x ${PROJECT_NAME}${PROJECT_ENV}.sh; rm -rf ${PROJECT_NAME}${PROJECT_ENV}.sh"
                  )
                 ])
               ])
             }
            }
                else if (env.PROJECT_ENV == 'prod') {
                   writeFile(file: "${PROJECT_NAME}${PROJECT_ENV}.sh", text: 
"""
#!/bin/bash
set -e
aws s3 cp s3://${DV_KEY_BUCKET}/rockit_key/dispatch-keys /home/ec2-user/dispatch-keys
DISPATCH_KEYS=\$(aws kms decrypt --region us-west-2 --ciphertext-blob fileb:///home/ec2-user/dispatch-keys --output text --query Plaintext | base64 --decode)
cat << EOF > /home/ec2-user/dispatch-env
\$DISPATCH_KEYS
EOF

chown ec2-user /home/ec2-user/dispatch-env
chmod go-rwx /home/ec2-user/dispatch-env
docker ps -aq --no-trunc -f status=exited -f status=created -f name=${PROJECT_NAME} | xargs docker rm 2>/dev/null || true
docker stop \$(docker ps -q -f name=${PROJECT_NAME}) 2>/dev/null || true
\$(aws ecr get-login --no-include-email --region us-west-2)
DISPATCH_IMAGE="${ECR_REPO_PROD}:${VERSION}"
echo "Pulling ${VERSION} dispatch image from \$DISPATCH_IMAGE ..."
docker pull \$DISPATCH_IMAGE

docker run \
-d -p 80:80 \
--env-file /home/ec2-user/dispatch-env \
-e ENV=pr \
-e PORT=80 \
-e "MONGO_HOST_1=internal-Mongo-1-ELB-239233192.us-west-2.elb.amazonaws.com" \
-e "MONGO_HOST_2=internal-Mongo-2-ELB-500941695.us-west-2.elb.amazonaws.com" \
-e "MONGO_HOST_3=internal-Mongo-3-ELB-2086926976.us-west-2.elb.amazonaws.com" \
--name ${PROJECT_NAME}_${PROJECT_ENV}_${VERSION}_${BUILD_NUMBER} \
\$DISPATCH_IMAGE

sleep 10s
docker image prune -a -f || true
sudo rm /home/ec2-user/dispatch-env
sudo rm /home/ec2-user/dispatch-keys 
        
""", encoding: "UTF-8")
             script {
                      sshPublisher(
                      continueOnError: false, failOnError: true,
                      publishers: [
                      sshPublisherDesc(
                      configName: "${PROJECT_NAME}${PROJECT_ENV}awsserver",
                      verbose: true,
                      transfers: [
                      sshTransfer(
                      sourceFiles: "${PROJECT_NAME}${PROJECT_ENV}.sh",
                      removePrefix: "",
                      remoteDirectory: "",
                      execCommand: "bash -x ${PROJECT_NAME}${PROJECT_ENV}.sh; rm -rf ${PROJECT_NAME}${PROJECT_ENV}.sh"
                  )
                 ])
               ])
             }
            }
                else {
                     echo "This stage is not for ${PROJECT_ENV}"    
            }
          }
        }
      }
        stage("Clean") {
        steps {
          cleanWs()
          sh """
            echo "Deleting dispatch unused images"
            docker image prune -a -f || true
          """
        }
      }           
     }
  post {
        aborted {
            echo('ERROR: Build process Aborted')
            mail to: "xxxx", subject:"Aborted: ${currentBuild.fullDisplayName}", body: "Name of the person who triggered this job - ${user_name} \n Email of the person who triggered this job - ${user_email} \n Build status - Aborted \n Job Name - ${env.JOB_NAME} \n Build Number - ${env.BUILD_NUMBER} \n Build URL - (${env.BUILD_URL}) \n"
        }
        failure {
            echo('ERROR: Build process failed')
            mail to: "xxxx", subject:"Failed: ${currentBuild.fullDisplayName}", body: "Name of the person who triggered this job - ${user_name} \n Email of the person who triggered this job - ${user_email} \n Build status - Failed \n Job Name - ${env.JOB_NAME} \n Build Number - ${env.BUILD_NUMBER} \n Build URL - (${env.BUILD_URL}) \n"
        }
        success {
            echo('INFO: Build process completed successfully')
            mail to: "xxxx", cc: "xxxx", bcc: "xxxx", subject:"Success: ${currentBuild.fullDisplayName}, Branch: ${BRANCH}, Environment: ${PROJECT_ENV}, Version: ${VERSION}", body: "Name of the person who triggered this job - ${user_name} \n Email of the person who triggered this job - ${user_email} \n Build status - Success \n Job Name - ${env.JOB_NAME} \n Build Number - ${env.BUILD_NUMBER} \n Build URL - (${env.BUILD_URL}) \n"
        }
    }
}
