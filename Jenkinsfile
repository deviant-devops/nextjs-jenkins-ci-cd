/* groovylint-disable CompileStatic */
// Sample Jenkins file template
pipeline {
    agent {
        // dockerContainer { image 'jenkins/agent:latest-bookworm' }
        // dockerfile true
        // Equivalent to "docker build -f Dockerfile.build --build-arg version=1.0.2 ./build/
        dockerfile {
            filename 'Dockerfile.ci'
        //     dir 'build'
        //     label 'my-defined-label'
        //     additionalBuildArgs  '--build-arg version=1.0.2'
        //     args '-v /tmp:/tmp'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    stages {
        stage('Check Node version') {
            steps {
                sh 'node --version'
            }
        }

        stage('Check NodeJS configuration') {
            steps {
                sh 'npm config ls'
            }
        }

        stage('Install dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Run Lint') {
            steps {
                sh 'npm run lint'
            }
        }
        // stage('Run Jest') {
        //     steps {
        //         sh 'npm test'
        //     }
        // }
        stage('Check Docker Version') {
            steps {
                sh 'docker --version'
            }
        }

        stage('Check Build ID') {
            steps {
                echo "Running ${env.BUILD_ID} on ${env.JENKINS_URL}, PR-${env.CHANGE_ID}"
            }
        }

        stage('Pull Request - merge if successful') {
            when {
                branch 'PR-*'
            }
            steps {
                // Use withCredentials to pass username and password credentials
                script {
                    withCredentials([usernamePassword(credentialsId: 'deviant-devops',
                        passwordVariable: 'PASSWORD',
                        usernameVariable: 'USERNAME')]) {
                        
                        def commentMessage = 'Comment from Jenkins!'
                        def GIT_REPO_NAME = env.GIT_URL.replaceFirst(/^.*\/([^\/]+?).git$/, '$1')
                        def GIT_USER_REPO_NAME = env.GIT_URL.replaceFirst(/^.*?(?::\/\/.*?\/|:)(.*).git$/. '$1')
                        // Inside this block, you can use USERNAME and PASSWORD variables
                        // For example:
                        sh "echo Username is $USERNAME"
                        sh "echo Password is $PASSWORD"
                        sh "echo $PASSWORD | gh auth login --with-token"
                        sh "echo ${GIT_REPO_NAME}"
                        sh "echo ${GIT_USER_REPO_NAME}"
                        sh "gh pr comment ${env.CHANGE_ID} --body '${commentMessage}' --repo $USERNAME/$GIT_REPO_NAME"
                    }
                }
            }
        }
    }
}
