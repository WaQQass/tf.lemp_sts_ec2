pipeline {
    agent any

    parameters {
        booleanParam(name: 'autoApprove', defaultValue: false, description: 'Automatically run apply after generating plan?')
        choice(name: 'action', choices: ['apply', 'destroy'], description: 'Select the action to perform')
    }

    environment {
        AWS_REGION = 'us-west-1' // Update region to match your Terraform configuration
    }

    stages {
        stage('Check Repository') {
            steps {
                git branch: 'main', credentialsId: 'git_lemp_new', url: 'https://github.com/WaQQass/tf.lemp_sts_ec2.git'
                sh 'ls -la' // List all files in the current directory
            }
        }
        stage('Assume Jenkins Role for Terraform') {
            steps {
                script {
                    // Assume the jenkins-role-for-terraform-ec2 role using AWS CLI
                    def assumeRoleOutput = sh(
                        script: 'aws sts assume-role --role-arn arn:aws:iam::372666185803:role/jenkins-role-for-terraform-ec2 --role-session-name jenkins-session',
                        returnStdout: true
                    ).trim()
                    def assumeRoleJson = readJSON(text: assumeRoleOutput)
                    env.AWS_ACCESS_KEY_ID = assumeRoleJson.Credentials.AccessKeyId
                    env.AWS_SECRET_ACCESS_KEY = assumeRoleJson.Credentials.SecretAccessKey
                    env.AWS_SESSION_TOKEN = assumeRoleJson.Credentials.SessionToken
                }
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
                            // Example:
                            def outputJson = readJSON text: outputs
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
}
