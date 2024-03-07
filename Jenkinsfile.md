```bash

#!/usr/bin/env groovy

@Library('tools')
import com.jamf.jenkins.KanikoBuild

//branch = branch()
//release = branch == 'stage'
//repository = release ? 'ga' : 'test'
//buildtype = 'dev'

pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '3')) // Keeps only the last 3 builds
    }
    parameters {
        choice(
            choices: ['euw2', 'use1'],
            description: 'Select the Kubernetes region for deployment.',
            name: 'REGION'
        )
        string(
            defaultValue: 'cicd',
            description: 'Updates values file using cicd branch in k8s manifest repo. Developers must create PRs to merge changes from cicd to non-prod and then to prod branches.',
            name: 'TARGET_ENVIRONMENT'
        )
        booleanParam(
            defaultValue: true,
            description: 'Automatically deploy to all regions for sbox, dev, and stage environments. Not applicable for prod.',
            name: 'DEPLOY_TO_ALL_REGIONS'
        )
    }

    environment {
        PROJECT = 'datajar-portal'
        SLACK_CHANNEL = "djk8s"
        SONARPROJECT = "com.jamf.datajar"
        SONAR_URL = "https://sonarqube.jamf.build"
        IMAGE_TAG = "${env.GIT_COMMIT.take(7)}"
        HELM_MANIFEST_REPO = "git@github.com:jamf/datajar-k8s-manifests.git"
        HELM_REPO_BRANCH = "cicd"
        VALUES_PATH_WEB = "portal/web"
        VALUES_PATH_QUEUE = "portal/queue"
        BUILDTYPE = 'dev'
        GIT_COMMITTER_EMAIL = "django@jamf.com"
        GIT_COMMITTER_NAME = "DJango Automation"
        ALL_REGIONS = "euw2,use1" // Store regions as a comma-seperated string
    }

    stages {
        // Preparation
        stage('Preparation') {
            steps {
                script {
                    // Branch logic to determine the repository
                    def branch = env.BRANCH_NAME
                    def release = branch == 'prod'
                    def repository = release ? 'ga' : 'test'
                    env.REPOSITORY = repository // Setting the repository environment variable
                }
            }
        }
        //Build images
        stage('Build Web Image') {
            when {
                anyOf { branch 'dev'; branch 'prod'; branch 'stage-new-1' }
            }
            agent {
                kubernetes {
                    yaml KanikoBuild.getDefaultAgentYaml()
                }
            }
            steps {
                script {
                    kanikoBuildImage('dev/web/Dockerfile', 'web-portal', "${env.IMAGE_TAG}", "${env.BUILDTYPE}")
                }
            }
        }
    
        stage('Build Queue Image') {
            when {
                anyOf { branch 'dev'; branch 'prod'; branch 'stage-new-1' }
            }
            agent {
                kubernetes {
                    yaml KanikoBuild.getDefaultAgentYaml()
                }
            }
            steps {
                script {
                    kanikoBuildImage('dev/queue/Dockerfile', 'queue-portal', "${env.IMAGE_TAG}", "${env.BUILDTYPE}")
                }
            }
        } 
        stage('Deploy to Sbox and Dev') {
            when {
                //branch 'dev' // Ensures this stage runs only for the 'dev' branch
                branch 'stage-new' //
            }
            agent {
                kubernetes {
                    defaultContainer 'aws'
                    yaml '''
                    apiVersion: v1
                    kind: Pod
                    spec:
                      serviceAccountName: kube-team-jenkins
                      podRetention: Never
                      containers:
                      - name: aws
                        image: docker.jamf.build/engineering/ci/jamf-remote-assist-deployment:latest
                        tty: true
                        command:
                        - cat
                    '''
                }
            }
            steps {
                script {
                    def regionsList = params.DEPLOY_TO_ALL_REGIONS ? env.ALL_REGIONS.tokenize(',') : [params.REGION]
                    ['sbox', 'dev'].each { envName ->
                        sshagent(['django-github-k8s']) {
                            container('aws') {
                                regionsList.each { region ->
                                    echo "Deploying ${envName} to region ${region}"
                                    deployToEnvironment(envName, region)
                                    echo "please let me sleep for 30 seconds more"
                                    sleep(time:30, unit: "SECONDS")
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('Wait for Agent') {
            steps {
                script {
                    // Wait for the Kubernetes pod to be ready
                    sleep(time: 30, unit: 'SECONDS')
                }
            }
        }       
        
        // Start Stage Helm chart updates
        stage('Deploy to Stage') {
           // Only run this stage for specific branch names or patterns
            when { 
                    expression {
                    return env.BRANCH_NAME == 'stage-new'
                    }
            }
            agent {
                kubernetes {
                    defaultContainer 'aws'
                    yaml '''
                    apiVersion: v1
                    kind: Pod
                    spec:
                        serviceAccountName: kube-team-jenkins
                        podRetention: Never
                        containers:
                        - name: aws
                        image: docker.jamf.build/engineering/ci/jamf-remote-assist-deployment:latest
                        tty: true
                        command:
                        - cat
                    '''
                }
            }
            steps {
                script {
                    sshagent(['django-github-k8s']) {
                        container('aws') {
                            def regionsList = params.DEPLOY_TO_ALL_REGIONS ? env.ALL_REGIONS.tokenize(',') : [params.REGION]
                            regionsList.each { region ->
                                echo "Deploying to stage in region ${region}"
                                deployToEnvironment('stage', region)
                            }
                        }
                    }
                }
            }
        }
        // Start Prod Helm chart updates
        stage('Promote to Prod') {
            when {
                branch 'prod' // Execute this stage only if the branch is 'prod'
            }
                agent {
                    kubernetes {
                        defaultContainer 'aws'
                        yaml '''
                        apiVersion: v1
                        kind: Pod
                        spec:
                          serviceAccountName: kube-team-jenkins
                          podRetention: Never
                          containers:
                          - name: aws
                            image: docker.jamf.build/engineering/ci/jamf-remote-assist-deployment:latest
                            tty: true
                            command:
                            - cat
                        '''
                    }
                }
            steps {
                input(message: "Ready to promote to 'prod' in selected region?", ok: 'Proceed')
                sshagent(['django-github-k8s']) {
                    container('aws') {
                        script {
                            deployToEnvironment('prod', params.REGION)
                        }
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            when {
                allOf {
                    branch 'main'
                    not { changeRequest() }
                }
            }
            agent {
                kubernetes {
                    defaultContainer 'sonar'
                    yaml '''
                    apiVersion: v1
                    kind: Pod
                    spec:
                      serviceAccountName: kube-team-jenkins
                      podRetention: Never
                      containers:
                      - name: sonar
                        image: docker.jamf.build/sonarsource/sonar-scanner-cli:latest
                        tty: true
                        command:
                        - cat
                    '''
                }
            }
            steps {
                container('sonar') {
                    script {
                        performSonarQubeAnalysis()
                    }
                }
            }
        }
    }

    post {
        always {
            publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: false,
                reportDir: 'build/reports/tests/test',
                reportFiles: 'index.html',
                reportName: 'Gradle Build Report',
                keepAll: true
            ])
        }
    }
}


// Define all the utility functions

def setupGitConfig() {
    echo 'Setting up git configuration'
    env.GIT_SSH_COMMAND = 'ssh -o StrictHostKeyChecking=no'
    sh """
    git config --global user.email "${env.GIT_COMMITTER_EMAIL}"
    git config --global user.name "${env.GIT_COMMITTER_NAME}"
    """
}

def cloneRepository(String repoDirectory) {
    try {
        echo "Cloning repository from ${env.HELM_MANIFEST_REPO} into ${repoDirectory}"
        
        sh """
        set -x
        git clone --single-branch --branch ${env.HELM_REPO_BRANCH} ${env.HELM_MANIFEST_REPO} ${repoDirectory}
        """
        
        echo "Repository cloned successfully into ${repoDirectory}"
    } catch (Exception e) {
        echo "Error while cloning repository: ${e.getMessage()}"
        throw e // Rethrow the exception to handle it at a higher level or fail the build
    }
}

def updateHelmValues(String fileName, String valuesPath) {
    try {
        echo "Updating Helm values file: ${valuesPath}/${fileName} with image tag ${env.IMAGE_TAG}"
        
        sh """
        set -x
        yq eval '.image.tag = \"${env.IMAGE_TAG}\"' -i ${valuesPath}/${fileName} >&2 || true
        """
        
        echo "Helm values file updated successfully."
    } catch (Exception e) {
        echo "Error while updating Helm values file: ${e.getMessage()}"
        throw e // Rethrow the exception to handle it at a higher level or fail the build
    }
}

def gitCommitAndPush(String repoDirectory) {
        try {
            echo "Committing and pushing changes..."
            sh """
            set -x
            git add . >&2 || true
            git commit -m "Promote ${env.HELM_REPO_BRANCH} environment with new image tag: ${env.IMAGE_TAG}" >&2 || true
            git diff --stat --staged origin/${env.HELM_REPO_BRANCH} >&2 || true
            git push origin >&2 || true
            """
            echo "Changes have been pushed successfully."
        } catch (Exception e) {
            echo "Error while committing and pushing: $e"
            throw e // Re-throw the exception to ensure the build fails appropriately
        }
}

def deployToEnvironment(String environmentPart, String regions) {
    def repoDirectory = "helm-${UUID.randomUUID().toString().take(8)}"
    try {
        setupGitConfig() // Ensure git configuration is set before operations
        cloneRepository(repoDirectory)

        def regionList = regions.tokenize(',')

        dir(repoDirectory) {
            regionList.each { regionPart ->
                def fileNameWeb = "values-${environmentPart}-${regionPart}.yaml"
                def fileNameQueue = "values-${environmentPart}-${regionPart}.yaml"
                
                // Check if the Helm values file exists for web and queue for each region before updating
                if (!fileExists("${env.VALUES_PATH_WEB}/${fileNameWeb}")) {
                    echo "WARNING: File ${env.VALUES_PATH_WEB}/${fileNameWeb} does not exist. Skipping update for this file."
                } else {
                    updateHelmValues(fileNameWeb, env.VALUES_PATH_WEB)
                }

                if (!fileExists("${env.VALUES_PATH_QUEUE}/${fileNameQueue}")) {
                    echo "WARNING: File ${env.VALUES_PATH_QUEUE}/${fileNameQueue} does not exist. Skipping update for this file."
                } else {
                    updateHelmValues(fileNameQueue, env.VALUES_PATH_QUEUE)
                }
            }

            // Attempt to commit and push changes even if there were errors in previous steps
            //gitCommitAndPush(repoDirectory, env.HELM_REPO_BRANCH)
            gitCommitAndPush(repoDirectory)
        }
    } catch (Exception e) {
        echo "An error occurred during deployment: ${e.message}. Continuing with the pipeline."
    }
}

def performSonarQubeAnalysis() {
    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_AUTH_TOKEN')]) {
        def gitUrl = sh(script: 'git config --get remote.origin.url', returnStdout: true).trim()
        echo "Git URL: $gitUrl"

        sh """
            sonar-scanner \
            -Dsonar.host.url="${env.SONAR_URL}" \
            -Dsonar.login=$SONAR_AUTH_TOKEN \
            -Dsonar.branch.name=$BRANCH_NAME \
            -Dsonar.projectKey="${env.SONARPROJECT}" \
            -Dsonar.sources=.
        """
    }
}

def kanikoBuildImage(String dockerfilePath, String imageName, String imageTag, String buildtype) {
    sh "cp ${dockerfilePath} ./Dockerfile && ls -al"
    kanikoBuild {
        debug true
        dockerfile 'Dockerfile'
        registry '359585083818.dkr.ecr.us-east-1.amazonaws.com'
        image "jamf/${env.REPOSITORY}/datajar/${imageName}"
        tag "${imageTag}"
        //buildArgs "${buildArgs}"
        buildArgs "build_type=${buildtype}"
        serviceAccountName 'ecr-ci-sa'
        args "--compressed-caching=false"
        awsRoleArn 'arn:aws:iam::359585083818:role/ecr-ci'
    }
    sh 'rm ./Dockerfile'
}

```
