pipeline {
    agent any
    environment {
          APP_NAME = "devops"
    }
    stages {
         stage("Cleanup Workspace") {
             steps {
                cleanWs()
             }
         }
         stage("Checkout from Git") {
             steps {
                     git branch: 'main', credentialsId: 'github', url: 'https://github.com/tien22521469/devops-gitops'
             }
         }
         stage("Update the Deployment Tags") {
            steps {
                sh """
                    cat deployment.yaml
                    sed -i 's/${APP_NAME}.*/${APP_NAME}:${IMAGE_TAG}/g' deployment.yaml
                    cat deployment.yaml
                """
            }
         }
         stage("Push the changed deployment file to GitHub") {
            steps {
                sh """
                    git config --global user.name "tien22521469"
                    git config --global user.email "22521469@gm.uit.edu.vn"
                    git add deployment.yaml
                    git commit -m "Updated Deployment Manifest"
                """
                withCredentials([gitUsernamePassword(credentialsId: 'github', gitToolName: 'Default')]) {
                    sh "git push https://github.com/tien22521469/devops-gitops main"
                }
            }
         }
    }
}
