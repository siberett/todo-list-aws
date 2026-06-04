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
        SAM_STAGE = 'staging'

        TEST_FILE = 'test/integration/todoApiTest.py'

        STATIC_PYTHON = '/home/ubuntu/.venv/bin/python'
        FLAKE8_BIN = '/home/ubuntu/.venv/bin/flake8'
        BANDIT_BIN = '/home/ubuntu/.venv/bin/bandit'

        TEST_PYTHON = '/home/ubuntu/.venv/bin/python'
        PYTEST_BIN = '/home/ubuntu/.venv/bin/pytest'
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
                    echo "Repository state:"
                    git status
                    git log -1 --oneline

                    echo "Repository files:"
                    ls -la

                    echo "Test files:"
                    find test -maxdepth 4 -type f | sort || true

                    echo "Source files:"
                    find src -maxdepth 4 -type f | sort || true
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
                    whoami
                    hostname
                    pwd

                    echo "--- Static agent tools ---"
                    ${STATIC_PYTHON} --version
                    ${FLAKE8_BIN} --version
                    ${BANDIT_BIN} --version

                    mkdir -p reports

                    echo "Running Flake8 only on src/"
                    set +e
                    ${FLAKE8_BIN} src --output-file=reports/flake8.log
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
                    ${BANDIT_BIN} -r src -f json -o reports/bandit.json
                    BANDIT_EXIT=$?
                    set -e

                    if [ ! -f reports/bandit.json ]; then
                        echo "ERROR: Bandit report was not generated"
                        exit 1
                    fi

                    echo "Bandit exit code: $BANDIT_EXIT"
                    echo "Bandit findings do not fail this stage because no quality gate is required."

                    echo "--- Reports generated ---"
                    ls -la reports
                '''

                archiveArtifacts artifacts: 'reports/*', fingerprint: true
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

                    echo "--- Controller tools ---"
                    git --version
                    aws --version
                    sam --version

                    echo "--- AWS identity ---"
                    aws sts get-caller-identity

                    echo "--- SAM validate ---"
                    sam validate --template-file template.yaml

                    echo "--- SAM build ---"
                    sam build --template-file template.yaml

                    echo "--- SAM deploy to staging ---"
                    echo "STACK_NAME=$STACK_NAME"
                    echo "AWS_DEFAULT_REGION=$AWS_DEFAULT_REGION"
                    echo "SAM_STAGE=$SAM_STAGE"

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
                        --parameter-overrides Stage="$SAM_STAGE"

                    echo "--- Getting API Gateway URL from CloudFormation outputs ---"

                    API_URL=$(aws cloudformation describe-stacks \
                        --stack-name "$STACK_NAME" \
                        --region "$AWS_DEFAULT_REGION" \
                        --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                        --output text)

                    if [ -z "$API_URL" ] || [ "$API_URL" = "None" ]; then
                        echo "ERROR: Could not obtain BaseUrlApi from CloudFormation outputs"
                        echo "Available outputs:"
                        aws cloudformation describe-stacks \
                            --stack-name "$STACK_NAME" \
                            --region "$AWS_DEFAULT_REGION" \
                            --query "Stacks[0].Outputs" \
                            --output table
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
                    whoami
                    hostname
                    pwd

                    echo "--- Test agent tools ---"
                    ${TEST_PYTHON} --version
                    ${PYTEST_BIN} --version
                    curl --version

                    mkdir -p reports

                    export BASE_URL="$(cat api_url.txt)"

                    echo "Running tests against:"
                    echo "$BASE_URL"

                    echo "Checking test file exists:"
                    ls -la "$TEST_FILE"

                    ${PYTEST_BIN} "$TEST_FILE" --junitxml=reports/pytest-results.xml
                '''

                junit allowEmptyResults: false, testResults: 'reports/pytest-results.xml'
                archiveArtifacts artifacts: 'reports/*', fingerprint: true
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

                        echo "--- Fetching branches ---"
                        git fetch origin development
                        git fetch origin master

                        echo "--- Checkout master ---"
                        git checkout master
                        git reset --hard origin/master

                        echo "--- Preserve current master Jenkinsfile if it exists ---"
                        if git cat-file -e HEAD:Jenkinsfile 2>/dev/null; then
                            git show HEAD:Jenkinsfile > /tmp/master-Jenkinsfile
                            MASTER_JENKINSFILE_EXISTS="yes"
                        else
                            MASTER_JENKINSFILE_EXISTS="no"
                        fi

                        echo "--- Merge development into master ---"
                        git merge --no-ff origin/development -m "Promote development to master from Jenkins CI"

                        if [ "$MASTER_JENKINSFILE_EXISTS" = "yes" ]; then
                            echo "--- Restore master Jenkinsfile to avoid overwriting future CD pipeline ---"
                            cp /tmp/master-Jenkinsfile Jenkinsfile
                            git add Jenkinsfile

                            if ! git diff --cached --quiet; then
                                git commit -m "Preserve master Jenkinsfile after promotion"
                            else
                                echo "Master Jenkinsfile did not change"
                            fi
                        fi

                        echo "--- Push master ---"
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
            node('controller') {
                cleanWs()
            }
        }
    }
}
