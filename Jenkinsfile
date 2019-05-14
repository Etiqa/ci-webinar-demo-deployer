def releasesList = [];

def select(values,title) {
    return input(
            id: 'userInput', message: title, parameters: [
            [$class: 'hudson.model.ChoiceParameterDefinition', choices: values.join('\n'), description: '', name: title]
    ])
}

def getReleasesList (serverAddress,user,password,group,artifact) {
    def authString  = "${user}:${password}".getBytes().encodeBase64().toString()
    def addr = "${serverAddress}/maven-metadata.xml"
    def conn = addr.toURL().openConnection()

    conn.setRequestProperty( "Authorization", "Basic ${authString}" )

    if( conn.responseCode == 200 ) {
        def response = new XmlSlurper().parseText( conn.content.text )
        def versionsList = []

        for (item in response.versioning.versions.version) {
            versionsList.push(item.text())
        }
        return versionsList.reverse()
    } else {
        throw new Exception("getReleasesList failed: ${conn.responseCode} ${conn.responseMessage}")
    }
}

//Clean Workspace
stage "Clean workspace"
node {
  deleteDir()
}

stage 'Select build'
withCredentials([usernamePassword(credentialsId: 'nexus', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
    releasesList = getReleasesList("http://localhost:8081/repository/maven-releases/it/etiqa/webinar-project", USERNAME, PASSWORD, null, null)
}

timeout(time:1, unit:'HOURS') {
  if (params.buildNumber) {
    buildNumber = params.buildNumber
  } else {
    timeout(time:15, unit:'DAYS') {
      buildNumber = select(releasesList,"Build number")
    }
  }
}

currentBuild.displayName = "Deploying ${buildNumber}"

stage 'Downloading artifact'
node {
withCredentials([usernamePassword(credentialsId: 'nexus', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
    sh  "curl http://${USERNAME}:${PASSWORD}@localhost:8081/repository/maven-releases/it/etiqa/webinar-project/${buildNumber}/webinar-project-${buildNumber}.tar.gz -o dist.tar.gz"
}

stage "Select deploy target"
timeout(time:1, unit:'HOURS') {
  if (params.profile) {
    serverProfile = params.serverProfile
  } else {
    timeout(time:15, unit:'DAYS') {
      serverProfile = select(["QA", "PROD"],"Select server")
    }
  }
}

currentBuild.displayName = "Deploying ${buildNumber} on ${serverProfile}"

switch(serverProfile) {
    case "QA":
        withCredentials([sshUserPrivateKey(credentialsId: 'webinarsshqa', keyFileVariable: 'keyfile')]) {
            stage('Deploy to QA') {
                sh "scp -i ${keyfile} dist.tar.gz ubuntu@18.184.209.203:/opt/webinar/"
                sh "ssh -i ${keyfile} ubuntu@18.184.209.203 'cd /opt/webinar && rm -fr dist.old && mv dist dist.old && tar zxvf dist.tar.gz && pm2 restart server'"
            }
        }
        break;
    case "PROD":
        withCredentials([sshUserPrivateKey(credentialsId: 'webinarsshprod', keyFileVariable: 'keyfile')]) {
            stage('Deploy to PROD') {
                sh "scp -i ${keyfile} dist.tar.gz ubuntu@18.195.61.217:/opt/webinar/"
                sh "ssh -i ${keyfile} ubuntu@18.195.61.217 'cd /opt/webinar && rm -fr dist.old && mv dist dist.old && tar zxvf dist.tar.gz && pm2 restart server'"
            }
        }
        break;
}
}
