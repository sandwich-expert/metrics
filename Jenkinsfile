// Describes the build process in Groovy

node
{
    try
    {
      stage 'Prepare'
      env.PATH = "${tool 'maven-3.5.4'}/bin:${env.PATH}"
      checkout scm

      sh "env"

      //copy settings.xml
      configFileProvider([configFile(fileId: '91f54dd7-2a64-4258-9b70-216d70fc021c', targetLocation: './.m2/settings.xml', variable: 'MAVEN_SETTINGS')])
      {
        stage 'Build and Test'

        // -U => Ask maven to download latest version of sandwich-agent bundles.zip
        // -B => Run in non-interactive mode, meaning, maven won't log download progress
        // -Pci => Enable profile `ci`, this profile will tag docker images with commit hashes
        // -s $MAVEN_SETTINGS => Use jenkins settings.xml with auth for docker hub and s3
        // -e => Print stack trace on errors
        // -T?C => Tell maven to build spaw ? thread per core. eg: -T1C means 8 cores => 8 threads
        // -ff => fail fast, fail as soon any error happen
        MAVEN_OPTS = "-U -B -Pci -s $MAVEN_SETTINGS -e -T1C -ff"

        // Deploy jars to s3 if building on master via deploy task
        sh "mvn clean deploy $MAVEN_OPTS"
      }

      stage 'Report'

      archiveUnitTestResults()
      getTestCoverage()

      currentBuild.result = "SUCCESS"

      stage 'Clean up'
      cleanupWorkspace()

      slackSend channel: '#tlp-sandwich-jenkins', color: 'good', message: ":smile: Build successful - ${env.JOB_NAME} ${env.BUILD_NUMBER} - ${env.BUILD_URL}"
    }
    catch (caughtError)
    {
      archiveUnitTestResults()

      // Send slack notifications for failed or unstable builds.
      // currentBuild.result must be non-null for this step to work.
      if ("${env.JOB_NAME}".contains("master"))
      {
        slackSend channel: '#tlp-sandwich', color: 'bad', message: ":cry: Build failed - ${env.JOB_NAME} ${env.BUILD_NUMBER} - ${env.BUILD_URL}"
      }
      slackSend channel: '#tlp-sandwich-jenkins', color: 'bad', message: ":cry: Build failed - ${env.JOB_NAME} ${env.BUILD_NUMBER} - ${env.BUILD_URL}"

      currentBuild.result = "FAILURE"
      throw caughtError
    }
}

def archiveUnitTestResults() {
   // `*-reports/TEST-` includes both surefire and failsafe reports
   step([$class: 'JUnitResultArchiver', allowEmptyResults: true, testResults: '**/target/*-reports/TEST-*.xml'])
}

def getTestCoverage(){
 step([$class: 'JacocoPublisher'])
}

def cleanupWorkspace(){
  //clean up workspace
  step([$class: 'WsCleanup', notFailBuild: true])
  // delete sandwich files from local cache
  // this is to prevent feature branches from leaking artifacts to sandwich build
  sh 'rm -rf ~/.m2/repository/com/thelastpickle/sandwich/'

  sh returnStatus: true, script: 'docker rm $(docker ps --all --quiet)'
  sh returnStatus: true, script: 'docker rmi --force $(docker images -q thelastpickle/service*)'
  sh returnStatus: true, script: 'docker rmi --force $(docker images --quiet --filter "dangling=true")'
}
