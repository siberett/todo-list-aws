pipeline {
    agent none

    options {
        timestamps()
        skipDefaultCheckout(true)
    }

    environment {
        REPO_URL = 'https://github.com/siberett/todo-list-aws.git'
        GIT_CREDENTIALS_ID = 'github-pat'

        AWS_DEFAULT_REGION = 'us-east-1'
        STACK_NAME = 'todo-list-aws-staging'
        STAGE_NAME = 'staging'

        TEST_FILE = 'test/integration/todoApiTest.py'
    }

    stages {
        stage('Get Code') {
            agent { label 'controller' }

            steps {
                echo '=== GET CODE ==='

                sh '''
                    whoami
                    hostname
                    pwd
                '''

                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/development']],
                    userRemoteConfigs: [[
                        url: "${REPO_URL}",
                        credentialsId: "${GIT_CREDENTIALS_ID}"
                    ]]
                ])

                sh '''
                    echo "Current repository state"
                    git status
                    git log -1 --oneline
                    ls -la
                '''

                stash name: 'source-code', includes: '**/*'
            }
        }

        stage('Static Test') {
            agent { label 'static-agent' }

            steps {
                echo '=== STATIC TEST: FLAKE8 + BANDIT ==='

                unstash 'source-code'

                sh '''
                    export PATH="$HOME/.local/bin:$PATH"

                    whoami
                    hostname
                    pwd

                    echo "Checking tools"
                    flake8 --version
                    bandit --version

                    mkdir -p reports

                    echo "Running Flake8 only on src/"
                    set +e
                    flake8 src --output-file=reports/flake8.log
                    FLAKE8_EXIT=$?
                    set -e

                    if [ ! -f reports/flake8.log ]; then
                        echo "ERROR: Flake8 report was not generated"
                        exit 1
                    fi

                    echo "Flake8 exit code: $FLAKE8_EXIT"
                    echo "Flake8 findings do not fail this stage because no quality gate is required."

                    echo "Running Bandit only on src/"
                    set +e
                    bandit -r src -f json -o reports/bandit.json
                    BANDIT_EXIT=$?
                    set -e

                    if [ ! -f reports/bandit.json ]; then
                        echo "ERROR: Bandit report was not generated"
                        exit 1
                    fi

                    echo "Bandit exit code: $BANDIT_EXIT"
                    echo "Bandit findings do not fail this stage because no quality gate is required."

                    ls -la reports
                '''
            }

            post {
                always {
                    recordIssues(
                        enabledForFailure: true,
                        aggregatingResults: false,
                        tools: [
                            flake8(pattern: 'reports/flake8.log'),
                            bandit(pattern: 'reports/bandit.json')
                        ],
                        qualityGates: []
                    )

                    archiveArtifacts artifacts: 'reports/*', fingerprint: true
                }
            }
        }

        stage('Deploy') {
            agent { label 'controller' }

            steps {
                echo '=== DEPLOY TO STAGING WITH AWS SAM ==='

                unstash 'source-code'

                sh '''
                    whoami
                    hostname
                    pwd

                    echo "Checking AWS identity"
                    aws sts get-caller-identity

                    echo "Checking SAM version"
                    sam --version

                    echo "Validating SAM template"
                    sam validate --template-file template.yaml

                    echo "Building SAM application"
                    sam build --template-file template.yaml

                    echo "Deploying SAM application to staging"

                    if [ -f samconfig.toml ]; then
                        mv samconfig.toml samconfig.toml.bak
                    fi

                    sam deploy \
                        --template-file .aws-sam/build/template.yaml \
                        --stack-name "$STACK_NAME" \
                        --region "$AWS_DEFAULT_REGION" \
                        --capabilities CAPABILITY_IAM \
                        --resolve-s3 \
                        --no-confirm-changeset \
                        --no-fail-on-empty-changeset \
                        --parameter-overrides Stage="$STAGE_NAME"

                    echo "Obtaining API Gateway base URL from CloudFormation outputs"

                    API_URL=$(aws cloudformation describe-stacks \
                        --stack-name "$STACK_NAME" \
                        --region "$AWS_DEFAULT_REGION" \
                        --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                        --output text)

                    if [ -z "$API_URL" ] || [ "$API_URL" = "None" ]; then
                        echo "ERROR: Could not obtain BaseUrlApi from CloudFormation outputs"
                        exit 1
                    fi

                    echo "API URL obtained:"
                    echo "$API_URL"

                    echo "$API_URL" > api_url.txt
                '''

                stash name: 'api-url', includes: 'api_url.txt'
            }
        }

        stage('Rest Test') {
            agent { label 'test-agent' }

            steps {
                echo '=== REST TEST WITH PYTEST ==='

                unstash 'source-code'
                unstash 'api-url'

                sh '''
                    export PATH="$HOME/.local/bin:$PATH"

                    whoami
                    hostname
                    pwd

                    echo "Checking pytest"
                    pytest --version

                    mkdir -p reports

                    export BASE_URL="$(cat api_url.txt)"

                    echo "Running tests against:"
                    echo "$BASE_URL"

                    pytest "$TEST_FILE" --junitxml=reports/pytest-results.xml
                '''
            }

            post {
                always {
                    junit allowEmptyResults: false, testResults: 'reports/pytest-results.xml'
                    archiveArtifacts artifacts: 'reports/*', fingerprint: true
                }
            }
        }

        stage('Promote') {
            agent { label 'controller' }

            steps {
                echo '=== PROMOTE DEVELOPMENT TO MASTER ==='

                unstash 'source-code'

                withCredentials([usernamePassword(
                    credentialsId: "${GIT_CREDENTIALS_ID}",
                    usernameVariable: 'GIT_USERNAME',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    sh '''
                        whoami
                        hostname
                        pwd

                        git config user.email "jenkins@example.com"
                        git config user.name "Jenkins CI"

                        git remote set-url origin https://${GIT_USERNAME}:${GIT_TOKEN}@github.com/siberett/todo-list-aws.git

                        git fetch origin development
                        git fetch origin master

                        git checkout master
                        git reset --hard origin/master

                        echo "Preserving current master Jenkinsfile if it exists"
                        if git cat-file -e HEAD:Jenkinsfile 2>/dev/null; then
                            git show HEAD:Jenkinsfile > /tmp/master-Jenkinsfile
                            MASTER_JENKINSFILE_EXISTS="yes"
                        else
                            MASTER_JENKINSFILE_EXISTS="no"
                        fi

                        echo "Merging development into master"
                        git merge --no-ff origin/development -m "Promote development to master from Jenkins CI"

                        if [ "$MASTER_JENKINSFILE_EXISTS" = "yes" ]; then
                            echo "Restoring master Jenkinsfile to avoid overwriting CD pipeline"
                            cp /tmp/master-Jenkinsfile Jenkinsfile
                            git add Jenkinsfile

                            if ! git diff --cached --quiet; then
                                git commit -m "Preserve master Jenkinsfile after promotion"
                            else
                                echo "Master Jenkinsfile did not change"
                            fi
                        fi

                        git push origin master
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI pipeline completed successfully. Code promoted to master.'
        }

        failure {
            echo 'CI pipeline failed. Code was not promoted to master.'
        }

        always {
            cleanWs()
        }
    }
}
