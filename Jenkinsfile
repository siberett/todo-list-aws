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
        STACK_NAME = 'todo-list-aws-production'
        SAM_ENVIRONMENT = 'production'

        TEST_FILE = 'test/integration/todoApiTest.py'
    }

    stages {

        stage('Get Code') {
            steps {
                cleanWs()

                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/master']],
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
                '''
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

                    echo "=== DESPLEGANDO EN PRODUCCIÓN ==="

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

                    echo "=== OBTENIENDO URL DE PRODUCCIÓN ==="

                    API_URL=$(aws cloudformation describe-stacks \
                        --stack-name "$STACK_NAME" \
                        --region "$AWS_DEFAULT_REGION" \
                        --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                        --output text)

                    if [ -z "$API_URL" ] || [ "$API_URL" = "None" ]; then
                        echo "ERROR: no se ha encontrado BaseUrlApi"
                        exit 1
                    fi

                    echo "API URL de producción: $API_URL"

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

                    if [ ! -f api_url.txt ]; then
                        echo "ERROR: no existe api_url.txt"
                        exit 1
                    fi

                    export BASE_URL="$(cat api_url.txt)"

                    if [ -z "$BASE_URL" ]; then
                        echo "ERROR: BASE_URL está vacía"
                        exit 1
                    fi

                    echo "=================================================="
                    echo "PRUEBAS REST DE PRODUCCIÓN"
                    echo "API utilizada: $BASE_URL"
                    echo "Pruebas seleccionadas:"
                    echo "- test_api_listtodos"
                    echo "- test_api_gettodo"
                    echo "=================================================="

                    pytest \
                        -v \
                        -s \
                        "$TEST_FILE" \
                        -k "test_api_listtodos or test_api_gettodo" \
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
    }

    post {
        success {
            echo 'Pipeline CD completado correctamente.'
        }

        failure {
            echo 'Pipeline CD fallido.'
        }

        always {
            cleanWs()
        }
    }
}
