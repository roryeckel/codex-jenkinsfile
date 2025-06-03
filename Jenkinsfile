pipeline {
    agent any // Specify a particular agent if needed (e.g., one with Node.js/npm and Git)

    parameters {
        string(name: 'PROMPT', description: 'Prompt for OpenAI Codex')
        credentials(name: 'OPENAI_API_KEY_CREDENTIAL_ID', description: 'Jenkins Credential ID for OpenAI API Key (Secret text type)', required: true, credentialType: 'com.cloudbees.plugins.credentials.impl.StringCredentialsImpl')
        string(name: 'OPENAI_API_BASE_URL', defaultValue: 'https://router.requesty.ai/v1', description: 'OpenAI API Base URL (e.g., for Azure OpenAI or other providers)')
        string(name: 'GIT_REPO_URL', description: 'Git repository URL to clone into the workspace for Codex')
        string(name: 'GIT_BRANCH', defaultValue: 'master', description: 'Git branch to checkout')
        string(name: 'GIT_USER_NAME', defaultValue: 'Jenkins CI', description: 'Git user name for commits')
        string(name: 'GIT_USER_EMAIL', defaultValue: 'jenkins@example.com', description: 'Git user email for commits')
        credentials(name: 'GIT_CREDENTIAL_ID', description: 'Jenkins Credential ID for cloning and pushing (optional, e.g., SSH key or username/password if not globally configured on agent)', credentialType: "com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl")
        string(name: 'MODEL', defaultValue: 'openai/gpt-4.1', description: 'Model to use for OpenAI Codex')
        string(name: 'PROVIDER', defaultValue: 'requesty', description: 'OpenAI-compatible provider to use (openai, requesty, etc...)')
        booleanParam(name: 'ENABLE_GIT_PUSH', defaultValue: false, description: 'Create and push any leftover changes to new branch after Codex has exited (like: codex-build-<BUILD_NUMBER>)')
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
                    
                    sh "git checkout -b ${branchName}"
                    sh "git add ." // Stage all changes
                    sh "git commit -m 'Changes by Codex (Build ${BUILD_NUMBER})\n\nPrompt: ${params.PROMPT}'"

                    if (params.ENABLE_GIT_PUSH) {
                        echo "Committing and pushing to branch ${branchName}..."
                        if (params.GIT_CREDENTIAL_ID != null && !params.GIT_CREDENTIAL_ID.isEmpty()) {
                            echo "Attempting to push using credentials provided by GIT_CREDENTIAL_ID (expected to be embedded in 'origin' remote URL)."
                        } else {
                            echo "Attempting to push using anonymous access or pre-configured Git credentials on the agent (GIT_CREDENTIAL_ID not provided or empty)."
                        }
                        sh "git push origin ${branchName}"
                        echo "Changes pushed to branch ${branchName} on remote 'origin'."
                    } else {
                        echo "ENABLE_GIT_PUSH is false. Skipping git push step."
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
