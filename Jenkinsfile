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

        stage('Get Repo Information') {
             // Extract owner and repo name from GIT_URL
            steps {
                script {
                    def gitUrl = env.GIT_URL
                    def repoInfo = gitUrl.split('/')[-2..-1].join('/').replace('.git', '')
                    
                    env.REPO_OWNER = repoInfo.split('/')[0]
                    env.REPO_NAME = repoInfo.split('/')[1]
                    env.REPO_INFO = repoInfo

                    echo "Repository owner: ${env.REPO_OWNER}"
                    echo "Repository name: ${env.REPO_NAME}"
                    echo "Repository: ${env.REPO_INFO}"
                }   
            }
        }

        stage('Add Comment to Pull Request') {
            when { // Only run steps if pull request
                branch 'PR-*'
            }
            steps {
                script {
                    // Use withCredentials to pass username and password credentials
                    withCredentials([usernamePassword(credentialsId: 'deviant-devops',
                        passwordVariable: 'PASSWORD',
                        usernameVariable: 'USERNAME')]) {
                        
                        def commentMessage = 'Comment from Jenkins!'
                        // Inside this block, you can use USERNAME and PASSWORD variables
                        // For example:
                        sh "echo $PASSWORD | gh auth login --with-token"
                        sh "gh pr comment ${env.CHANGE_ID} --body '${commentMessage}' --repo ${env.REPO_INFO}"
                    }
                }
            }
        }

        stage('Build and Push Docker Image to Docker Hub') {
            when { // Only run steps if pull request
                branch 'MAIN'
            }
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-deviantdevops', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        // Log in to Docker Hub
                        sh "echo $DOCKER_PASSWORD | docker login -u ${DOCKER_USERNAME} --password-stdin"
                        def imageName = "${DOCKER_USERNAME}/${env.REPO_NAME}:latest"
                        sh "docker build -t ${imageName} ."
                        sh "docker push ${imageName}"
                    }
                }
            }
        }
    }
}
