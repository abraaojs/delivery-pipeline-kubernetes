#!groovy
podTemplate(label: 'mypod', containers: [
    containerTemplate(name: 'git', image: 'alpine/git', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'maven', image: 'maven:3.3.9-jdk-8-alpine', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true)
  ],
  volumes: [
    hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
  ]
) {
def releasedVersion
node('mypod') {
    def dockerTool = tool name: 'docker', type: 'org.jenkinsci.plugins.docker.commons.tools.DockerTool'
    withEnv(["DOCKER=${dockerTool}/bin"]) {
    stage('Prepare Check running containers') {
        container('docker') {
        sh 'hostname -i'   
        sh 'docker ps'
        deleteDir()
        parallel Checkout: {
            checkout scm
        }, 'Run Zalenium': {
            docker '''run -d --name zalenium -p 4444:4444 \
            -v /var/run/docker.sock:/var/run/docker.sock \
            --network="host" \
            --privileged dosel/zalenium:3.4.0a start --videoRecordingEnabled false --chromeContainers 1 --firefoxContainers 0'''
        }
    }
  }
    stage('Build') {
        container('maven') {
        withMaven(maven: 'Maven 3') {
            dir('app') {
                sh 'mvn clean package'
                docker 'build --tag automatingguy/sparktodo:SNAPSHOT .'
            }
        }
    }
} 

    stage('Deploy') {
        stage('Deploy') {
            dir('app') {
                docker 'run -d -p 9999:9999 --name "snapshot" --network="host" automatingguy/sparktodo:SNAPSHOT'
            }
        }
    }

    stage('Tests') {
        try {
            dir('tests/rest-assured') {
                sh './gradlew clean test'
            }
        } finally {
            junit testResults: 'tests/rest-assured/build/*.xml', allowEmptyResults: true
            archiveArtifacts 'tests/rest-assured/build/**'
        }

        docker 'rm -f snapshot'
        docker 'run -d -p 9999:9999 --name "snapshot" --network="host" automatingguy/sparktodo:SNAPSHOT'

        try {
            withMaven(maven: 'Maven 3') {
                dir('tests/bobcat') {
                    sh 'mvn clean test -Dmaven.test.failure.ignore=true'
                }
            }
        } finally {
            junit testResults: 'tests/bobcat/target/*.xml', allowEmptyResults: true
            archiveArtifacts 'tests/bobcat/target/**'
        }

        docker 'rm -f snapshot'
        docker 'stop zalenium'
        docker 'rm zalenium'
    }

    stage('Release') {
        withMaven(maven: 'Maven 3') {
            dir('app') {
                releasedVersion = getReleasedVersion()
                withCredentials([usernamePassword(credentialsId: 'github', passwordVariable: 'password', usernameVariable: 'username')]) {
                    sh "git config user.email test@automatingguy.com && git config user.name Jenkins"
                    sh "mvn release:prepare release:perform -Dusername=${username} -Dpassword=${password}"
                }
                docker "build --tag automatingguy/sparktodo:${releasedVersion} ."
            }
        }
    }

    stage('Deploy @ Prod') {
        docker "run -d -p 9999:9999 --name 'production' automatingguy/sparktodo:${releasedVersion}"
     }
    }
  }
}  
def docker(args) {
    sh "sudo ${DOCKER}/docker ${args}"
}
def getReleasedVersion() {
    return (readFile('pom.xml') =~ '<version>(.+)-SNAPSHOT</version>')[0][1]
    }
