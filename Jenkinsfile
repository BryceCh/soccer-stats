#!groovy​

// FULL_BUILD -> true/false build parameter to define if we need to run the entire stack for lab purpose only
final FULL_BUILD = params.FULL_BUILD
// HOST_PROVISION -> server to run ansible based on provision/inventory.ini
final HOST_PROVISION = params.HOST_PROVISION
final GIT_URL = params.GIT_URL
final NEXUS_URL = params.NEXUS_URL
final NEXUS_REPO = 'maven-releases'
final TEMPLATE_ID = params.TEMPLATE_ID
final INVENTORY_ID = params.INVENTORY_ID

stage('Build') {
    node {
        git GIT_URL
        withEnv(["PATH+MAVEN=${tool 'm3'}/bin"]) {
            if(FULL_BUILD) {
                def pom = readMavenPom file: 'pom.xml'
                sh "mvn -B versions:set -DnewVersion=${pom.version}-${BUILD_NUMBER}"
                sh "mvn -B -Dmaven.test.skip=true clean package"
                stash name: "artifact", includes: "target/soccer-stats-*.war"
                stash name: "pom", includes: "pom.xml"
            }
        }
        parallel(
           unitTest: {
               stage('Unit Test') {
                   catchError {
                       sh "mvn -B clean test"
                   }
                   junit allowEmptyResults: true, testResults: '**/target/surefire-reports/TEST-*.xml'
               }
            },
            integrationTest: {
                stage('Integration Test') {
                    catchError {
                        sh "mvn -B clean verify -DskipUnitTests -Parq-wildfly-swarm"
                    }
                    junit allowEmptyResults: true, testResults: '**/target/failsafe-reports/*.xml'
                }
            }
        )
    }
}


if(FULL_BUILD) {
    stage('Statical Code Analysis') {
        withSonarQubeEnv('sonar') {
            sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=soccer-stats'
        }
    }
}


if(FULL_BUILD) {
    stage('Quality Gate') {
          if (currentBuild.currentResult == 'SUCCESS') {
              timeout(time: 10, unit: 'MINUTES') {
                  def qg = waitForQualityGate()
                  if (qg.status != 'OK') {
                      unstable("Pipeline unstable due to quality gate failure: ${qg.status}")
                  }
              }
          }
      }
}


if(FULL_BUILD) {
    stage('Approval') {
        timeout(time:3, unit:'DAYS') {
            input 'Do I have your approval for deployment?'
        }
    }
}


if(FULL_BUILD) {
    stage('Artifact Upload') {
        node {
            unstash 'pom'
            unstash 'artifact'

            def pom = readMavenPom file: 'pom.xml'
            def file = "${pom.artifactId}-${pom.version}"
            def jar = "target/${file}.war"

            sh "cp pom.xml ${file}.pom"

            nexusArtifactUploader artifacts: [
                    [artifactId: "${pom.artifactId}", classifier: '', file: "target/${file}.war", type: 'war'],
                    [artifactId: "${pom.artifactId}", classifier: '', file: "${file}.pom", type: 'pom']
                ], 
                credentialsId: 'nexus', 
                groupId: "${pom.groupId}", 
                nexusUrl: NEXUS_URL, 
                nexusVersion: 'nexus3', 
                protocol: 'https', 
                repository: NEXUS_REPO, 
                version: "${pom.version}"        
        }
    }
}


stage('Deploy') {
    node {
        unstash 'pom'
        def pom = readMavenPom file: "pom.xml"
        def repoPath =  "${pom.groupId}".replace(".", "/") + 
                        "/${pom.artifactId}"

        def version = pom.version

        if(!FULL_BUILD) { //takes the last version from repo
            sh "curl -o metadata.xml -s http://${NEXUS_URL}/repository/${NEXUS_REPO}/${repoPath}/maven-metadata.xml"
            version = sh script: 'xmllint metadata.xml --xpath "string(//latest)"',
                         returnStdout: true
        }
        def artifactUrl = "https://${NEXUS_URL}/repository/${NEXUS_REPO}/${repoPath}/${version}/${pom.artifactId}-${version}.war"
        def hostLimit = (HOST_PROVISION == "all" || HOST_PROVISION == null) ? "" : HOST_PROVISION
        def inventoryId = INVENTORY_ID
        def templateId = TEMPLATE_ID

        //you must specify your own ids
        ansibleTower(
                towerServer: 'tower',
                templateType: 'workflow',
                jobTemplate: "${templateId}",
                importTowerLogs: true,
                inventory: "${inventoryId}",
                removeColor: false,
                verbose: true,
                limit: "${hostLimit}",
                extraVars: """---
ARTIFACT_URL:  "${artifactUrl}"
APP_NAME: "${pom.artifactId}" 
"""
        )
    }
}
