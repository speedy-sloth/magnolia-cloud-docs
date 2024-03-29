 pipeline {

    options {
        withAWS(region: "${env.PLATFORM_AWS_REGION}", credentials: "${env.PLATFORM_CREDENTIALS_ID}")
    }

    agent {
        label 'docker'
    }

    environment {
        AWS_REGION      = "eu-central-1"
        TF_CLI_ARGS     = "-input=false"
        PRODUCTION_S3_BUCKET  = "docs.beta.de.magnolia-cloud.com"
        OKTA_API_TOKEN   = credentials('okta/api_token')
    }

    stages {
        stage('Verify changes') {
            when {
                beforeAgent true
                anyOf {
                    branch 'master'
                    changeRequest()
                }
            }

            agent {
                docker {
                    image "$env.STB_CI_IMG_NAME:$env.STB_VERSION"
                    label "docker"
                    reuseNode true
                    alwaysPull true
                    args '-u root:root --entrypoint=\'\''
                }
            }

            steps {
                script {
                    dir('infra/') {
                        // Build lambda
                        sh "zip --junk-paths lambda.zip auth_lambda/*"
                        // Apply changes
                        sh "aws iam get-user"
                        sh "terraform init -reconfigure -backend-config='bucket=magnolia-internal-docs-infra-tfstate' -backend-config=\"role_arn=arn:aws:iam::${env.MAGNOLIA_CLOUD_STAGING_AWS_ACCOUNT_ID}:role/sre-platform\""
                        sh "terraform plan -var-file=prod.tfvars"
                    }
                }
            }
        }

        // NOTE: linting is part of UI bundle build already
        stage('Build UI bundle') {
            when {
                beforeAgent true
                anyOf {
                    branch 'master'
                    changeRequest()
                }
            }

        // For testing use same docker image above. and try the commands to get output.... 

            steps {
                script {
                    docker.withRegistry('') {
                        docker.image('node:10.22.1-alpine3.9').inside('-u root:root --entrypoint=\'\'') {
                            echo 'Building UI bundle ...'
                            dir('ui/') {
                                sh 'apk add build-base libtool automake autoconf nasm zlib'
                                sh 'rm -rf build'
                                sh 'npm install'
                                sh 'npm install gulp-cli'
                                sh './node_modules/.bin/gulp bundle'
                            }
                        }
                    }
                }
            }
        }


        stage('Build Antora site') {
            when {
                beforeAgent true
                anyOf {
                    branch 'master'
                    changeRequest()
                }
            }

            steps {
                script {
                    docker.withRegistry('') {
                        docker.image('antora/antora:2.3.4').inside {
                          withCredentials([string(credentialsId: 'BITBUCKET_ACCESS_TOKEN', variable: 'BITBUCKET_ACCESS_TOKEN')]) {
                              sh "echo https://sre.robot:${BITBUCKET_ACCESS_TOKEN}@git.magnolia-cms.com >> ~/.git-credentials"
                              echo 'Building Antora documentation ...'
                              sh 'npm install'
                              sh 'rm -rf build/site'
                              sh 'npm install'
                              withCredentials([usernamePassword(credentialsId: 'NEXUS_CREDENTIALS', usernameVariable: 'NEXUS_USERNAME', passwordVariable: 'NEXUS_PASSWORD')]) {
                                sh 'antora generate --fetch playbook.yml'
                              }
                          }
                        }
                    }
                }
            }
        }

        stage('Deploy Infra in Production') {
            when {
                beforeAgent true
                branch 'master'
            }

            agent {
                docker {
                    image "$env.STB_CI_IMG_NAME:$env.STB_VERSION"
                    label "docker"
                    reuseNode true
                    alwaysPull true
                    args '-u root:root --entrypoint=\'\''
                }
            }

            steps {
                dir('infra/') {
                    sh "terraform init -reconfigure -backend-config='bucket=magnolia-internal-docs-infra-tfstate' -backend-config=\"role_arn=arn:aws:iam::${env.MAGNOLIA_CLOUD_STAGING_AWS_ACCOUNT_ID}:role/sre-platform\""
                    sh "terraform apply -var-file=prod.tfvars -auto-approve"
                }
            }
        }

        stage('Deploy Antora site to S3 in Production') {
            when {
                beforeAgent true
                branch 'master'
            }

            agent {
                docker {
                    image "$env.STB_CI_IMG_NAME:$env.STB_VERSION"
                    label "docker"
                    reuseNode true
                    alwaysPull true
                    args '-u root:root --entrypoint=\'\''
                }
            }

            steps {
                withAWS(role: "sre-platform", roleAccount: "${env.MAGNOLIA_CLOUD_STAGING_AWS_ACCOUNT_ID}", region: "${env.AWS_REGION}") {
                    sh "aws s3 cp build/site/ s3://${env.PRODUCTION_S3_BUCKET}/ --recursive"
                    sh "aws s3 sync build/site/ s3://${env.PRODUCTION_S3_BUCKET}/ --delete"
                }
            }
        }
    }
}

def setBuildName() {
    currentBuild.displayName = "${env.BUILD_NUMBER}-prod"
}
