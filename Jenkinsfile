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
         checkout scm
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
            bc = openshift.selector( "bc/${APP_NAME}" ).object()
            image = bc.spec.output.to.name
            image = image.replaceAll("-\\d{1,}\$","-${BUILD_NUMBER}")
            echo image
            bc.spec.output.to.name=image
            openshift.apply(bc)
            
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
   stage('Image Scan'){
      steps{
         script {
             tag = image.replaceAll("^.+?:","")
             tagInfo = httpRequest ignoreSslErrors:true, url:"http://quay-enterprise-quay-enterprise.apps.andy-e2.casl-contrib.osp.rht-labs.com/api/v1/repository/admin/security-demo/tag/${tag}/images"
             tagInfo = readJSON text: tagInfo.content
             index_max = -1
             for( imageRef in tagInfo.images ) {
                 if( imageRef.sort_index > index_max ) {
                     imageId = imageRef.id
                 }
             }

             timeout(time: 5, unit: 'MINUTES') {

                 waitUntil() {

                     vulns = httpRequest ignoreSslErrors:true, url:"https://quay-enterprise-quay-enterprise.apps.andy-e2.casl-contrib.osp.rht-labs.com/api/v1/repository/admin/security-demo/image/${imageId}/security?vulnerabilities=true"
                     vulns = readJSON text: vulns.content  
                     if(vulns.status != "scanned"){
                         return false
                     }

                     low=[]
                     medium=[]
                     high=[]
                     critical=[]
                     
                     for ( rpm in vulns.data.Layer.Features ){
                         vulnList = rpm.Vulnerabilities
                         if(vulnList != null && vulnList.size() != 0){
                             for(vuln in vulnList){
                                 switch(vuln.Severity){
                                     case "Low":
                                         low.add(vuln)
                                         break
                                     case "Medium":
                                         medium.add(vuln)
                                         break
                                     case "High":
                                         high.add(vuln)
                                         break
                                     case "Critical":
                                         critical.add(vuln)
                                         break
                                     default:
                                         echo "Should never be here"
                                         currentBuild.result = "FAILURE"
                                         break
                                   }
                              
                                 }
                             }
                         }
                         
                     


                     return true
                 }

             }
             
             echo low
             echo medium
             echo high
             echo critical
         }
      }
   }
  }
}
