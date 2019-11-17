#!/usr/bin/groovy
node {
    try{
        notifyBuild('STARTED')

        if(env.BRANCH_NAME == 'master'){

          stage('Checkout') {
            echo "Let the Builds Begin"
            checkout scm
          }

          stage('Tests') {
            checkout scm
            docker.image('node:carbon').inside {
              withEnv(['HOME=.']){
                sh 'npm install'
              }
            }
          }

          stage('Docker Build') {
            checkout scm
            docker.withRegistry('https://366670401047.dkr.ecr.us-east-2.amazonaws.com/future-airlines/k8s-nodejs','ecr:us-east-2:aws-ecr') {
              def customImage = docker.build("future-airlines/k8s-nodejs:${env.BRANCH_NAME}-${env.BUILD_NUMBER}")
              customImage.push()
            }

          } // build docker image

          stage('Deploy Staging'){
            checkout scm
            sh 'wget https://storage.googleapis.com/kubernetes-helm/helm-v2.13.1-linux-amd64.tar.gz && tar -xvf helm-v2.13.1-linux-amd64.tar.gz && mv linux-amd64/helm .'
            sh './helm lint --strict k8s-nodejs'
            sh './helm package k8s-nodejs --save=false'

            withCredentials(bindings: [sshUserPrivateKey(credentialsId: 'aws-ssh-access', keyFileVariable: 'AWSKEY')]) {
              sh 'ssh -oStrictHostKeyChecking=accept-new 18.221.14.187 uptime || true'
              sh 'scp -i $AWSKEY k8s-nodejs-0.1.0.tgz admin@18.221.14.187:/tmp/'

              sh 'ssh -i $AWSKEY admin@18.221.14.187 sudo microk8s.helm upgrade --install k8s-nodejs /tmp/k8s-nodejs-0.1.0.tgz --set image.tag=$BRANCH_NAME-$BUILD_NUMBER --dry-run'
              sh 'ssh -i $AWSKEY admin@18.221.14.187 sudo microk8s.helm upgrade --install k8s-nodejs /tmp/k8s-nodejs-0.1.0.tgz --set image.tag=$BRANCH_NAME-$BUILD_NUMBER'

            }

              
          } // deploy to Staging


        } //run only on develop branch

    } catch(e){
        // If there was an exception thrown, the build failed
        currentBuild.result = "FAILED"
        throw e
    } finally {
        notifyBuild(currentBuild.result)
    }

} //node

// slack and email notifications

def notifyBuild(String buildStatus = 'STARTED') {
  // build status of null means successful
  buildStatus =  buildStatus ?: 'SUCCESSFUL'

  // Default values
  def colorName = 'RED'
  def colorCode = '#FF0000'
  def subject = "${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
  def summary = "${subject} (${env.BUILD_URL})"
  def details = """<p>STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
    <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>"""

  // Override default values based on build status
  if (buildStatus == 'STARTED') {
    color = 'YELLOW'
    colorCode = '#FFFF00'
  } else if (buildStatus == 'SUCCESSFUL') {
    color = 'GREEN'
    colorCode = '#00FF00'
  } else {
    color = 'RED'
    colorCode = '#FF0000'
  }

  // Send notifications
  slackSend (color: colorCode, message: summary)
} //notifyBuild