# terraform_with_jenkins

pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        TF_VERSION = '1.5.0'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 1, unit: 'HOURS')
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                git(
                    branch: 'main',
                    url: 'https://github.com/Sandiip007/terraform_with_jenkins.git'
                )

                echo 'Code checked out successfully.'
            }
        }

        stage('Check Files') {
            steps {
                sh '''
                    echo "Current directory:"
                    pwd

                    echo "Files:"
                    ls -la

                    echo "Terraform files:"
                    find . -name "*.tf" -type f
                '''
            }
        }

        stage('Terraform Init') {
            steps {
                sh 'terraform init -reconfigure -no-color'
            }
        }

        stage('Terraform Validate & Format') {
            steps {
                sh 'terraform validate -no-color'
                sh 'terraform fmt -check -recursive -no-color'
            }
        }

        stage('Terraform Plan') {
            steps {
                sh 'terraform plan -out=tfplan -no-color'
                sh 'terraform show -no-color tfplan'
            }
        }

        stage('Manual Approval') {
            steps {
                input(
                    message: 'Approve Terraform Apply?',
                    ok: 'Yes, Apply'
                )
            }
        }

        stage('Terraform Apply') {
            steps {
                sh 'terraform apply -auto-approve tfplan'
            }
        }
    }

    post {
        always {
            script {
                if (fileExists('tfplan')) {
                    sh 'rm -f tfplan'
                }
            }

            echo "Pipeline finished with status: ${currentBuild.currentResult}"
        }

        success {
            echo 'Terraform pipeline completed successfully.'
        }

        failure {
            echo 'Terraform pipeline failed. Check the logs for details.'
        }
    }
}
