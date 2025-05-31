pipeline {
    agent any // Specify a particular agent if needed (e.g., one with Node.js/npm and Git)

    parameters {
        string(name: 'PROMPT', description: 'Prompt for OpenAI Codex')
        credentials(name: 'OPENAI_API_KEY_CREDENTIAL_ID', description: 'Jenkins Credential ID for OpenAI API Key (Secret text type)', required: true, credentialType: 'com.cloudbees.plugins.credentials.impl.StringCredentialsImpl')
        string(name: 'OPENAI_API_BASE_URL', defaultValue: 'https://api.openai.com/v1', description: 'OpenAI API Base URL (e.g., for Azure OpenAI or other providers)')
        string(name: 'GIT_REPO_URL', description: 'Git repository URL to clone')
        string(name: 'GIT_BRANCH', defaultValue: 'master', description: 'Git branch to checkout')
        string(name: 'GIT_USER_NAME', defaultValue: 'Jenkins CI', description: 'Git user name for commits')
        string(name: 'GIT_USER_EMAIL', defaultValue: 'jenkins@example.com', description: 'Git user email for commits')
        credentials(name: 'GIT_CREDENTIAL_ID', description: 'Jenkins Credential ID for Git push (optional, e.g., SSH key or username/password if not globally configured on agent)', credentialType: "com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl")
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
                echo "Initializing/refreshing workspace for repository: ${params.GIT_REPO_URL}, branch: ${params.GIT_BRANCH}"
                sh "git init"
                sh "git remote rm origin || true" // Remove existing origin, if any, ignore error if not present
                sh "git remote add origin ${params.GIT_REPO_URL}"
                sh "git fetch origin ${params.GIT_BRANCH}" // Fetch updates for the specified branch
                sh "git checkout -f ${params.GIT_BRANCH}" // Switch to or create the local branch, force if necessary
                // Reset the local branch to exactly match the state of the remote branch
                sh "git reset --hard origin/${params.GIT_BRANCH}"
                // Remove any untracked files and directories to ensure a clean workspace
                sh "git clean -fdx"
                sh "git status" // Verify Git repository and branch
            }
        }

        stage('Install Codex CLI') {
            steps {
                echo "Installing OpenAI Codex CLI..."
                // Ensure Node.js and npm are available on the agent
                // The -g flag installs it globally. Ensure the Jenkins user has permissions
                // and the global bin directory is in PATH for subsequent stages.
                sh 'npm install -g @openai/codex'
            }
        }

        stage('Invoke Codex') {
            steps {
                withCredentials([string(credentialsId: params.OPENAI_API_KEY_CREDENTIAL_ID, variable: 'API_KEY_SECRET')]) {
                    sh """
                        export OPENAI_API_KEY="\${API_KEY_SECRET}"
                        export OPENAI_BASE_URL="${params.OPENAI_API_BASE_URL}"
                        
                        echo "OpenAI API Key and Base URL environment variables set."
                        echo "Invoking Codex with prompt: '${params.PROMPT}'"
                        
                        # Ensure codex command is found. If npm global path issues, might need:
                        # export PATH=\$PATH:\$(npm prefix -g)/bin
                        codex "${params.PROMPT}" --approval-mode full-auto
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

                    echo "Committing and pushing to branch ${branchName}..."
                    // Handle Git push credentials
                    if (params.GIT_CREDENTIAL_ID != null && !params.GIT_CREDENTIAL_ID.isEmpty()) {
                        // Example for SSH key credentials. Adjust type for username/password.
                        // For usernamePassword, it would be:
                        // withCredentials([usernamePassword(credentialsId: params.GIT_CREDENTIAL_ID, usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        //    sh 'git push https://\${GIT_USERNAME}:\${GIT_PASSWORD}@yourgithost.com/yourrepo.git ${branchName}'
                        // }
                        // For SSH:
                        withCredentials([sshUserPrivateKey(credentialsId: params.GIT_CREDENTIAL_ID, keyFileVariable: 'GIT_SSH_KEY', passphraseVariable: 'GIT_SSH_PASSPHRASE', usernameVariable: 'GIT_SSH_USERNAME')]) {
                            // The GIT_SSH_COMMAND environment variable can be used to specify the SSH command with the key.
                            // This often requires more setup (e.g. known_hosts).
                            // A simpler approach for SSH is often to ensure the Jenkins agent's SSH environment is pre-configured.
                            // However, if using explicit key:
                            sh """
                                # Ensure correct permissions for the SSH key file
                                mkdir -p ~/.ssh
                                cp "\${GIT_SSH_KEY}" ~/.ssh/id_rsa_jenkins_job
                                chmod 600 ~/.ssh/id_rsa_jenkins_job
                                export GIT_SSH_COMMAND="ssh -i ~/.ssh/id_rsa_jenkins_job -o IdentitiesOnly=yes -o StrictHostKeyChecking=no"
                                git push origin ${branchName}
                                rm -f ~/.ssh/id_rsa_jenkins_job # Clean up key
                            """
                        }
                    } else {
                        // Assumes Git credentials are set up on the agent (e.g., via SSH agent forwarding or .git-credentials file)
                        echo "Attempting to push using pre-configured Git credentials on the agent."
                        sh "git push origin ${branchName}"
                    }
                    echo "Changes pushed to branch ${branchName} on remote 'origin'."
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
