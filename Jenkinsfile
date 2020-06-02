openshift.withCluster() {
  env.NAMESPACE =  openshift.project()
  env.APP_NAME = "${env.JOB_NAME}".replaceAll(/-?${env.NAMESPACE}-?/, '').replaceAll(/-?pipeline-?/, '').replaceAll('/', '')
  echo "begin build of " + env.APP_NAME
}

def template = "https://raw.githubusercontent.com/jgoldsmith613/java-base-march-security/master/signing-template.yaml"
def quayURL = "example-quayecosystem-quay-quay.apps.cluster-nyc-ea98.nyc-ea98.example.opentlc.com"
def repo = "security/${APP_NAME}"
def dev_namespace = "ola-dev"
def prod_namespace = "ola-prod"
def dev_route = "ola-ola-dev.apps.cluster-nyc-ea98.nyc-ea98.example.opentlc.com"

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

    stage('Sonar Scan') {
      steps {
        echo "stub for sonar scan" 
        sleep 3
      }
    }
    
    stage('Application Component Scan') {
      steps {
        echo "stub for Application Componenet Scan"
        sleep 3
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
             tagInfo = httpRequest ignoreSslErrors:true, url:"http://${quayURL}/api/v1/repository/${repo}/tag/${tag}/images"
             tagInfo = readJSON text: tagInfo.content
             index_max = -1
             for( imageRef in tagInfo.images ) {
                 if( imageRef.sort_index > index_max ) {
                     imageId = imageRef.id
                     index_max = imageRef.sort_index
                 }
             }

             timeout(time: 5, unit: 'MINUTES') {

                 waitUntil() {

                     vulns = httpRequest ignoreSslErrors:true, url:"https://${quayURL}/api/v1/repository/${repo}/image/${imageId}/security?vulnerabilities=true"
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
                             i = 0;
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
                                         echo "new vuln type?"
                                         break
                                   }
                              
                                 }
                             }
                         }
                         
                     


                     return true
                 }

             }

             if(critical.size() > 0 || high.size() > 0){
                 input "Image has ${critical.size()} critical vulnerabilities and ${high.size()} high vulnerabilities.  Please check https://${quayURL}/repository/${repo}/image/${imageId}?tab=vulnerabilities.  Would you like to proceed anyway?"
                 currentBuild.result = "UNSTABLE"
             }
             
                
         }
      }
   }
   stage('Sign Image'){
       steps {
           script{
               openshift.withCluster() {
                   sh " oc import-image ${APP_NAME}:${tag} --from=${quayURL}/${repo}:${tag} --insecure=true"
                   obj = "${APP_NAME}-${env.BUILD_NUMBER}"
                   created = openshift.create(openshift.process(template, "-p IMAGE_SIGNING_REQUEST_NAME=${obj} -p IMAGE_STREAM_TAG=${APP_NAME}:${tag}"))

                   imagesigningrequest = created.narrow('imagesigningrequest').name();

                   echo "ImageSigningRequest ${imagesigningrequest.split('/')[1]} Created"

                   timeout(time: 5, unit: 'MINUTES') {

                   waitUntil() {

                      def isr = openshift.selector("${imagesigningrequest}")

                      if(isr.object().status) {

                          def phase = isr.object().status.phase

                          if(phase == "Failed") {
                              echo "Signing Action Failed: ${isr.object().status.message}"
                              currentBuild.result = "FAILURE"
                              return true
                          }
                          else if(phase == "Completed") {
                              env.SIGNED_IMAGE = isr.object().status.signedImage
                              echo "Signing Action Completed. Signed Image: ${SIGNED_IMAGE}"
                              return true
                         }
                    }
                    else {
                        echo "Status is null"
                    }

                    return false

                 }
                }  
               } 

           }

       }
  }
  stage('Dev Deploy'){
      steps {
          script {
              openshift.withCluster() {
                  openshift.tag("${APP_NAME}:${tag}", "${dev_namespace}/${APP_NAME}:dev")
              }
          }
      }
  }

  stage('Dev Smoke Test'){
      steps {
          script {
              sleep 5
              timeout(time: 5, unit: 'MINUTES') {
                  waitUntil() {
                      responce = httpRequest ignoreSslErrors:true, url:"http://${dev_route}"
                      if( responce.status == 200 ){
                          return true
                      }
                      return false

                  }
              }
          }
      }
  }

  stage('Prod Deploy'){
      steps {
          script {
              input "Deploy to Prod"
              openshift.withCluster() {
                  openshift.tag("${APP_NAME}:${tag}", "${prod_namespace}/${APP_NAME}:prod")
              }
          }
      }
  }

  }
}
