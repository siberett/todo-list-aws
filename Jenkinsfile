stage('Rest Test') {
    steps {
        sh '''
            set -e

            pytest --version

            mkdir -p reports

            export BASE_URL="$(cat api_url.txt)"

            if [ -z "$BASE_URL" ]; then
                echo "BASE_URL está vacía"
                exit 1
            fi

            echo "API utilizada para las pruebas: $BASE_URL"

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
