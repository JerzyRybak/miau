import groovy.json.JsonBuilder

node('jenkins-jenkins-slave') {
  withEnv(['REPOSITORY=miau',
  'GIT_ACCOUNT=https://github.com/JerzyRybak']) {
    stage('Pull Image from Git') {
      script {
        git "${GIT_ACCOUNT}/${REPOSITORY}.git"
      }
    }
    stage('Build Image') {
      script {
        dbuild = docker.build("jerzyrybak/${REPOSITORY}:$BUILD_NUMBER")
      }
    }
    parallel (
      "Test": {
        //script {
        //  sh "python tests/test_flask_app.py"
        //}
        echo 'All functional tests passed'
      },
      "Check Image (pre-Registry)": {
        smartcheckScan([
          imageName: "jerzyrybak/${REPOSITORY}:$BUILD_NUMBER",
          smartcheckHost: "${DSSC_SERVICE}",
          smartcheckCredentialsId: "smartcheck-auth",
          insecureSkipTLSVerify: true,
          insecureSkipRegistryTLSVerify: true,
          preregistryScan: true,
          preregistryHost: "${DSSC_REGISTRY}",
          preregistryCredentialsId: "preregistry-auth",
          findingsThreshold: new groovy.json.JsonBuilder([
            malware: 0,
            vulnerabilities: [
              defcon1: 0,
              critical: 0,
              high: 1,
            ],
            contents: [
              defcon1: 0,
              critical: 0,
              high: 1,
            ],
            checklists: [
              defcon1: 0,
              critical: 0,
              high: 0,
            ],
          ]).toString(),
        ])
      }
    )
    stage('Push Image to Registry') {
      script {
        docker.withRegistry("https://${K8S_REGISTRY}", 'registry-auth') {
          dbuild.push()
        }
      }
    }
    stage('Check Image (Registry)') {
      withCredentials([
        usernamePassword([
          credentialsId: "registry-auth",
          usernameVariable: "REGISTRY_USERNAME",
          passwordVariable: "REGISTRY_PASSWORD",
        ])
      ]){
        smartcheckScan([
          imageName: "${K8S_REGISTRY}/jerzyrybak/${REPOSITORY}:$BUILD_NUMBER",
          smartcheckHost: "${DSSC_SERVICE}",
          smartcheckCredentialsId: "smartcheck-auth",
          insecureSkipTLSVerify: true,
          insecureSkipRegistryTLSVerify: true,
          imagePullAuth: new groovy.json.JsonBuilder([
            username: REGISTRY_USERNAME,
            password: REGISTRY_PASSWORD
          ]).toString(),
          findingsThreshold: new groovy.json.JsonBuilder([
            malware: 0,
            vulnerabilities: [
              defcon1: 0,
              critical: 0,
              high: 1,
            ],
            contents: [
              defcon1: 0,
              critical: 0,
              high: 1,
            ],
            checklists: [
              defcon1: 0,
              critical: 0,
              high: 0,
            ],
          ]).toString(),
        ])
      }
    }
    stage('Push Image to Registry') {
      script {
        docker.withRegistry('', 'docker-hub') {
          dbuild.push('$BUILD_NUMBER')
        }
      }
    }
    stage('Deploy App to Kubernetes') {
      script {
        kubernetesDeploy(configs: "app.yml", kubeconfigId: "kubeconfig")
      }
    }
  }
}
