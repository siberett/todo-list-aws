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

                    echo "=== ÚLTIMO COMMIT ==="
                    git log -1 --oneline

                    echo "=== ESTADO DEL REPOSITORIO ==="
                    git status

                    echo "=== ARCHIVOS DE LA APLICACIÓN ==="
                    find src -maxdepth 3 -type f | sort

                    echo "=== ARCHIVOS DE PRUEBAS ==="
                    find test/integration -maxdepth 2 -type f | sort
                '''
            }
        }

        stage('Static Test') {
            steps {
                sh '''
                    set -e

                    echo "=== VERSIONES DE LAS HERRAMIENTAS ==="
                    flake8 --version
                    bandit --version

                    mkdir -p reports

                    echo "=== EJECUTANDO FLAKE8 ==="

                    flake8 \
                        --exit-zero \
                        --format=pylint \
                        src > reports/flake8.out

                    test -f reports/flake8.out

                    echo "=== EJECUTANDO BANDIT ==="

                    set +e

                    bandit \
                        -r src \
                        -f custom \
                        -o reports/bandit.out \
                        --msg-template "{abspath}:{line}: [{test_id}] {msg}"

                    BANDIT_EXIT=$?

                    set -e

                    test -f reports/bandit.out

                    if [ "$BANDIT_EXIT" -gt 1 ]; then
                        echo "Bandit ha fallado con código $BANDIT_EXIT"
                        exit "$BANDIT_EXIT"
                    fi

                    echo "=== INFORMES GENERADOS ==="
                    ls -la reports
                '''

                recordIssues(
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

                    echo "=== VERSIONES DE AWS Y SAM ==="
                    aws --version
                    sam --version

                    echo "=== IDENTIDAD AWS ==="
                    aws sts get-caller-identity

                    echo "=== VALIDANDO TEMPLATE SAM ==="
                    sam validate --template-file template.yaml

                    echo "=== CONSTRUYENDO APLICACIÓN SAM ==="
                    sam build --template-file template.yaml

                    restore_samconfig() {
                        if [ -f samconfig.toml.disabled ]; then
                            mv samconfig.toml.disabled samconfig.toml
                        fi
                    }

                    if [ -f samconfig.toml ]; then
                        mv samconfig.toml samconfig.toml.disabled
                    fi

                    trap restore_samconfig EXIT

                    echo "=== DESPLEGANDO EN STAGING ==="

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

                    echo "=== OBTENIENDO URL DE LA API ==="

                    API_URL=$(aws cloudformation describe-stacks \
                        --stack-name "$STACK_NAME" \
                        --region "$AWS_DEFAULT_REGION" \
                        --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                        --output text)

                    if [ -z "$API_URL" ] || [ "$API_URL" = "None" ]; then
                        echo "No se ha encontrado BaseUrlApi"
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

                    echo "=== VERSIÓN DE PYTEST ==="
                    pytest --version

                    mkdir -p reports

                    export BASE_URL="$(cat api_url.txt)"

                    if [ -z "$BASE_URL" ]; then
                        echo "BASE_URL está vacía"
                        exit 1
                    fi

                    echo "=================================================="
                    echo "PRUEBAS REST"
                    echo "API utilizada: $BASE_URL"
                    echo "Archivo de pruebas: $TEST_FILE"
                    echo "=================================================="

                    pytest \
                        -v \
                        -s \
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

                        git config user.name "Jenkins CI"
                        git config user.email "jenkins@localhost"

                        git remote set-url origin \
                            "https://${GIT_USERNAME}:${GIT_TOKEN}@github.com/siberett/todo-list-aws.git"

                        git fetch origin development master

                        git checkout -B master origin/master

                        git merge \
                            --no-ff \
                            -X ours \
                            origin/development \
                            -m "Promote development to master"

                        git push origin master
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completado correctamente.'
        }

        failure {
            echo 'Pipeline fallido.'
        }

        always {
            cleanWs()
        }
    }
}
