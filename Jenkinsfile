// Versioning - edit these variables to set version information
dockerfileVersion = '1.0.2'
latestVersion = '12.16.0'
nodeVersions = ['8.17.0', '12.16.0']

// Constants
registry = DOCKER_REGISTRY
imageNameDevelopment = 'node-development'
imageNameProduction = 'node'

// Variables
repoUrl = ''
commitSha = ''
versionTag = ''
imageRepositoryDevelopment = ''
imageRepositoryProduction = ''
imageRepositoryDevelopmentLatest = ''
imageRepositoryProductionLatest = ''
tagExists = false

def setCommonVariables() {
  repoUrl = getRepoUrl()
  commitSha = getCommitSha()
  imageRepositoryDevelopmentLatest = "$registry/$imageNameDevelopment"
  imageRepositoryProductionLatest = "$registry/$imageNameProduction"
}

def setImageVariables(nodeVersion) {
  versionTag = "$dockerfileVersion-node$nodeVersion"
  imageRepositoryDevelopment = "$registry/$imageNameDevelopment:$versionTag"
  imageRepositoryProduction = "$registry/$imageNameProduction:$versionTag"  
}

def getRepoUrl() {
  return sh(returnStdout: true, script: "git config --get remote.origin.url").trim()
}

def getCommitSha() {
  return sh(returnStdout: true, script: "git rev-parse HEAD").trim()
}

def updateGithubCommitStatus(message, state) {
  step([
    $class: 'GitHubCommitStatusSetter',
    reposSource: [$class: "ManuallyEnteredRepositorySource", url: repoUrl],
    commitShaSource: [$class: "ManuallyEnteredShaSource", sha: commitSha],
    errorHandlers: [[$class: 'ShallowAnyErrorHandler']],
    statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: message, state: state]] ]
  ])
}

def getImageTags(image) {
  def tags = sh(script: "curl https://index.docker.io/v1/repositories/$image/tags", returnStdout: true)
  return tags
}

def checkTagExists(image) {
  def existingTags = getImageTags(image)
  if(existingTags.contains(versionTag)) {
    echo "current tag exists in repository"
    tagExists = true
  } else {
    tagExists = false
  }
}

def buildImage(image, target, nodeVersion) {
  sh "docker build --no-cache \
    --tag $image \
    --build-arg NODE_VERSION=$nodeVersion \
    --build-arg VERSION=$dockerfileVersion \
    --target $target \
    ."
}

def pushImage(image) {
  withCredentials([
    usernamePassword(credentialsId : DOCKERHUB_CREDENTIALS_ID, usernameVariable: 'username', passwordVariable: 'password')    
  ]) {
    sh "docker login --username $username --password $password"
    sh "docker push $image"
  }
}

node {
  checkout scm
  try {
    stage('Set common variables') {
      setCommonVariables()
    }
    stage('Set GitHub status pending') {
      updateGithubCommitStatus('Build started', 'PENDING')
    }
    if(BRANCH_NAME == 'master') {
      nodeVersions.each {
        stage('Set image variables') {
          setImageVariables(it)
        }
        stage("Check if tag exists in repository ($it)") {
          checkTagExists(imageRepositoryProductionLatest)
        }
        if(!tagExists) {
          stage("Build development image ($it)") {
            buildImage(imageRepositoryDevelopment, 'development', it)
          }
          stage("Build production image ($it)") {
            buildImage(imageRepositoryProduction, 'production', it)
          }
          stage("Push development image ($it)") {
            pushImage(imageRepositoryDevelopment)
          }
          stage("Push production image ($it)") {
            pushImage(imageRepositoryProduction)
          }
          if(it == latestVersion) {
            stage('Build development image (latest)') {
              buildImage(imageRepositoryDevelopmentLatest, 'development', it)
            }
            stage('Build production image (latest)') {
              buildImage(imageRepositoryProductionLatest, 'production', it)
            }
            stage('Push development image (latest)') {
              pushImage("$imageRepositoryDevelopmentLatest:latest")
            }
            stage('Push production image (latest)') {
              pushImage("$imageRepositoryProductionLatest:latest")
            }
          }
        }
      }
    }
    stage('Set GitHub status success') {
      updateGithubCommitStatus('Build successful', 'SUCCESS')
    }
  } catch(e) {
    stage('Set GitHub status failure') {
      updateGithubCommitStatus(e.message, 'FAILURE')
    }
    throw e
  }
}