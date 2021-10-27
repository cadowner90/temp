properties([
  parameters ([
          string(name: 'DOCKER_REGISTRY_DOWNLOAD_URL',
                  defaultValue: 'nexus-docker-private-group.ossim.io',
                  description: 'Repository of docker images')
  ]),13

  pipelineTriggers([
          [$class: "GitHubPushTrigger"]
  ]),

  [$class: 'GithubProjectProperty', displayName: '', projectUrlStr: 'https://github.com/cadowner90/temp'],

])

podTemplate(
  containers: [
    containerTemplate(
        image: "maven:3.6.2-jdk-8",
        name: 'maven',
        command: 'cat',
        ttyEnabled: true
    ),
    containerTemplate(
        name: 'fortify',
        image: "${DOCKER_REGISTRY_DOWNLOAD_URL}/fortifydocker/sca:20.2.0",
        ttyEnabled: true,
        command: 'cat',
        privileged: true
    )
  ],
  volumes: [
    hostPathVolume(
      hostPath: '/var/run/docker.sock',
      mountPath: '/var/run/docker.sock'
    ),
  ]
)
{
  node(POD_LABEL){

    stage("Checkout branch"){
      scmVars = checkout(scm)

      GIT_BRANCH_NAME = scmVars.GIT_BRANCH
      BRANCH_NAME = """${sh(returnStdout: true, script: "echo ${GIT_BRANCH_NAME} | awk -F'/' '{print \$2}'").trim()}"""
      VERSION = readMavenPom().getVersion()
      ARTIFACT_NAME = readMavenPom().getArtifactId()

      script {
        if (BRANCH_NAME != 'master') {
          VERSION = "${VERSION}-SNAPSHOT"
        }
      }

      buildName "${VERSION} - ${BRANCH_NAME}"
    }

    stage("Load Variables"){

      step([$class     : "CopyArtifact",
            projectName: "gegd-dgcs-jenkins-artifacts",
            filter     : "common-variables.groovy",
            flatten    : true])

      load "common-variables.groovy"
    }

    stage('Fortify Scan') {
      container('fortify') {
         script{
         sh """
                 echo "Running Fortify analysis and scan"
                 /opt/Fortify/Fortify_SCA_and_Apps_20.2.0/bin/sourceanalyzer -Xmx1G -b "op-dg-utils-scan" "${WORKSPACE}"/**/*.java
                 /opt/Fortify/Fortify_SCA_and_Apps_20.2.0/bin/sourceanalyzer -Xmx1G -b "op-dg-utils-scan" -scan -f "${WORKSPACE}"/"op-dg-utils-scan".fpr
                 echo "Done"
         """
         }
         archiveArtifacts "*.fpr"

         fortifyUpload appName: 'op-dg-utils', appVersion: VERSION, failureCriteria: '', filterSet: '', pollingInterval: '', resultsFile: 'op-dg-utils-scan.fpr'
      }
    }
  }
}
