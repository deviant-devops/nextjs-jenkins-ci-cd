/* groovylint-disable CompileStatic */
// Sample Jenkins file template
pipeline {
    agent {
        dockerContainer { image 'jenkins/agent:latest-bookworm' }
        // dockerfile true
        // Equivalent to "docker build -f Dockerfile.build --build-arg version=1.0.2 ./build/
        // dockerfile {
        //     filename 'Dockerfile.build'
        //     dir 'build'
        //     label 'my-defined-label'
        //     additionalBuildArgs  '--build-arg version=1.0.2'
        //     args '-v /tmp:/tmp'
        // }
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
        stage('Run Jest') {
            steps {
                sh 'npm test'
            }
        }
    }
}
