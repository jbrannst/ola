node('maven') {
   def mvnCmd = "mvn"

   checkout scm
   stage ('Build') {
      sh "oc new-build --binary --name=ola -l app=ola"
      sh "${mvnCmd} package -DskipTests"
   }

   stage ('Test and Analysis') {
     parallel (
         'Test': {
            sh "${mvnCmd} test"
            step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
         //},
         //'Static Analysis': {
         //   sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube:9000 -DskipTests=true"
         }
     )
   }

   //stage ('Push to Nexus') {
   // sh "${mvnCmd} deploy -DskipTests=true"
   //}

   stage ('Deploy TEST') {
      sh "oc start-build ola --from-dir=. --wait"
      sh "oc new-app ola --name=ola-test -l app=ola,app=ola-test,hystrix.enabled=true"
   }
   
   stage ('Smoke tests') {
     verify("ola-test")
     // further tests..
   }

   stage ('Deploy PROD') {
     //timeout(time:5, unit:'MINUTES') {
     //   input message: "Promote to PROD?", ok: "Promote"
     //}
     
     // generate tag for production using the unique commit hash
     VERSION =  sh ( 
        script: "git rev-parse --short=12 HEAD", 
        returnStdout: true 
     ).trim()
     sh "oc tag ola:latest ola:${VERSION}"
     
     // check if blue deployment is active
     BLUE_ACTIVE =  sh ( 
        script: "oc get route production | grep ola-blue", 
        returnStatus: true 
     ) == 0
     
     // deploy to passive
     TARGET = BLUE_ACTIVE ? "green" : "blue"
     echo "Deploying to ${TARGET}"
     
     //deploy("ola-${TARGET}", "${VERSION}")
     sh "oc new-app ola:${VERSION} --name=ola-${TARGET} -l app=ola,app=ola-${TARGET},hystrix.enabled=true"
     verify("ola-${TARGET}")
     sh "oc process -f bluegreen-route-template.yaml -p  APPLICATION_INSTANCE=${TARGET} APPLICATION_NAME=ola | oc apply -f - "
   }
}
def verify(application) {
  openshiftVerifyDeployment(deploymentConfig: "${application}")
  sh "curl http://${application}.svc.cluster.local:8080/api/health | grep \"ok\""
}
