#!groovy​

def settings = [
  scmCredentials: 'project-github',
  environments: []
]

settings.environments.put(test, [
  branches: '.*development',
  instances: [
    [
      label: "Author",
      credentials: 'project-test-author',
      url: "http://localhost:4502"
    ],
    [
      label: "Publish",
      credentials: 'project-test-publish1',
      url: "http://localhost:4503"
    ],
    [
      label: "Publish2",
      credentials: 'project-test-publish2',
      url: "http://localhost:4504"
    ]
  ]
])

properties([[$class: 'BuildDiscarderProperty', strategy: [$class: 'LogRotator', numToKeepStr: '10']]])

def branch_type = get_branch_type "${env.BRANCH_NAME}"
def branch_deployment_environment = get_branch_deployment_environment branch_type

// Build stage
stage('Build') {
    node {
        checkout scm
        def v = version()
        currentBuild.displayName = "${env.BRANCH_NAME}-${v}-${env.BUILD_NUMBER}"
        mvn "clean install"
    }
}

// Deploy stage
if (branch_deployment_environment) {
    def buildEnvironment = settings.environments[branch_deployment_environment];

    stage('Deploy') {
        if (branch_deployment_environment == "prod") {
            timeout(time: 1, unit: 'DAYS') {
                input "Deploy to ${branch_deployment_environment} ?"
            }
        }

        node {
            sh "echo Deploying to ${branch_deployment_environment}"
            //TODO specify the deployment
        }
    }
}

// Verify stage
if (branch_deployment_environment && branch_deployment_environment != "prod") {
    stage('Verify deployment') {
        node {
            sh "echo Running integration tests in ${branch_deployment_environment}"
            //TODO do the actual tests
        }
    }
}

// Start release
if (branch_type == "dev") {
    stage('Promote build to UAT') {
        timeout(time: 1, unit: 'HOURS') {
            input "Do you want to start a release?"
        }
        node {
            startRelease(settings.scmCredentials);
        }
    }
}

// Finish release
if (branch_type == "release") {
    stage('Promote build to PRD') {
        timeout(time: 1, unit: 'HOURS') {
            input "Is the release finished?"
        }
        node {
            finishRelease(settings.scmCredentials)
        }
    }
}

// Hotfix
if (branch_type == "hotfix") {
    stage('finish hotfix') {
        timeout(time: 1, unit: 'HOURS') {
            input "Is the hotfix finished?"
        }
        node {
            finishHotfix(settings.scmCredentials)
        }
    }
}

// Utility functions
def get_branch_type(String branch_name) {
    //Must be specified according to <flowInitContext> configuration of jgitflow-maven-plugin in pom.xml
    def dev_pattern = ".*development"
    def release_pattern = ".*release/.*"
    def feature_pattern = ".*feature/.*"
    def hotfix_pattern = ".*hotfix/.*"
    def master_pattern = ".*master"
    if (branch_name =~ dev_pattern) {
        return "dev"
    } else if (branch_name =~ release_pattern) {
        return "release"
    } else if (branch_name =~ master_pattern) {
        return "master"
    } else if (branch_name =~ feature_pattern) {
        return "feature"
    } else if (branch_name =~ hotfix_pattern) {
        return "hotfix"
    } else {
        return null;
    }
}

def get_branch_deployment_environment(String branch_type) {
    if (branch_type == "dev") {
        return "dev"
    } else if (branch_type == "release") {
        return "staging"
    } else if (branch_type == "master") {
        return "prod"
    } else {
        return null;
    }
}

def startRelease(String credentials) {
  jgitFlow("release-start", credentials)
}

def finishRelease(String credentials) {
  jgitFlow("release-finish", credentials)
}

def finishHotfix(String credentials) {
  jgitFlow("hotfix-finish", credentials)
}

def jgitFlow(String command, String credentials) {
  withCredentials([usernamePassword(credentialsId: credentials, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
    mvn("jgitflow:${command} -DpushReleases=true -DpushHotfixes=true -Dusername=${USERNAME} -Dpassword=${PASSWORD}")
  }
}

def mvn(String goals) {
    def mvnHome = tool "Maven-3.2.3"
    def javaHome = tool "JDK1.8.0_102"

    withEnv(["JAVA_HOME=${javaHome}", "PATH+MAVEN=${mvnHome}/bin"]) {
        sh "mvn -B ${goals}"
    }
}

def version() {
    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
    return matcher ? matcher[0][1] : null
}
