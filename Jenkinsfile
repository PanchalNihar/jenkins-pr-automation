pipeline {
    agent any
    
    environment {
        GITHUB_TOKEN = credentials('github-token')
        REPO_OWNER = 'PanchalNihar'
        REPO_NAME = 'jenkins-pr-automation'
        BASE_BRANCH = 'main'
        FEATURE_BRANCH = "automated-update-${BUILD_NUMBER}"
        
        GIT_AUTHOR_NAME = 'Jenkins Bot'
        GIT_AUTHOR_EMAIL = 'jenkins@yourcompany.com'
        
        // Flag to track if PR is needed
        CHANGES_DETECTED = 'false'
    }
    
    stages {
        stage('Checkout Repository') {
            steps {
                script {
                    echo "=== Checking out repository ==="
                    checkout scm
                    sh """
                        git config user.name "${GIT_AUTHOR_NAME}"
                        git config user.email "${GIT_AUTHOR_EMAIL}"
                    """
                }
            }
        }
        
        stage('Detect Changes Needed') {
            steps {
                script {
                    echo "=== Analyzing if updates are needed ==="
                    
                    // Example 1: Check for outdated npm packages
                    def npmOutdated = sh(
                        script: 'npm outdated --json || true',
                        returnStdout: true
                    ).trim()
                    
                    // Example 2: Check if code formatting needed
                    def formattingNeeded = sh(
                        script: 'npx prettier --check . || echo "needs-formatting"',
                        returnStdout: true
                    ).contains('needs-formatting')
                    
                    // Example 3: Check if documentation is outdated
                    def docsOutdated = sh(
                        script: 'find src -name "*.js" -newer README.md | wc -l',
                        returnStdout: true
                    ).trim().toInteger() > 0
                    
                    // Decide if changes are needed
                    if (npmOutdated || formattingNeeded || docsOutdated) {
                        env.CHANGES_DETECTED = 'true'
                        echo "✅ Changes detected! PR will be created."
                    } else {
                        env.CHANGES_DETECTED = 'false'
                        echo "✅ No changes needed. Exiting gracefully."
                    }
                }
            }
        }
        
        stage('Create Feature Branch') {
            when {
                expression { env.CHANGES_DETECTED == 'true' }
            }
            steps {
                script {
                    echo "=== Creating feature branch ${FEATURE_BRANCH} ==="
                    sh "git checkout -b ${FEATURE_BRANCH}"
                }
            }
        }
        
        stage('Apply Automated Updates') {
            when {
                expression { env.CHANGES_DETECTED == 'true' }
            }
            steps {
                script {
                    echo "=== Applying automated updates ==="
                    
                    // Update npm dependencies
                    sh """
                        npm update || true
                    """
                    
                    // Auto-format code
                    sh """
                        npx prettier --write . || true
                    """
                    
                    // Regenerate documentation
                    sh """
                        npm run docs || true
                    """
                    
                    // Update changelog
                    sh """
                        echo "\n## Automated Update - \$(date +%Y-%m-%d)" >> CHANGELOG.md
                        echo "- Dependency updates" >> CHANGELOG.md
                        echo "- Code formatting applied" >> CHANGELOG.md
                        echo "- Documentation regenerated" >> CHANGELOG.md
                    """
                }
            }
        }
        
        stage('Verify Changes Exist') {
            when {
                expression { env.CHANGES_DETECTED == 'true' }
            }
            steps {
                script {
                    echo "=== Verifying actual file changes ==="
                    
                    // Check if git actually has changes
                    def gitStatus = sh(
                        script: 'git status --porcelain',
                        returnStdout: true
                    ).trim()
                    
                    if (gitStatus == '') {
                        echo "⚠️  No actual file changes after updates. Skipping PR creation."
                        env.CHANGES_DETECTED = 'false'
                        // Clean up the branch we created
                        sh "git checkout ${BASE_BRANCH}"
                        sh "git branch -D ${FEATURE_BRANCH} || true"
                    } else {
                        echo "✅ File changes confirmed: ${gitStatus}"
                    }
                }
            }
        }
        
        stage('Commit Changes') {
            when {
                expression { env.CHANGES_DETECTED == 'true' }
            }
            steps {
                script {
                    echo "=== Committing changes ==="
                    sh """
                        git add .
                        git commit -m "chore: Automated maintenance updates
                        
                        - Update dependencies to latest versions
                        - Apply code formatting standards
                        - Regenerate documentation
                        
                        Generated by Jenkins Build #${BUILD_NUMBER}"
                    """
                }
            }
        }
        
        stage('Push to GitHub') {
            when {
                expression { env.CHANGES_DETECTED == 'true' }
            }
            steps {
                script {
                    echo "=== Pushing to GitHub ==="
                    withCredentials([usernamePassword(
                        credentialsId: 'github-credentials',
                        passwordVariable: 'GIT_PASSWORD',
                        usernameVariable: 'GIT_USERNAME'
                    )]) {
                        sh """
                            git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${REPO_OWNER}/${REPO_NAME}.git ${FEATURE_BRANCH}
                        """
                    }
                }
            }
        }
        
        stage('Create Pull Request') {
            when {
                expression { env.CHANGES_DETECTED == 'true' }
            }
            steps {
                script {
                    echo "=== Creating Pull Request ==="
                    
                    def prBody = """
                    ## 🤖 Automated Maintenance Update
                    
                    This PR contains automated updates generated by Jenkins.
                    
                    ### Changes Include:
                    - 📦 Dependency updates to latest stable versions
                    - 💅 Code formatting applied
                    - 📚 Documentation regenerated
                    
                    ### Details:
                    - **Build Number**: #${BUILD_NUMBER}
                    - **Generated**: ${new Date()}
                    - **Branch**: \`${FEATURE_BRANCH}\`
                    
                    Please review the changes and merge if everything looks good.
                    """
                    
                    def response = sh(script: """
                        curl -X POST \
                        -H "Authorization: token ${GITHUB_TOKEN}" \
                        -H "Accept: application/vnd.github.v3+json" \
                        https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/pulls \
                        -d '{
                            "title": "chore: Automated maintenance updates - Build #${BUILD_NUMBER}",
                            "body": ${groovy.json.JsonOutput.toJson(prBody)},
                            "head": "${FEATURE_BRANCH}",
                            "base": "${BASE_BRANCH}"
                        }'
                    """, returnStdout: true)
                    
                    echo "GitHub API Response: ${response}"
                    
                    def prNumber = sh(
                        script: "echo '${response}' | grep -o '\"number\":[0-9]*' | grep -o '[0-9]*'",
                        returnStdout: true
                    ).trim()
                    
                    echo "✅ Pull Request #${prNumber} created successfully!"
                }
            }
        }
    }
    
    post {
        success {
            script {
                if (env.CHANGES_DETECTED == 'true') {
                    echo '✅ Pipeline completed! PR created successfully.'
                } else {
                    echo '✅ Pipeline completed! No updates needed.'
                }
            }
        }
        failure {
            echo '❌ Pipeline failed. Check logs for details.'
        }
        cleanup {
            // Switch back to main branch
            sh "git checkout ${BASE_BRANCH} || true"
            cleanWs()
        }
    }
}
