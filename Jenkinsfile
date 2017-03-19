// Mandatory environment variables:
// DOCKER_REGISTRY (without http://, no ending /)
// KUBERNETES_SECRET_KEY
// k8s_username
// k8s_password
// k8s_tenant
// k8s_name
// k8s_resourceGroup

def buildVersion = null
def short_commit = null
echo "Building ${env.BRANCH_NAME}"
properties([[$class: 'BuildDiscarderProperty', strategy: [$class: 'LogRotator', artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '5']]])
stage 'Build'
node('docker-cloud') {
    git 'https://github.com/jpbriend/mobile-deposit-api-devoxx2017.git'
    sh('git rev-parse HEAD > GIT_COMMIT')
    git_commit=readFile('GIT_COMMIT')
    short_commit=git_commit.take(7)
    docker.image('kmadel/maven:3.3.3-jdk-8').inside('-v /data:/data') {
        sh "mvn -Dmaven.repo.local=/data/mvn/repo -DGIT_COMMIT='${short_commit}' -DBUILD_NUMBER=${env.BUILD_NUMBER} -DBUILD_URL=${env.BUILD_URL} clean package"
    }
    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/TEST-*.xml'
    stash name: 'pom', includes: 'pom.xml'
    stash name: 'jar-dockerfile', includes: '**/target/*.jar,**/target/Dockerfile'
    stash name: 'deployment.yml', includes:'deployment.yml'
}


checkpoint 'Quality Analysis Complete'
def dockerTag = "${env.BUILD_NUMBER}-${short_commit}"
stage('Version Release') {
node('docker-cloud') {
    unstash 'pom'
    
    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
    if (matcher) {
        buildVersion = matcher[0][1]
        echo "Release version: ${buildVersion}"
    }
    matcher = null
    
    stage 'Build Docker Image'
    def mobileDepositApiImage
    //unstash Spring Boot JAR and Dockerfile
    unstash 'jar-dockerfile'
    dir('target') {
        mobileDepositApiImage = docker.build "${DOCKER_REGISTRY}/mobile-deposit-api:${dockerTag}"
    }
    
    stage('Publish Docker Image') {
      withDockerRegistry([url: "https://${DOCKER_REGISTRY}/v2", credentialsId: 'test-registry']) { 
        mobileDepositApiImage.push()
      }
    }
  
  //set checkpoint before deployment
  checkpoint 'Build Complete'
  stage('Deploy to Prod') {

    docker.image('jcorioland/devoxx2017attendee').inside('-v /data:/data') {

            withCredentials([file(credentialsId: 'kuby', variable: 'KUBERNETES_SECRET_KEY')]) {
                unstash 'deployment.yml'
                sh """
                  az login --service-principal -u ${env.k8s_username} -p ${env.k8s_password} --tenant ${env.k8s_tenant}
                  mkdir -p ~/.ssh

                  (cat ${KUBERNETES_SECRET_KEY}; echo '\n') > ~/.ssh/id_rsa

                  az acs kubernetes get-credentials -n ${env.k8s_name} -g ${env.k8s_resourceGroup}
                  kubectl version
                  
                  kubectl create -f ./deployment.yml ||true
                  
                  if [ "\$?" -ne "0" ]; then
                    kubectl apply -f ./deployment.yml
                  fi
                  
                  
                  kubectl set image deployment/mobile-deposit-api-deployment mobile-deposit-api=${env.DOCKER_REGISTRY}/mobile-deposit-api:${dockerTag}
                  kubectl rollout status deployment/mobile-deposit-api-deployment
                  
                  kubectl expose -f ./deployment.yml --type=LoadBalancer ||true
                  
                  """
                  //send commit status to GitHub
                  //step([$class: 'GitHubCommitStatusSetter', contextSource: [$class: 'ManuallyEnteredCommitContextSource', context: 'Jenkins'], statusResultSource: [$class: 'ConditionalStatusResultSource', results: [[$class: 'BetterThanOrEqualBuildResult', message: 'Pipeline completed successfully', result: 'SUCCESS', state: 'SUCCESS']]]])
                  currentBuild.result = "success"
            }
      }
    }
  }
}
