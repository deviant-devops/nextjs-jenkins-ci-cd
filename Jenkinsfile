pipeline {
    /**
    TODO: 
    - Update package.json version when updating semver git tag
     */
    agent {
        // dockerContainer { image 'jenkins/agent:latest-bookworm' }
        // dockerfile true

        // Equivalent to "docker build -f Dockerfile.build --build-arg version=1.0.2 ./build/
        dockerfile {
            //     filename 'Dockerfile.build'
            //     dir 'build'
            //     label 'my-defined-label'
            //     additionalBuildArgs  '--build-arg version=1.0.2'
            //     args '-v /tmp:/tmp'
            filename 'Dockerfile.ci'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    stages {

        stage('Check Build ID') {
            steps {
                echo "Running ${env.BUILD_ID} on ${env.JENKINS_URL}, PR-${env.CHANGE_ID}"
            }
        }

        stage('Get Repo Information') {
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

        stage('Check Docker Version') {
            steps {
                sh 'docker --version'
            }
        }

        /**  Continuous Integration Section - Linting and Error Checking **/
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
        
        /**  Continuous Delivery Section - Merging, Tagging, Releasing **/

        stage('Authenticate GitHub CLI') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'deviant-devops',
                        passwordVariable: 'PASSWORD',
                        usernameVariable: 'USERNAME')]) {
                        sh 'echo $PASSWORD | gh auth login --with-token' // use single quotes for sensitive info like passwords or use triple quotes """
                    }
                }
            }
        }
         
        stage('Add Comment to Pull Request') {
            when {
                branch 'PR-*'
                // same as 
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
            }
            steps {
                script {
                    sh "gh pr merge ${env.CHANGE_ID} --merge --delete-branch --repo ${env.GIT_URL}"
                }
            }
        }

        stage('Determine New Version') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh "git fetch --tags"
                    
                    // Get the commit hash of the latest tag
                    def latestTagCommit = sh(script: "git rev-list --tags --max-count=1", returnStdout: true).trim()

                    // Get the latest commit hash in the current branch
                    def latestCommit = sh(script: "git rev-parse HEAD", returnStdout: true).trim()

                    // Get the latest tag
                    // run "git ls-remote --tags origin | grep -Eo 'v[0-9.]+\$'" to get all versions
                    def latestTag = sh(script: "git tag -l --sort=-creatordate | head -n 1", returnStdout: true).trim()

                    echo "Latest tag: ${latestTag}"
                    echo "Latest tag commit: ${latestTagCommit}"
                    echo "Latest commit: ${latestCommit}"

                    if (latestTagCommit == latestCommit) {
                        echo "No new commits since the latest tag. Skipping version update."
                        currentBuild.result = 'ABORTED'
                        error("Aborting the build.")
                    }

                    // Parse the latest version, note that tags starts with v (example v0.1.0)
                    def (major, minor, patch) = latestTag.replace("v", "").tokenize('.')

                    // Determine the new version (example: increment patch)
                    patch = patch.toInteger() + 1

                    // Define the new tag name
                    newVersion = "v${major}.${minor}.${patch}"

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
                    def prList = sh(script: """
                        gh pr list --repo ${env.GIT_URL} --state merged --json number,headRefName,title
                    """, returnStdout: true).trim()

                    def prJson = readJSON(text: prList)

                    // Format release notes
                    def releaseNotes = "## Release ${env.NEW_IMAGE_TAG}\n\n"
                    releaseNotes += prJson.collect { pr ->
                        return "- PR #${pr.number}: ${pr.title} (${pr.headRefName})"
                    }.join('\n')

                    env.VERSION_NOTES = releaseNotes
                    echo "Generated Release Notes:\n${env.VERSION_NOTES}"
                }
            }
        }

        stage('Create Git Tag with Release Notes') {
            when {
                branch 'main'
            }
            steps {
                script {
                    // Check if the tag already exists
                    def tagExists = sh(script: "git ls-remote --tags origin | grep -w refs/tags/${env.NEW_IMAGE_TAG}", returnStatus: true) == 0

                    if (!tagExists) {
                        sh """
                            git tag -a -f ${env.NEW_IMAGE_TAG} -m "${env.VERSION_NOTES}"
                        """
                    } 

                    withCredentials([usernamePassword(credentialsId: 'deviant-devops',
                        passwordVariable: 'PASSWORD',
                        usernameVariable: 'USERNAME')]) {
                            sh """
                                git push https://$USERNAME:$PASSWORD@github.com/${env.REPO_INFO}.git ${env.NEW_IMAGE_TAG} -f
                            """ 
                        }
                    
                    sh """
                        gh release create ${env.NEW_IMAGE_TAG} \
                            --notes "${env.VERSION_NOTES}" \
                            --repo ${env.GIT_URL}
                    """
                }
            }
        }

        stage('Build and Push Docker Image to Docker Hub') {
            when {
                branch 'MAIN'
            }
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-deviantdevops', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh 'echo $DOCKER_PASSWORD | docker login -u ${DOCKER_USERNAME} --password-stdin'
                        def imageName = "${DOCKER_USERNAME}/${env.REPO_NAME}:latest"
                        def imageName2 = "${DOCKER_USERNAME}/${env.REPO_NAME}:${env.NEW_IMAGE_TAG}"

                        env.IMAGE_NAME = "${DOCKER_USERNAME}/${env.REPO_NAME}"

                        sh "docker build -t ${imageName} ."
                        sh "docker build -t ${imageName2} ."
                        sh "docker push ${imageName}"
                        sh "docker push ${imageName2}"
                    }
                }
            }
        }

        stage('Update Deployment Repo') {
            when {
                branch 'main'
            }
            steps {
                script {
                    // Clone the deployment repository
                    withCredentials([usernamePassword(credentialsId: 'deviant-devops',
                        passwordVariable: 'PASSWORD',
                        usernameVariable: 'USERNAME')]) {
                        
                        env.DEPLOYMENT_GIT_REPO_URL = "https://github.com/deviant-devops/nextjs-jenkins-deployment.git"
                        env.DEPLOYMENT_FILE_PATH = "deployment.yaml"

                        sh """
                            git clone ${env.DEPLOYMENT_GIT_REPO_URL}
                            cd \$(basename ${env.DEPLOYMENT_GIT_REPO_URL} .git)
                        """
                    }
                    
                    // Update the deployment YAML file
                    def repoName = env.DEPLOYMENT_GIT_REPO_URL.tokenize('/').last().replace('.git', '')
                    def deploymentYaml = readFile "${repoName}/${env.DEPLOYMENT_FILE_PATH}"
                    def newYaml = deploymentYaml.replaceAll("${env.IMAGE_NAME}:.*", "${env.IMAGE_NAME}:${env.NEW_IMAGE_TAG}")
                    writeFile file: "${repoName}/${env.DEPLOYMENT_FILE_PATH}", text: newYaml

                    // Commit and push the changes
                    withCredentials([usernamePassword(credentialsId: 'deviant-devops',
                        passwordVariable: 'PASSWORD',
                        usernameVariable: 'USERNAME')]) {
                        sh """
                            cd ${repoName}
                            git add ${env.DEPLOYMENT_FILE_PATH}
                            git commit -m "Update image tag to ${env.NEW_IMAGE_TAG}"
                            git push --repo=https://$USERNAME:$PASSWORD@github.com/deviant-devops/nextjs-jenkins-deployment.git main
                        """
                    }
                }
            }
        }
    }
    post {
        always {
            // Clean workspace after the pipeline run
            cleanWs()
        }
    }
}
