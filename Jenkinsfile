 pipeline {
    agent any

    options {
        timestamps()
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
    }

    environment {
        REPOSITORY_URL = 'https://github.com/siberett/todo-list-aws.git'
        GIT_CREDENTIALS_ID = 'github-pat'

        AWS_DEFAULT_REGION = 'us-east-1'
        STACK_NAME = 'todo-list-aws-staging'
        SAM_ENVIRONMENT = 'staging'

        TEST_FILE = 'test/integration/todoApiTest.py'
    }

    stages {

        stage('Get Code') {
            steps {
                cleanWs()

                sh '''
                    set -e

                    echo "=== GET CODE ==="
                    whoami
                    hostname
                    pwd
                    git --version
                '''

                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/development']],
                    userRemoteConfigs: [[
                        url: "${REPOSITORY_URL}",
                        credentialsId: "${GIT_CREDENTIALS_ID}"
                    ]]
                ])

                sh '''
                    set -e

                    echo "--- Revision descargada ---"
                    git log -1 --oneline
                    git status

                    echo "--- Código fuente ---"
                    find src -maxdepth 3 -type f | sort

                    echo "--- Pruebas de integración ---"
                    find test/integration -maxdepth 2 -type f | sort
                '''
            }
        }

        stage('Static Test') {
            steps {
                sh '''
                    set -e

                    echo "=== STATIC TEST ==="
                    whoami
                    hostname
                    pwd

                    echo "--- Versiones ---"
                    flake8 --version
                    bandit --version

                    mkdir -p reports

                    echo "--- Flake8 sobre src/ ---"

                    flake8 \
                        --exit-zero \
                        --format=pylint \
                        src > reports/flake8.out

                    test -f reports/flake8.out

                    echo "--- Bandit sobre src/ ---"

                    bandit \
                        --exit-zero \
                        -r src \
                        -f custom \
                        -o reports/bandit.out \
                        --msg-template "{abspath}:{line}: [{test_id}] {msg}"

                    test -f reports/bandit.out

                    echo "--- Informes generados ---"
                    ls -la reports
                '''

                recordIssues(
                    enabledForFailure: true,
                    tools: [
                        flake8(
                            name: 'Flake8',
                            pattern: 'reports/flake8.out'
                        ),
                        pyLint(
                            name: 'Bandit',
                            pattern: 'reports/bandit.out'
                        )
                    ]
                )
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    set -e

                    echo "=== DEPLOY STAGING ==="
                    whoami
                    hostname
                    pwd

                    echo "--- Versiones ---"
                    aws --version
                    sam --version

                    echo "--- Identidad AWS ---"
                    aws sts get-caller-identity

                    echo "--- Validación SAM ---"
                    sam validate --template-file template.yaml

                    echo "--- Construcción SAM ---"
                    sam build --template-file template.yaml

                    echo "--- Despliegue no interactivo en Staging ---"
                    echo "Stack: $STACK_NAME"
                    echo "Region: $AWS_DEFAULT_REGION"
                    echo "Stage: $SAM_ENVIRONMENT"

                    # Se desactiva temporalmente samconfig.toml para evitar
                    # utilizar buckets antiguos o inexistentes.
                    if [ -f samconfig.toml ]; then
                        mv samconfig.toml samconfig.toml.disabled
                    fi

                    restore_samconfig() {
                        if [ -f samconfig.toml.disabled ]; then
                            mv samconfig.toml.disabled samconfig.toml
                        fi
                    }

                    trap restore_samconfig EXIT

                    sam deploy \
                        --template-file .aws-sam/build/template.yaml \
                        --stack-name "$STACK_NAME" \
                        --region "$AWS_DEFAULT_REGION" \
                        --capabilities CAPABILITY_IAM \
                        --resolve-s3 \
                        --no-confirm-changeset \
                        --no-fail-on-empty-changeset \
                        --parameter-overrides Stage="$SAM_ENVIRONMENT"

                    restore_samconfig
                    trap - EXIT

                    echo "--- Obtención de la URL de API Gateway ---"

                    API_URL=$(aws cloudformation describe-stacks \
                        --stack-name "$STACK_NAME" \
                        --region "$AWS_DEFAULT_REGION" \
                        --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                        --output text)

                    if [ -z "$API_URL" ] || [ "$API_URL" = "None" ]; then
                        echo "ERROR: No se encontró el output BaseUrlApi."
                        exit 1
                    fi

                    echo "API URL: $API_URL"
                    printf '%s' "$API_URL" > api_url.txt
                '''
            }
        }

        stage('Rest Test') {
            steps {
                sh '''
                    set -e

                    echo "=== REST TEST ==="
                    whoami
                    hostname
                    pwd

                    echo "--- Versiones ---"
                    pytest --version
                    curl --version

                    mkdir -p reports

                    export BASE_URL="$(cat api_url.txt)"

                    if [ -z "$BASE_URL" ]; then
                        echo "ERROR: BASE_URL está vacía."
                        exit 1
                    fi

                    echo "Ejecutando pruebas contra: $BASE_URL"

                    test -f "$TEST_FILE"

                    pytest \
                        "$TEST_FILE" \
                        --junitxml=reports/pytest-results.xml
                '''
            }

            post {
                always {
                    junit(
                        testResults: 'reports/pytest-results.xml',
                        allowEmptyResults: false
                    )
                }
            }
        }

        stage('Promote') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${GIT_CREDENTIALS_ID}",
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh '''
                        set -e

                        echo "=== PROMOTE DEVELOPMENT TO MASTER ==="
                        whoami
                        hostname
                        pwd

                        git config user.name "Jenkins CI"
                        git config user.email "jenkins@localhost"

                        git remote set-url origin \
                            "https://${GIT_USERNAME}:${GIT_TOKEN}@github.com/siberett/todo-list-aws.git"

                        git fetch origin development master

                        git checkout -B master origin/master

                        git merge \
                            --no-ff \
                            origin/development \
                            -m "Promote development to master from Jenkins CI"

                        git push origin master

                        echo "--- Último commit de master ---"
                        git log -1 --oneline
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline CI completado. La rama development se ha mergeado en master.'
        }

        failure {
            echo 'Pipeline CI fallido. No se ha promocionado el código a master.'
        }

        always {
            cleanWs()
        }
    }
}
