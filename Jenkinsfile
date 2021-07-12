pipeline {
    environment {
        VERSION = getVersion()
        DOCKER_VERSION = getDockerVersion()
        registryCredential = 'docker-hub'
    }
    agent {
        docker {
            image 'python:3.9.5'
            args '-u root'
        }
    } 

    stages {
        stage('build') {
            steps {
                sh 'pip install --no-cache-dir -r requirements.txt'
            }
        }
        stage('build docker') {
            steps {
                script {
                        docker.withRegistry('https://registry.hub.docker.com/v1/repositories/woped', registryCredential) {
                            def dockerImage = docker.build("woped/text2process-stanford:$DOCKER_VERSION")
                            def dockerImageLatest = docker.build("woped/text2process-stanford:latest")
                            dockerImage.push();
                            dockerImageLatest.push();
                        }
                }
            }
        }

        stage('deploy when master') {

            when { branch 'master' }

            steps {
                script {
                    def remote = [:]
                    remote.name = "woped"
                    remote.host = "woped.dh-karlsruhe.de"
                    remote.allowAnyHosts = true
                    remote.sudo = true
                    remote.pty = true
                            
                    withCredentials([usernamePassword(credentialsId: 'sshUserWoPeD', passwordVariable: 'password', usernameVariable: 'userName')]) {
                        remote.user = userName
                        remote.password = password
                    }

                    stage('Remote SSH') {
                        sshCommand remote: remote, command: "sudo docker-compose -f /usr/local/bin/woped-webservice/docker-compose.yml pull t2p-stanford", sudo: true
                        sshCommand remote: remote, command: "sudo docker-compose -f /usr/local/bin/woped-webservice/docker-compose.yml up -d", sudo: true
                        sshCommand remote: remote, command: "sudo docker image prune -f", sudo: true
                    }
                    
                }
            }
        }

    }

    post {
        always {
            cleanWs()
            
            sh 'docker image prune -af'
        }
        success {
            setBuildStatus("Build succeeded", "SUCCESS");
        }
        failure {
            setBuildStatus("Build not Successfull", "FAILURE");
            
            emailext body: "Something is wrong with ${env.BUILD_URL}",
                subject: "Failed Pipeline: ${currentBuild.fullDisplayName}",
                to: '${DEFAULT_RECIPIENTS}'
        }
    }
}

def getVersion() {
    return '3.8.0-SNAPSHOT'
}

def getDockerVersion() {
    version = '3.8.0-SNAPSHOT'

    if(version.toString().contains('SNAPSHOT')) {
        return version + '-' + "${currentBuild.startTimeInMillis}"
    } else {
        return version
    }
}

void setBuildStatus(String message, String state) {
  step([
      $class: "GitHubCommitStatusSetter",
      reposSource: [$class: "ManuallyEnteredRepositorySource", url: "https://github.com/tfreytag/woped-stanford-core-rest-service"],
      contextSource: [$class: "ManuallyEnteredCommitContextSource", context: "ci/jenkins/build-status"],
      errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
      statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: message, state: state]] ]
  ]);
}
