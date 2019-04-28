pipeline {
    agent any
    tools{
        maven "Maven"
    }
    stages {
        stage('Git Checkout Slave') {
            agent{label 'gceslave0'}
            steps {
                git branch: 'master',
                url: 'https://github.com/Tailp/CICDpipeline'
            }
        }
        stage('Maven Build') {
            agent{label 'gceslave0'}
            steps {
                sh "mvn -Dmaven.test.skip=true install"
            }
        }
        stage('Maven Test') {
            agent{label 'gceslave0'}
            steps {
                sh "mvn test"
            }
        }
        stage('Git Checkout Master') {
            agent{label 'master'}
            steps {
                git branch: 'master',
                url: 'https://github.com/Tailp/CICDpipeline'
            }
        }
        stage('Deploy'){
            agent{label 'master'}
            steps {
                withEnv(['GCLOUD_PATH=/home/tailp/google-cloud-sdk/bin']) {
                    sh '$GCLOUD_PATH/gcloud auth activate-service-account --key-file=/home/tailp/jenkinscrt/Jsonkeyfile.json'
                    sh '$GCLOUD_PATH/gcloud container clusters get-credentials ClusterName --zone europe-west2-c --project ProjectID'
                    sh '$GCLOUD_PATH/gcloud config set project ProjectID'
                    sh '$GCLOUD_PATH/gcloud builds submit --tag gcr.io/ProjectID/m-ci:latest .'
                }
                withEnv(['KUBECTL_PATH=/usr/bin']) {
                    sh '$KUBECTL_PATH/kubectl run mci --image=gcr.io/ProjectID/m-ci:latest --port 8080'
                    sh '$KUBECTL_PATH/kubectl expose deployment mci --type=LoadBalancer --port 8000 --target-port 8000'
                }
            }
        }
        stage ("Wait for checking") {
            steps {
                echo "Waiting 150 seconds for checking"
                sleep 150// seconds
            }
        }
        stage('Clean up') {
            agent{label 'master'}
            steps {
                withEnv(['KUBECTL_PATH=/home/tailp/google-cloud-sdk/bin']) {
                    sh '$KUBECTL_PATH/kubectl delete svc mci'
                    sh '$KUBECTL_PATH/kubectl delete deployment mci'
                }
            }
            
        }
    }
}
