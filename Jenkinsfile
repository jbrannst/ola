node('maven') {
   def mvnCmd = "mvn"
   def TARGET = "blue"

   checkout scm
   stage ('Build') {
      //sh "oc new-build --binary --image-stream=openshift/redhat-openjdk18-openshift:1.1 --name=ola -l app=ola || true"
      sh "oc new-build --binary --name=ola -l app=ola || true"
      sh "${mvnCmd} package -DskipTests"
   }

   stage ('Deploy TEST') {
      //sh "mkdir -p ./target/oc-build/deployments/; mv -f target/ola.jar ./target/oc-build/deployments/"
      //sh "oc start-build ola --from-dir=./target/oc-build/ --wait"
      sh "oc start-build ola --from-dir=. --wait"
      sh "oc new-app ola --name=ola-test -l app=ola,app=ola-test,hystrix.enabled=true || oc deploy ola-test"
      sh "oc set probe dc/ola-test --readiness --get-url=http://:8080/api/health"
   }
   
   stage ('Smoke tests') {
      verify("ola-test")
     // further tests..
   }

   stage ('Deploy PROD') {
     
     // generate tag for production using the unique commit hash
     REVISION =  sh ( 
        script: "git rev-parse --short=12 HEAD", 
        returnStdout: true 
     ).trim()
     VERSION = "${REVISION}_${env.BUILD_NUMBER}"
     sh "oc tag ola:latest ola:${VERSION}"
     
     // check if blue deployment is active
     BLUE_ACTIVE =  sh ( 
        script: "oc get route ola | grep ola-blue", 
        returnStatus: true 
     ) == 0
      
     // deploy to passive
     TARGET = BLUE_ACTIVE ? "green" : "blue"
     echo "Deploying to ${TARGET}"
     sh "oc tag ola:${VERSION} ola:${TARGET}"

     
     sh "oc new-app ola:${TARGET} --name=ola-${TARGET} -l app=ola,app=ola-${TARGET},hystrix.enabled=true || true"
     sh "oc set probe dc/ola-${TARGET} --readiness --get-url=http://:8080/api/health"
     verify("ola-${TARGET}")
   }
   
   stage ('Activate'){
     timeout(time:5, unit:'MINUTES') {
        input message: "Activate in PROD?", ok: "Activate"
     }
     sh "oc process -f bluegreen-route-template.yaml -p  APPLICATION_INSTANCE=${TARGET} APPLICATION_NAME=ola | oc apply -f - "
   }
}
def verify(application) {
  openshiftVerifyDeployment(deploymentConfig: "${application}")
  sh "curl http://${application}:8080/api/health | grep \"ok\""
}
