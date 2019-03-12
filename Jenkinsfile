openshift.withCluster() {
  env.NAMESPACE =  openshift.project()
  env.APP_NAME = "${env.JOB_NAME}".replaceAll(/-?${env.NAMESPACE}-?/, '').replaceAll(/-?pipeline-?/, '').replaceAll('/', '')
  echo "begin build of " + env.APP_NAME
}

pipeline {
  agent { label 'maven' }
  stages {
    stage('Code Checkout') {
      steps {
        git url: "${SOURCE_CODE_URL}", branch: "${SOURCE_CODE_REF}"
      }
    }

    stage('Code Build') {
      steps {
        sh "mvn clean package"
      }
    }

    stage('Image Build') {
      steps {
        echo 'Building Image from Jar File'
        sh """
          set +x
          rm -rf oc-build && mkdir -p oc-build/deployments
          for t in \$(echo "jar;war;ear" | tr ";" "\\n"); do
            cp -rfv ./target/*.\$t oc-build/deployments/ 2> /dev/null || echo "No \$t files"
          done
        """

        script {
          openshift.withCluster() {
            build = openshift.startBuild("${APP_NAME}", "--from-dir=oc-build")

            timeout(10) {
              build.untilEach {
                def phase = it.object().status.phase
                echo "Build Status: ${phase}"

                if (phase == "Complete") {
                  return true
                }
                else if (phase == "Failed") {
                  currentBuild.result = "FAILURE"
                  buildErrorLog = it.logs().actions[0].err
                  return true
                }
                else {
                  return false
                }
              }
            }

            if (currentBuild.result == 'FAILURE') {
              error(buildErrorLog)
              return
            }
          }
        }
      }
    }
  }
}
