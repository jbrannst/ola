node('maven') {
 // maven-settings should point to local nexus
   def mvnCmd = "mvn -s configuration/maven-settings.xml"
   def APPLICATION="kitchensink"
   def ARTIFACT="jboss-kitchensink-angularjs.war"
   def S2I_BUILDER="jboss-eap70-openshift"

 // fetch project configuration from parameters if available
   def TEST_PROJECT=params.TEST_PROJECT_PARAM == null ? "ks-test" : params.TEST_PROJECT_PARAM
   def PROD_PROJECT=params.PROD_PROJECT_PARAM == null ? "ks-prod" : params.PROD_PROJECT_PARAM

   checkout scm
   stage ('Build') {
     sh "${mvnCmd} clean install -DskipTests=true"
   }

   stage ('Test and Analysis') {
     parallel (
         'Test': {
            sh "${mvnCmd} test"
            step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
         },
         'Static Analysis': {
            sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube:9000 -DskipTests=true"
         }
     )
   }

   stage ('Push to Nexus') {
    sh "${mvnCmd} deploy -DskipTests=true"
   }

   stage ('Deploy TEST') {
     sh "rm -rf oc-build && mkdir -p oc-build/deployments"
     sh "cp target/${ARTIFACT} oc-build/deployments/ROOT.war"
     sh "oc project ${TEST_PROJECT}"
     // create build. override the exit code since it complains about exising imagestream and buildconfig
     sh "oc new-build --name=${APPLICATION} --image-stream=${S2I_BUILDER} --binary=true --labels=application=${APPLICATION} -n ${TEST_PROJECT} || true"
     // build image
     sh "oc start-build ${APPLICATION} --from-dir=oc-build --wait=true -n ${TEST_PROJECT}"
     // deploy
     deploy("${TEST_PROJECT}", "${APPLICATION}")
   }
   
   stage ('Smoke tests') {
     verify("${TEST_PROJECT}", "${APPLICATION}")
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
     sh "oc tag ${TEST_PROJECT}/${APPLICATION}:latest ${PROD_PROJECT}/${APPLICATION}:${VERSION}"
     sh "oc project ${PROD_PROJECT}"
     
     // check if blue deployment is active
     BLUE_ACTIVE =  sh ( 
        script: "oc get route ${APPLICATION} -n ${PROD_PROJECT} | grep ${APPLICATION}-blue", 
        returnStatus: true 
     ) == 0
     
     // deploy to passive
     TARGET = BLUE_ACTIVE ? "green" : "blue"
     echo "Deploying to ${TARGET}"
     
     deploy("${PROD_PROJECT}", "${APPLICATION}", "${TARGET}", "${VERSION}")
     verify("${PROD_PROJECT}", "${APPLICATION}", "${TARGET}")
     sh "oc process -f configuration/bluegreen-route-template.yaml -p  APPLICATION_INSTANCE=${TARGET} APPLICATION_NAME=${APPLICATION} | oc apply -f -  -n ${PROD_PROJECT}"
   }
}
def deploy(namespace, application, flavor = "test", version = "latest") {
  sh "oc process -f configuration/db-secret-template.yaml -p APPLICATION_NAME=${application}-${flavor} | oc create -f -  -n ${namespace} || true"
  sh "oc process -f configuration/postgres-template.yaml -p APPLICATION_NAME=${application}-${flavor} | oc apply -f -  -n ${namespace}"
  openshiftVerifyDeployment(deploymentConfig: "${application}-${flavor}-postgresql", namespace: "${namespace}")
  sh "oc process -f configuration/app-template.yaml -p IMAGE_VERSION=${version} APPLICATION_NAME=${application}-${flavor} | oc apply -f -  -n ${namespace}"
  openshiftVerifyDeployment(deploymentConfig: "${application}-${flavor}", namespace: "${namespace}")
}
def verify(namespace, application, flavor = "test") {
  openshiftVerifyDeployment(deploymentConfig: "${application}-${flavor}", namespace: "${namespace}")
  sh "curl -H \"Content-Type: application/json\" -X POST -d \'{\"name\":\"John Smith\",\"email\":\"john.smith@xyz.com\",\"phoneNumber\":\"1234567890\"}\' http://kitchensink-${flavor}.${namespace}.svc.cluster.local:8080/rest/members"
  sh "curl http://kitchensink-${flavor}.${namespace}.svc.cluster.local:8080/rest/members | grep -q \"John Smith\""
}
