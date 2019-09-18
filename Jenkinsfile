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
            sh 'make build'
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
            sh "make test"
          }
        }
      }
      post {
        always {
          junit(allowEmptyResults: true,
            keepLongStdio: true,
            testResults: "${BASE_DIR}/**/junit-*.xml")
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
            dockerLogin(secret: "${env.DOCKER_REGISTRY_SECRET}", registry: "${env.REGISTRY}")
            sh script: "VERSION=${env.GIT_BASE_COMMIT} IMAGE=${env.TMP_IMAGE} make publish", label: "push docker image to ${env.TMP_IMAGE}"
          }
        }
      }
    }
    stage('Integration Tests') {
      agent none
      steps {
        build(job: env.ITS_PIPELINE, propagate: env.CHANGE_ID?.trim() ? false : true,
              wait: env.CHANGE_ID?.trim() ? false : true,
              parameters: [string(name: 'AGENT_INTEGRATION_TEST', value: 'Opbeans'),
                           string(name: 'BUILD_OPTS', value: "--with-opbeans-java --opbeans-java-image ${env.TMP_IMAGE} --opbeans-java-version ${env.GIT_BASE_COMMIT}"),
                           string(name: 'GITHUB_CHECK_NAME', value: env.GITHUB_CHECK_ITS_NAME),
                           string(name: 'GITHUB_CHECK_REPO', value: env.REPO),
                           string(name: 'GITHUB_CHECK_SHA1', value: env.GIT_BASE_COMMIT)])
        githubNotify(context: "${env.GITHUB_CHECK_ITS_NAME}", description: "${env.GITHUB_CHECK_ITS_NAME} ...", status: 'PENDING', targetUrl: "${env.JENKINS_URL}search/?q=${env.ITS_PIPELINE.replaceAll('/','+')}")
      }
    }
    stage('Release') {
      when {
        beforeAgent true
        allOf {
          anyOf {
            branch 'master'
            tag pattern: 'v\\d+\\.\\d+\\.\\d+*', comparator: 'REGEXP'
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
                dockerLogin(secret: "${DOCKERHUB_SECRET}", registry: 'docker.io')
                sh "VERSION=${BRANCH_NAME.equals('master') ? 'latest' : BRANCH_NAME} make publish"
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
                dockerLogin(secret: "${env.DOCKER_REGISTRY_SECRET}", registry: "${env.REGISTRY}")
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
  post {
    always {
      notifyBuildResult()
    }
  }
}
