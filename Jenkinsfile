#!/usr/bin/env groovy
@Library('apm@current') _

pipeline {
  agent { label 'linux && immutable' }
  environment {
    REPO = 'opbeans-java'
    BASE_DIR = "src/github.com/elastic/${env.REPO}"
    JOB_GCS_CREDENTIALS = 'apm-ci-gcs-plugin'
    DOCKERHUB_SECRET = 'secret/apm-team/ci/elastic-observability-dockerhub'
    PIPELINE_LOG_LEVEL = 'INFO'
    GITHUB_CHECK_ITS_NAME = 'Integration Tests'
    ITS_PIPELINE = 'apm-integration-tests-selector-mbp/master'
    DOCKER_REGISTRY_SECRET = 'secret/apm-team/ci/docker-registry/prod'
    TMP_IMAGE = "${env.REGISTRY}/observability-ci/${env.REPO}"
    REGISTRY = 'docker.elastic.co'
    PATH = "${env.PATH}:${env.WORKSPACE}/bin:${env.WORKSPACE}/${BASE_DIR}/.ci/scripts"
    HOME = "${env.WORKSPACE}"
  }
  options {
    timeout(time: 1, unit: 'HOURS')
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '20', daysToKeepStr: '30'))
    timestamps()
    ansiColor('xterm')
    disableResume()
    durabilityHint('PERFORMANCE_OPTIMIZED')
    rateLimitBuilds(throttle: [count: 60, durationName: 'hour', userBoost: true])
    quietPeriod(10)
  }
  triggers {
    issueCommentTrigger('(?i).*(?:jenkins\\W+)?run\\W+(?:the\\W+)?tests(?:\\W+please)?.*')
  }
  stages {
    /**
     Checkout the code and stash it, to use it on other stages.
    */
    stage('Checkout') {
      options { skipDefaultCheckout() }
      steps {
        deleteDir()
        gitCheckout(basedir: "${BASE_DIR}")
        stash allowEmpty: true, name: 'source', useDefaultExcludes: false
      }
    }
    /**
    Build the project from code..
    */
    stage('Build') {
      steps {
        withGithubNotify(context: 'Build') {
          deleteDir()
          unstash 'source'
          dir(BASE_DIR){
            sh 'echo build'
          }
        }
      }
    }
    /**
    Execute unit tests.
    */
    stage('Test') {
      steps {
        withGithubNotify(context: 'Test', tab: 'tests') {
          deleteDir()
          unstash 'source'
          dir(BASE_DIR){
            sh "echo test"
          }
        }
      }
    }
    /**
    Push docker image to the staging docker registry
    */
    stage('Staging') {
      steps {
        withGithubNotify(context: 'Staging') {
          deleteDir()
          unstash 'source'
          dir(BASE_DIR){
            sh script: "VERSION=${env.GIT_BASE_COMMIT} IMAGE=${env.TMP_IMAGE} echo publish", label: "push docker image to ${env.TMP_IMAGE}"
          }
        }
      }
    }
    stage('Integration Tests') {
      agent none
      steps {
        echo 'its'
      }
    }
    stage('Release') {
      when {
        beforeAgent true
        allOf {
          anyOf {
            branch 'master'
            tag pattern: 'v\\d+\\.\\d+\\.\\d+.*', comparator: 'REGEXP'
          }
        }
      }
      stages {
        stage('Publish') {
          steps {
            withGithubNotify(context: 'Publish') {
              deleteDir()
              unstash 'source'
              dir(BASE_DIR){
                sh "VERSION=${BRANCH_NAME.equals('master') ? 'latest' : BRANCH_NAME} echo publish"
              }
            }
          }
        }
        stage('Release Notes') {
          steps {
            withGithubNotify(context: 'Release Notes') {
              deleteDir()
              unstash 'source'
              dir(BASE_DIR){
                withCredentials([string(credentialsId: '2a9602aa-ab9f-4e52-baf3-b71ca88469c7', variable: 'GREN_GITHUB_TOKEN')]) {
                  sh(label: 'Creating Release Notes', script: '.ci/scripts/release-notes.sh')
                }
              }
            }
          }
        }
      }
    }
  }
}
