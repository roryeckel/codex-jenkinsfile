def gitPushWithAskpass(String gitCommand) {
    return """
        set -eu
        tmp_askpass=\$(mktemp)
        trap 'rm -f "\$tmp_askpass"' EXIT
        cat <<'EOF' > "\$tmp_askpass"
#!/bin/sh
case "\$1" in
  Username*) printf '%s\\n' "\${GIT_PUSH_USERNAME}" ;;
  Password*) printf '%s\\n' "\${GIT_PUSH_PASSWORD}" ;;
  *) exit 1 ;;
esac
EOF
        chmod +x "\$tmp_askpass"
        GIT_ASKPASS="\$tmp_askpass" GIT_TERMINAL_PROMPT=0 ${gitCommand}
    """
}

pipeline {
    agent any // Specify a particular agent if needed (e.g., one with Node.js/npm and Git)

    parameters {
        string(name: 'PROMPT', description: 'Prompt for OpenAI Codex')
        credentials(name: 'OPENAI_API_KEY_CREDENTIAL_ID', description: 'Jenkins Credential ID for OpenAI API Key (Secret text type)', required: true, credentialType: 'com.cloudbees.plugins.credentials.impl.StringCredentialsImpl')
        string(name: 'OPENAI_API_BASE_URL', defaultValue: 'https://api.openai.com/v1', description: 'OpenAI API Base URL (e.g., for Requesty or other providers)')
        string(name: 'GIT_REPO_URL', description: 'Git repository URL to clone into the workspace for Codex')
        string(name: 'GIT_BRANCH', defaultValue: 'master', description: 'Git branch to checkout')
        string(name: 'GIT_USER_NAME', defaultValue: 'Jenkins CI', description: 'Git user name for commits')
        string(name: 'GIT_USER_EMAIL', defaultValue: 'jenkins@example.com', description: 'Git user email for commits')
        credentials(name: 'GIT_CREDENTIAL_ID', description: 'Jenkins Credential ID for cloning and pushing (optional, e.g., SSH key or username/password if not globally configured on agent)', credentialType: "com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl")
        string(name: 'MODEL', defaultValue: 'gpt-5-codex', description: 'Model to use for OpenAI Codex')
        string(name: 'PROVIDER', defaultValue: 'openai', description: 'OpenAI-compatible provider to use (openai, requesty, etc...)')
        booleanParam(name: 'ENABLE_GIT_PUSH', defaultValue: false, description: 'Create and push any leftover changes to new branch after Codex has exited (like: codex-build-<BUILD_NUMBER>)')
        booleanParam(name: 'ENABLE_SUBMODULES', defaultValue: false, description: 'Initialize and update Git submodules after repository checkout')
    }

    stages {
        stage('Validate Parameters') {
            steps {
                script {
                    // Validate that the required parameters are set
                    if (!params.PROMPT) {
                        error "PROMPT parameter is required."
                    }
                    if (!params.OPENAI_API_KEY_CREDENTIAL_ID) {
                        error "OPENAI_API_KEY_CREDENTIAL_ID parameter is required."
                    }
                    if (!params.GIT_REPO_URL) {
                        error "GIT_REPO_URL parameter is required."
                    }
                }
            }
        }

        stage('Initialize Workspace') {
            steps {
                script {
                    echo "Initializing/refreshing workspace for repository: ${params.GIT_REPO_URL}, branch: ${params.GIT_BRANCH}"
                    sh "git init"
                    sh "git remote rm origin || true" // Remove existing origin, if any, ignore error if not present

                    def repoUrlToUse = params.GIT_REPO_URL
                    if (params.GIT_CREDENTIAL_ID != null && !params.GIT_CREDENTIAL_ID.isEmpty()) {
                        echo "Using GIT_CREDENTIAL_ID for repository access."
                        withCredentials([usernamePassword(credentialsId: params.GIT_CREDENTIAL_ID, usernameVariable: 'GIT_USERNAME_PLAIN', passwordVariable: 'GIT_PASSWORD_PLAIN')]) {
                            
                            def encodedUsername = java.net.URLEncoder.encode(GIT_USERNAME_PLAIN, "UTF-8")
                            def encodedPassword = java.net.URLEncoder.encode(GIT_PASSWORD_PLAIN, "UTF-8").replace('+', '%20')
                            
                            def repoUrlNoProto = params.GIT_REPO_URL.replace("https://", "").replace("http://", "")
                            def authenticatedRepoUrl = "https://${encodedUsername}:${encodedPassword}@${repoUrlNoProto}"
                            
                            echo "Configuring remote 'origin' with URL-encoded credentials."
                            // Use withEnv to scope the environment variable containing the sensitive URL
                            // and avoid direct Groovy string interpolation of the secret in the sh step.
                            withEnv(["AUTHENTICATED_REPO_URL_FOR_GIT=${authenticatedRepoUrl}"]) {
                                sh 'git remote add origin "$AUTHENTICATED_REPO_URL_FOR_GIT"'
                            }
                            sh "git fetch origin ${params.GIT_BRANCH}"
                            sh "git checkout -f ${params.GIT_BRANCH}" // Switch to or create the local branch, force if necessary
                            sh "git reset --hard origin/${params.GIT_BRANCH}" // Reset the local branch to exactly match the state of the remote branch
                        }
                    } else {
                        echo "No GIT_CREDENTIAL_ID provided or it's empty. Attempting anonymous access or agent pre-configured credentials."
                        sh "git remote add origin \"${repoUrlToUse}\"" // Uses original params.GIT_REPO_URL
                        sh "git fetch origin ${params.GIT_BRANCH}"
                        sh "git checkout -f ${params.GIT_BRANCH}"
                        sh "git reset --hard origin/${params.GIT_BRANCH}"
                    }
                    
                    // Initialize and update submodules if requested
                    if (params.ENABLE_SUBMODULES) {
                        echo "ENABLE_SUBMODULES is true. Initializing and updating Git submodules..."
                        sh "git submodule init"
                        sh "git submodule update --recursive"
                        echo "Git submodules initialized and updated."
                    } else {
                        echo "ENABLE_SUBMODULES is false. Skipping submodule initialization."
                    }
                    
                    // Remove any untracked files and directories to ensure a clean workspace
                    sh "git clean -fdx"
                    sh "git status" // Verify Git repository and branch
                }
            }
        }

        stage('Invoke Codex') {
            steps {
                withCredentials([string(credentialsId: params.OPENAI_API_KEY_CREDENTIAL_ID, variable: 'API_KEY_SECRET')]) {
                    sh """
                        export OPENAI_API_KEY="\${API_KEY_SECRET}"
                        export OPENAI_BASE_URL="${params.OPENAI_API_BASE_URL}"
                        
                        echo "OpenAI API Key: \$OPENAI_API_KEY"
                        echo "OpenAI Base URL: \$OPENAI_BASE_URL"
                        echo "Invoking Codex with prompt: '${params.PROMPT}'"
                        
                        codex "${params.PROMPT}" --model "${params.MODEL}" --provider "${params.PROVIDER}" -a auto-edit --quiet
                    """
                }
            }
        }

        stage('Check for Git Changes') {
            steps {
                script {
                    echo "Checking for Git changes made by Codex..."
                    // git status --porcelain returns non-empty output if there are changes
                    def changes = sh(script: 'git status --porcelain', returnStdout: true).trim()
                    if (changes) {
                        echo "Changes detected by Codex:"
                        sh 'git status --short' // Show a summary of changes
                        
                        // Show submodule status if submodules are enabled
                        if (params.ENABLE_SUBMODULES) {
                            echo "Checking submodule status:"
                            sh 'git submodule status || echo "No submodules found or submodule status unavailable"'
                        }
                        
                        env.CHANGES_DETECTED = "true"
                    } else {
                        echo "No changes detected by Codex."
                        env.CHANGES_DETECTED = "false"
                    }
                }
            }
        }

        stage('Commit and Push Changes') {
            when {
                expression { env.CHANGES_DETECTED == "true" }
            }
            steps {
                script {
                    def branchName = "codex-build-${BUILD_NUMBER}"
                    echo "Changes detected. Creating and pushing changes to branch: ${branchName}"

                    // Configure git user for commit, if not already configured on the agent
                    // Using || true to avoid failure if already configured or not permitted to change global config
                    sh "git config user.name '${params.GIT_USER_NAME}' || true"
                    sh "git config user.email '${params.GIT_USER_EMAIL}' || true"

                    def submoduleCommitsCreated = false
                    
                    // Handle submodule commits first if submodules are enabled
                    if (params.ENABLE_SUBMODULES) {
                        echo "ENABLE_SUBMODULES is true. Checking for submodule changes and committing them first..."

                        // Ensure submodule URLs are current before updating content
                        sh "git submodule sync --recursive || true"
                        sh "git submodule update --init --recursive || true"

                        // Get list of submodules using a robust, path-aware approach
                        def submodules = sh(
                            script: "git submodule foreach --recursive 'printf %s\\\\n \"\\$sm_path\"' || true",
                            returnStdout: true
                        ).trim()
                        if (submodules) {
                            submodules.split('\n').each { submodulePath ->
                                def submodule = submodulePath.trim()
                                if (submodule) {
                                    echo "Checking submodule: ${submodule}"
                                    
                                    // Check if there are changes in this submodule
                                    def submoduleChanges = sh(
                                        script: "git -C '${submodule}' status --porcelain",
                                        returnStdout: true
                                    ).trim()
                                    if (submoduleChanges) {
                                        echo "Changes detected in submodule ${submodule}. Committing..."
                                        
                                        // Configure git user in submodule
                                        sh "git -C '${submodule}' config user.name '${params.GIT_USER_NAME}' || true"
                                        sh "git -C '${submodule}' config user.email '${params.GIT_USER_EMAIL}' || true"
                                        
                                        // Commit changes in submodule
                                        sh "git -C '${submodule}' checkout -B ${branchName}"
                                        sh "git -C '${submodule}' add -A"
                                        sh "git -C '${submodule}' commit -m 'Changes by Codex in submodule (Build ${BUILD_NUMBER})\\n\\nPrompt: ${params.PROMPT}' || true"
                                        submoduleCommitsCreated = true
                                        
                                        // Push submodule changes if enabled and credentials available
                                        if (params.ENABLE_GIT_PUSH) {
                                            echo "Attempting to push submodule ${submodule} changes to branch ${branchName}..."

                                            def remoteNamesRaw = sh(
                                                script: "git -C '${submodule}' remote",
                                                returnStdout: true
                                            ).trim()
                                            def pushRemoteName = null
                                            def pushRemoteUrl = null
                                            if (remoteNamesRaw) {
                                                remoteNamesRaw.split('\n').each { remoteName ->
                                                    if (!pushRemoteName) {
                                                        def candidateUrl = sh(
                                                            script: "git -C '${submodule}' remote get-url --push ${remoteName} || true",
                                                            returnStdout: true
                                                        ).trim()
                                                        if (candidateUrl) {
                                                            pushRemoteName = remoteName.trim()
                                                            pushRemoteUrl = candidateUrl
                                                        }
                                                    }
                                                }
                                            }

                                            if (pushRemoteName && pushRemoteUrl) {
                                                def isHttpRemote = pushRemoteUrl.startsWith("http://") || pushRemoteUrl.startsWith("https://")
                                                if (isHttpRemote && params.GIT_CREDENTIAL_ID && !params.GIT_CREDENTIAL_ID.trim().isEmpty()) {
                                                    withCredentials([usernamePassword(credentialsId: params.GIT_CREDENTIAL_ID, usernameVariable: 'GIT_PUSH_USERNAME', passwordVariable: 'GIT_PUSH_PASSWORD')]) {
                                                        sh gitPushWithAskpass("git -C '${submodule}' push -u ${pushRemoteName} ${branchName}")
                                                    }
                                                } else if (isHttpRemote) {
                                                    echo "HTTP(S) push remote detected for ${submodule} but no credentials provided; attempting unauthenticated push."
                                                    sh "git -C '${submodule}' push -u ${pushRemoteName} ${branchName} || echo 'Failed to push submodule ${submodule} without credentials'"
                                                } else {
                                                    sh "git -C '${submodule}' push -u ${pushRemoteName} ${branchName} || echo 'Failed to push submodule ${submodule}; ensure credentials or SSH keys allow pushing'"
                                                }
                                            } else {
                                                echo "No push remote detected for ${submodule}; skipping push."
                                            }
                                        }
                                    } else {
                                        echo "No changes detected in submodule ${submodule}."
                                    }
                                }
                            }
                        } else {
                            echo "No submodules found."
                        }
                    }
                    
                    // Now commit the parent repository (this will include updated submodule references)
                    sh "git checkout -B ${branchName}"
                    sh "git add -A"
                    def parentCommitCreated = false

                    if (params.ENABLE_SUBMODULES && submoduleCommitsCreated) {
                        def stagedSubmoduleSummary = sh(
                            script: "git diff --cached --submodule=short",
                            returnStdout: true
                        ).trim()
                        if (!stagedSubmoduleSummary) {
                            error "Submodule commits were created but no submodule pointer updates are staged in the parent repository."
                        }
                        echo "Staged submodule updates:\n${stagedSubmoduleSummary}"
                    }

                    def parentStagedChanges = sh(script: "git diff --cached --stat", returnStdout: true).trim()
                    if (parentStagedChanges) {
                        echo "Parent repository staged changes:\n${parentStagedChanges}"
                        sh "git commit -m 'Changes by Codex (Build ${BUILD_NUMBER})\\n\\nPrompt: ${params.PROMPT}'"
                        parentCommitCreated = true
                    } else {
                        echo "No staged changes detected in parent repository; skipping commit."
                    }

                    if (params.ENABLE_GIT_PUSH && parentCommitCreated) {
                        echo "Preparing to push parent repository to branch ${branchName}..."

                        def parentPushUrl = sh(script: "git remote get-url --push origin || true", returnStdout: true).trim()
                        if (!parentPushUrl) {
                            echo "No push URL configured for remote 'origin'; skipping parent push."
                        } else {
                            def parentPushIsHttp = parentPushUrl.startsWith("http://") || parentPushUrl.startsWith("https://")
                            if (parentPushIsHttp && params.GIT_CREDENTIAL_ID && !params.GIT_CREDENTIAL_ID.trim().isEmpty()) {
                                withCredentials([usernamePassword(credentialsId: params.GIT_CREDENTIAL_ID, usernameVariable: 'GIT_PUSH_USERNAME', passwordVariable: 'GIT_PUSH_PASSWORD')]) {
                                    sh gitPushWithAskpass("git push -u origin ${branchName}")
                                }
                            } else if (parentPushIsHttp) {
                                echo "HTTP(S) remote detected but no credentials supplied; attempting unauthenticated parent push."
                                sh "git push -u origin ${branchName} || echo 'Failed to push parent repository without credentials.'"
                            } else {
                                sh "git push -u origin ${branchName} || echo 'Failed to push parent repository; ensure agent has required credentials.'"
                            }
                        }
                    } else {
                        echo "Skipping parent push because ENABLE_GIT_PUSH is false or no commit was created."
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished.'
            // Clean up workspace or other post-build actions if necessary
            // deleteDir() // Uncomment to clean up workspace
        }
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}
