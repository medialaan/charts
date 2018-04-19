#!/usr/bin/env groovy

if (!(env.BRANCH_NAME == 'medialaan')) {
  currentBuild.result = 'ABORTED'
  return
}

slackSend "<${env.JOB_DISPLAY_URL}|${env.JOB_NAME}> - build <${env.RUN_DISPLAY_URL}|${env.BUILD_DISPLAY_NAME}> started"
def slackResultColor
try {
  def podTemplateSuffix = "${env.JOB_NAME}-${env.BUILD_NUMBER}".replaceAll(/(\/|%2F)/, '-').toLowerCase()
  podTemplate(label: 'k8s-' + podTemplateSuffix, name: 'jenkins-agent-' + podTemplateSuffix, serviceAccount: 'jenkins', volumes: [hostPathVolume(hostPath: '/mnt/disks/ssd0', mountPath: '/home/jenkins')], nodeSelector: 'cloud.google.com/gke-preemptible=true,cloud.google.com/gke-local-ssd=true', nodeUsageMode: 'EXCLUSIVE', containers: [
    containerTemplate(
      name: 'jnlp',
      image: 'eu.gcr.io/medialaan-production/jenkins-jnlp-slave:3.16-1',
      args: '${computer.jnlpmac} ${computer.name}'
    ),
    containerTemplate(
      name: 'gcp',
      image: 'eu.gcr.io/medialaan-production/gcp:0.1.0',
      ttyEnabled: true,
      command: 'cat',
      envVars: [
        envVar(key: 'CLOUDSDK_CONFIG', value: '/tmp/.config'),
        envVar(key: 'HELM_HOME', value: '/tmp/.helm')
      ]
    )]) {
    node('k8s-' + podTemplateSuffix) {
      def scmVars
      stage('Checkout') {
        scmVars = checkout scm
      }
      container('gcp') {
        stage('Clean') {
          sh """
            rm -rf dist
          """
        }
        stage('Package and index') {
          sh """
            helm init --client-only
            helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com/
            helm repo remove local
            mkdir dist
            cd dist
            for repo in stable incubator; do
              mkdir \$repo
              cd \$repo
              for chart in ../../\$repo/*; do
                helm package \$chart --save=false --dependency-update
              done
              helm repo index . --url=http://charts.mldigital.be/kubernetes/\$repo/
              cd ..
            done
          """
        }
        stage('Push') {
          withCredentials([string(credentialsId: 'gcp', variable: 'gcpServiceAccount')]) {
            sh """
              printf '%s' '${gcpServiceAccount}' > /gcpServiceAccountKey.json
              gcloud auth activate-service-account --key-file /gcpServiceAccountKey.json --quiet
            """
          }
          sh """
            gsutil -m -h "Cache-Control:private, max-age=0, no-transform" rsync -d -r dist/ gs://charts.mldigital.be/kubernetes/
          """
        }
      }
    }
  }
  currentBuild.result = 'SUCCESS'
} catch (e) {
  currentBuild.result = 'FAILURE'
  throw e
} finally {
  switch (currentBuild.result) {
    case 'SUCCESS':
      slackResultColor = 'good'
      break
    case 'FAILURE':
      slackResultColor = 'danger'
      break
  }
  slackSend(color: slackResultColor, message: "<${env.JOB_DISPLAY_URL}|${env.JOB_NAME}> - build <${env.RUN_DISPLAY_URL}|${env.BUILD_DISPLAY_NAME}> result: ${currentBuild.result} (<${env.RUN_CHANGES_DISPLAY_URL}|changes>)")
}
