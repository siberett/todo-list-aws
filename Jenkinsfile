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

                    if [ ! -f reports/flake8.out ]; then
                        echo "ERROR: Flake8 no ha generado el informe."
                        exit 1
                    fi

                    echo "--- Bandit sobre src/ ---"

                    set +e

                    bandit \
                        -r src \
                        -f custom \
                        -o reports/bandit.out \
                        --msg-template "{abspath}:{line}: [{test_id}] {msg}"

                    BANDIT_EXIT=$?

                    set -e

                    if [ ! -f reports/bandit.out ]; then
                        echo "ERROR: Bandit no ha generado el informe."
                        exit 1
                    fi

                    if [ "$BANDIT_EXIT" -gt 1 ]; then
                        echo "ERROR: Bandit ha fallado técnicamente con código $BANDIT_EXIT."
                        exit "$BANDIT_EXIT"
                    fi

                    echo "Bandit ha finalizado con código $BANDIT_EXIT."

                    if [ "$BANDIT_EXIT" -eq 1 ]; then
                        echo "Bandit ha encontrado hallazgos."
                        echo "Los hallazgos no bloquean el pipeline porque no existen Quality Gates."
                    fi

                    echo "--- Informes generados ---"
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

                    echo "=== DEPLOY ==="
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

                    echo "--- Despliegue en Staging ---"
                    echo "Stack: $STACK_NAME"
                    echo "Región: $AWS_DEFAULT_REGION"
                    echo "Stage: $SAM_ENVIRONMENT"

                    restore_samconfig() {
                        if [ -f samconfig.toml.disabled ]; then
                            mv samconfig.toml.disabled samconfig.toml
                        fi
                    }

                    if [ -f samconfig.toml ]; then
                        mv samconfig.toml samconfig.toml.disabled
                    fi

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
                        echo "ERROR: No se ha encontrado el output BaseUrlApi."
                        exit 1
                    fi

                    echo "API URL obtenida:"
                    echo "$API_URL"

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

                    if [ ! -f api_url.txt ]; then
                        echo "ERROR: No existe el fichero api_url.txt."
                        exit 1
                    fi

                    export BASE_URL="$(cat api_url.txt)"

                    if [ -z "$BASE_URL" ]; then
                        echo "ERROR: BASE_URL está vacía."
                        exit 1
                    fi

                    echo "Ejecutando pruebas contra:"
                    echo "$BASE_URL"

                    if [ ! -f "$TEST_FILE" ]; then
                        echo "ERROR: No existe el fichero de pruebas $TEST_FILE."
                        exit 1
                    fi

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

                        echo "=== PROMOTE ==="
                        whoami
                        hostname
                        pwd

                        echo "--- Configuración Git ---"

                        git config user.name "Jenkins CI"
                        git config user.email "jenkins@localhost"

                        git remote set-url origin \
                            "https://${GIT_USERNAME}:${GIT_TOKEN}@github.com/siberett/todo-list-aws.git"

                        echo "--- Actualización de ramas remotas ---"

                        git fetch origin development
                        git fetch origin master

                        echo "--- Checkout de master ---"

                        git checkout -B master origin/master

                        echo "--- Merge development en master ---"

                        git merge \
                            --no-ff \
                            origin/development \
                            -m "Promote development to master from Jenkins CI"

                        echo "--- Push de master ---"

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
            echo 'Pipeline CI completado correctamente. Development se ha mergeado en master.'
        }

        failure {
            echo 'Pipeline CI fallido. El código no se ha promocionado a master.'
        }

        always {
            cleanWs()
        }
    }
}
