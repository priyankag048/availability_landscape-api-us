pipeline {
  agent any
  environment {
    HELM_KUBEAPISERVER='http://host.docker.internal:8000'
  }
  stages {
    stage("Prepare") {
      steps {
        checkout([$class: 'GitSCM',
          branches: [[name: "origin/main"]],
          doGenerateSubmoduleConfigurations: false,
          extensions: [[$class: 'LocalBranch']],
          submoduleCfg: []])
      }
    }
    stage("deploy") {
      steps {
        script {
          docker.image('priyankaguha/packaged-software:2').inside('-u root') {
            withCredentials([[$class: 'VaultTokenCredentialBinding', credentialsId: 'availability-key', vaultAddr: 'http://host.docker.internal:8200', vaultNamespace: 'availability-asia'  ]]) {
              sh '''
                RESPONSE=$(curl -H "X-Vault-Token: $VAULT_TOKEN" -L http://host.docker.internal:8200/v1/secret/data/availability-us?version=1)
                KEY=$(echo -n $RESPONSE | jq -r '.data.data."encryption-key"')
                openssl enc -d -aes-256-cbc -a -md sha512 -pbkdf2 -iter 100000 -salt -k ${KEY} -in values.txt.enc -out values.yaml
              '''
            }
            withCredentials([file(credentialsId: 'kubeConfig', variable: 'KUBECONFIG')]) {
              sh '''
                helm package ./availability_service/helm-chart -d helm-logs
                helm upgrade --install --atomic --wait availability helm-logs/availability-chart-0.1.0.tgz -f values.yaml 
              '''
            }
          }
        }
      }
    }
  }
  post {
    always {
      cleanWs(
          deleteDirs: true
        )
    }
  }
}