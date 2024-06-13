pipeline {
    agent any

    parameters {
        booleanParam(name: 'autoApprove', defaultValue: false, description: 'Automatically run apply after generating plan?')
        choice(name: 'action', choices: ['apply', 'destroy'], description: 'Select the action to perform')
    }

    environment {
        AWS_REGION = 'us-west-1' // Update with your AWS region
    }

    stages {
        stage('Assume Jenkins Role for Terraform') {
            steps {
                script {
                    // Assume the jenkins-role-for-waqas-user role using AWS CLI
                    def assumeRoleOutput = sh(
                        script: 'aws sts assume-role --role-arn arn:aws:iam::372666185803:role/jenkins-role-for-waqas-user --role-session-name jenkins-session',
                        returnStdout: true
                    ).trim()
                    echo "Assumed Role Output:"
                    echo assumeRoleOutput  // Print the assume role output for debugging

                    def assumeRoleJson = new groovy.json.JsonSlurper().parseText(assumeRoleOutput)
                    env.AWS_ACCESS_KEY_ID = assumeRoleJson.Credentials.AccessKeyId
                    env.AWS_SECRET_ACCESS_KEY = assumeRoleJson.Credentials.SecretAccessKey
                    env.AWS_SESSION_TOKEN = assumeRoleJson.Credentials.SessionToken

                    echo "Assumed Role Credentials:"
                    echo "Access Key ID: ${env.AWS_ACCESS_KEY_ID}"
                    echo "Secret Access Key: ${env.AWS_SECRET_ACCESS_KEY}"
                    echo "Session Token: ${env.AWS_SESSION_TOKEN}"
                }
            }
        }

        stage('Check Repository') {
            steps {
                git branch: 'main', credentialsId: 'git_lemp_new', url: 'https://github.com/WaQQass/tf.lemp_sts_ec2.git'
                sh 'ls -la' // List all files in the current directory
            }
        }

        stage('Terraform Init') {
            steps {
                dir('terraform') {
                    sh 'terraform init'
                }
            }
        }

        stage('Plan') {
            steps {
                dir('terraform') {
                    sh 'terraform plan -out=tfplan'
                    sh 'terraform show -no-color tfplan > tfplan.txt'
                }
            }
        }

        stage('Apply / Destroy') {
            steps {
                script {
                    if (params.action == 'apply') {
                        if (!params.autoApprove) {
                            def plan = readFile 'terraform/tfplan.txt'
                            input message: "Do you want to apply the plan?",
                                  parameters: [text(name: 'Plan', description: 'Please review the plan', defaultValue: plan)]
                        }
                        dir('terraform') {
                            sh 'terraform apply -input=false tfplan'
                            sleep 30 // Wait for 30 seconds to ensure instance creation

                            // Capture the outputs
                            def outputs = sh(script: 'terraform output -json', returnStdout: true).trim()
                            echo "Terraform Outputs: ${outputs}"
                            writeFile file: 'terraform/outputs.json', text: outputs

                            // Parse JSON output and set environment variables if needed
                            def outputJson = new groovy.json.JsonSlurper().parseText(outputs)
                            env.INSTANCE_ID = outputJson.instance_id.value
                            env.PUBLIC_IP = outputJson.public_ip.value
                        }
                    } else if (params.action == 'destroy') {
                        dir('terraform') {
                            sh 'terraform destroy --auto-approve'
                        }
                    } else {
                        error "Invalid action selected. Please choose either 'apply' or 'destroy'."
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                if (params.action == 'apply') {
                    // Trigger the second job after the first job completes successfully and the action chosen is 'apply'
                    build job: 'Check-EC2-Status-and-IP', wait: false, parameters: [
                        string(name: 'INSTANCE_ID', value: "${env.INSTANCE_ID}"),
                        string(name: 'PUBLIC_IP', value: "${env.PUBLIC_IP}")
                    ]
                } else {
                    echo "Second pipeline will not be triggered as 'destroy' action was chosen."
                }
            }
        }
    }
}
