#!groovy
podTemplate(
  label: "skopeo-pod",
  cloud: "openshift",
  inheritFrom: "maven",
  containers: [
    containerTemplate(
      name: "jnlp",
      image: "docker-registry.default.svc:5000/${GUID}-jenkins/jenkins-agent-appdev",
      resourceRequestMemory: "1Gi",
      resourceLimitMemory: "2Gi",
      resourceRequestCpu: "1",
      resourceLimitCpu: "2"
    )
  ]
) {
  node('skopeo-pod') {
    // Define Maven Command to point to the correct
    // settings for our Nexus installation
    def mvnCmd = "mvn -s ../nexus_settings.xml"

    // Checkout Source Code.
    stage('Checkout Source') {
      checkout scm
    }

    // Build the Tasks Service
    dir('openshift-tasks') {
      // The following variables need to be defined at the top level
      // and not inside the scope of a stage - otherwise they would not
      // be accessible from other stages.
      // Extract version from the pom.xml
      def version = getVersionFromPom("pom.xml")

      // TBD Set the tag for the development image: version + build number
      def devTag  = ""
      // Set the tag for the production image: version
      def prodTag = ""

      // Using Maven build the war file
      // Do not run tests in this step
      stage('Build war') {
        echo "Building version ${devTag}"

        sh "${mvnCmd} clean package -DskipTests"
      }

      // TBD: The next two stages should run in parallel

      // Using Maven run the unit tests
      stage('Unit Tests') {
        echo "Running Unit Tests"

        sh "${mvnCmd} test"
      }

      // Using Maven to call SonarQube for Code Analysis
      stage('Code Analysis') {
        echo "Running Code Analysis"

        sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube.gpte-hw-cicd.svc.cluster.local:9000 -Dsonar.projectName=${JOB_BASE_NAME}-${devTag}"
      }

      // Publish the built war file to Nexus
      stage('Publish to Nexus') {
        echo "Publish to Nexus"

        sh "${mvnCmd} deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://nexus3.gpte-hw-cicd.svc.cluster.local:8081/repository/all-maven-public"
      }

      // Build the OpenShift Image in OpenShift and tag it.
      stage('Build and Tag OpenShift Image') {
        echo "Building OpenShift container image tasks:${devTag}"

        sh "oc start-build tasks --follow --from-file=./target/openshift-tasks.war -n 1f34-tasks-dev"
	openshiftTag alias: 'false', destStream: 'tasks', destTag: devTag, destinationNamespace: '1f34-tasks-dev', namespace: '1f34-tasks-dev', srcStream: 'tasks', srcTag: 'latest', verbose: 'false'
      }

      // Deploy the built image to the Development Environment.
      stage('Deploy to Dev') {
        echo "Deploying container image to Development Project"

        // TBD: Deploy to development Project
        //      Set Image, Set VERSION
        //      Make sure the application is running and ready before proceeding

	sh "oc set image dc/tasks tasks=docker-registry.default.svc:5000/1f34-tasks-dev/tasks:${devTag} -n 1f34-tasks-dev"
	sh "oc delete configmap tasks-config -n 1f34-tasks-dev --ignore-not-found=true"
  	sh "oc create configmap tasks-config --from-file=./configuration/application-users.properties --from-file=./configuration/application-roles.properties -n 1f34-tasks-dev"

	openshiftDeploy depCfg: 'tasks', namespace: '1f34-tasks-prod', verbose: 'false', waitTime: '', waitUnit: 'sec'
  	openshiftVerifyDeployment depCfg: 'tasks', namespace: '1f34-tasks-prod', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: '', waitUnit: 'sec'
  	openshiftVerifyService namespace: '1f34-tasks-prod', svcName: 'tasks', verbose: 'false'
      }

      // Copy Image to Nexus container registry
      stage('Copy Image to Nexus container registry') {
        echo "Copy image to Nexus container registry"

        sh "skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds openshift:\$(oc whoami -t) --dest-creds admin:admin123 docker://docker-registry.default.svc.cluster.local:5000/1f34-tasks-dev/tasks:${devTag} docker://nexus3.gpte-hw-cicd.svc.cluster.local:8081/repository/all-maven-public:${devTag}"

        openshiftTag alias: 'false', destStream: 'tasks', destTag: prodTag, destinationNamespace: '1f34-tasks-dev', namespace: '1f34-tasks-dev', srcStream: 'tasks', srcTag: devTag, verbose: 'false'
      }

      // Blue/Green Deployment into Production
      // -------------------------------------
      def destApp   = "tasks-green"
      def activeApp = ""

      stage('Blue/Green Production Deployment')       stage('Blue/Green Production Deployment') {
        // TBD: Determine which application is active
        //      Set Image, Set VERSION
        //      Deploy into the other application
        //      Make sure the application is running and ready before proceeding
		
		activeApp = sh(returnStdout: true, script: "oc get route tasks -n 1f34-tasks-prod -o jsonpath='{ .spec.to.name }'").trim()
		if (activeApp == "tasks-green") {destApp = "tasks-blue"}
		echo "Active Application:      " + activeApp
		echo "Destination Application: " + destApp
		
		sh "oc set image dc/${destApp} ${destApp}=docker-registry.default.svc:5000/1f34-tasks-dev/tasks:${prodTag} -n 1f34-tasks-prod"
		
		sh "oc delete configmap ${destApp}-config -n 1f34-tasks-prod --ignore-not-found=true"
		sh "oc create configmap ${destApp}-config --from-file=./configuration/application-users.properties --from-file=./configuration/application-roles.properties -n 1f34-tasks-prod"
		openshiftDeploy depCfg: destApp, namespace: '1f34-tasks-prod', verbose: 'false', waitTime: '', waitUnit: 'sec'
		openshiftVerifyDeployment depCfg: destApp, namespace: '1f34-tasks-prod', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'true', waitTime: '', waitUnit: 'sec'
		openshiftVerifyService namespace: '1f34-tasks-prod', svcName: destApp, verbose: 'false'
      }

      stage('Switch over to new Version') {
        echo "Switching Production application to ${destApp}."
        sh 'oc patch route tasks -n 1f34-tasks-prod -p \'{"spec":{"to":{"name":"' + destApp + '"}}}\''
      }
    }
  }
}

// Convenience Functions to read version from the pom.xml
// Do not change anything below this line.
// --------------------------------------------------------
def getVersionFromPom(pom) {
  def matcher = readFile(pom) =~ '<version>(.+)</version>'
  matcher ? matcher[0][1] : null
}
