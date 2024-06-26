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
        stage('Print environment variables') {
            steps {
                echo sh(script: 'env|sort', returnStdout: true)
            }
        }
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

        stage('Authenticate with GitHub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'deviant-devops',
                        passwordVariable: 'PASSWORD',
                        usernameVariable: 'USERNAME')]) {
                        sh 'echo $PASSWORD | gh auth login --with-token' // use single quotes for sensitive info like passwords
                    }
                }
            }
        }

        stage('Add Comment to Pull Request') {
            when { // Only run steps if pull request
                branch 'PR-*'
                // expression { env.BRANCH_NAME.startsWith('PR-') }
            }
            steps {
                script {
                    def commentMessage = "Finished Testing PR-${env.CHANGE_ID}. All successful, will now merge PR with main."
                    sh "gh pr comment ${env.CHANGE_ID} --body '${commentMessage}' --repo ${env.REPO_INFO}"
                }
            }
        }

        stage('Merge Pull Request') {
            when {
                branch 'PR-*'
                // expression { env.BRANCH_NAME.startsWith('PR-') }
            }
            steps {
                script {
                    // def prNumber = env.BRANCH_NAME.split('-')[1]
                    // sh """
                    //     gh pr merge ${prNumber} --merge --delete-branch --repo ${env.MAIN_GIT_REPO_URL}
                    // """
                    sh "gh pr merge ${env.CHANGE_ID} --merge --delete-branch --repo ${env.GIT_URL}"
                }
            }
        }

        stage('Generate New SemVer Tag') {
            when {
                branch 'main'
            }
            steps {
                script {
                    def currentVersion = sh(script: """
                        gh release list --repo ${env.GIT_URL} --limit 1 --json tagName --jq '.[0].tagName'
                    """, returnStdout: true).trim() == '0'
                    
                    def newVersion

                    if (currentVersion) {
                        def (major, minor, patch) = currentVersion.tokenize('.')
                        newVersion = "${major}.${minor}.${(patch.toInteger() + 1)}"
                    } else {
                        // Set the initial version
                        newVersion = "0.1.0"
                    }

                    env.NEW_IMAGE_TAG = newVersion
                    echo "New Version: ${env.NEW_IMAGE_TAG}"
                }
            }
        }

        stage('Generate Release Notes') {
            when {
                branch 'main'
            }
            steps {
                script {
                    // Get the merged PRs since the last release
                    // gh pr list --repo ${env.GIT_URL} --state merged --json number,headRefName,title --jq '.[] | {number, headRefName, title}'
                    def prList = sh(script: """
                        gh pr list --repo ${env.GIT_URL} --state merged --json number,headRefName,title
                    """, returnStdout: true).trim()

                    echo prList
                    def prJson = readJSON(text: prList)
                    print prJson
                    prJson.each { pr ->
                       echo "PR Title: ${pr.title}"
                    }

                    // // Format release notes
                    // def releaseNotes = "## Release ${env.NEW_IMAGE_TAG}\n\n"
                    // releaseNotes += prJson.each { pr ->
                    //     return "- PR #${pr.number}: ${pr.title} (${pr.headRefName})"
                    // }.join('\n')

                    // env.VERSION_NOTES = releaseNotes
                    // echo "Generated Release Notes:\n${env.VERSION_NOTES}"
                }
            }
        }

        stage('Create Git Tag with Release Notes') {
            when {
                branch 'main'
            }
            steps {
                script {
                    // Tag the Git repository with release notes
                    sh """
                        git tag -a ${env.NEW_IMAGE_TAG} -m "${env.VERSION_NOTES}"
                        git push origin ${env.NEW_IMAGE_TAG}
                    """

                    // Create a GitHub release with release notes
                    sh """
                        gh release create ${env.NEW_IMAGE_TAG} \
                            --notes "${env.VERSION_NOTES}" \
                            --repo ${env.GIT_URL}
                    """
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
                        sh 'echo $DOCKER_PASSWORD | docker login -u ${DOCKER_USERNAME} --password-stdin' // use single quotes for sensitive info like passwords
                        def imageName = "${DOCKER_USERNAME}/${env.REPO_NAME}:latest"
                        def imageName2 = "${DOCKER_USERNAME}/${env.REPO_NAME}:v${env.NEW_IMAGE_TAG}"
                        sh "docker build -t ${imageName} ."
                        sh "docker build -t ${imageName2} ."
                        sh "docker push ${imageName}"
                        sh "docker push ${imageName2}"
                    }
                }
            }
        }
    }
}
