#!/usr/bin/env groovy

pipeline {
  agent { label 'executor-v2' }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '30'))
  }

  triggers {
    cron(getDailyCronString())
  }

  parameters { 
    booleanParam(
      name: 'TEST_OCP_NEXT',
      description: 'Whether or not to run the pipeline against the next OCP version',
      defaultValue: false) 
  }

  stages {
    // Postgres Tests with Host-ID-based and Annotation-based Authn against OSS
    stage('Deploy Demos against OSS on Openshift') {
      parallel {
        stage('OpenShift v(current), v5 Conjur OSS, Postgres, Annotation-based Authn') {
          steps {
            sh 'cd ci && CONJUR_OSS=true summon --environment current ./test oc postgres annotation-based'
          }
        }
      }
    }
  }

  post {
    always {
      cleanupAndNotify(currentBuild.currentResult)
    }
  }
}
