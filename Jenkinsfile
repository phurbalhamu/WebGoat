pipeline {
    agent any

    environment {
        PRODUCT_NAME = "Jenkins-CICD-App"
        ENGAGEMENT_NAME = "CI Scan"
    }

    stages {

        stage('Secret Scan') {
            steps {
                sh '''
                set +e
                mkdir -p security-reports/trufflehog

                trufflehog filesystem . --json \
                  > security-reports/trufflehog/trufflehog.json

                echo "<html><body><h1>TruffleHog Report</h1><pre>" \
                  > security-reports/trufflehog/trufflehog.html
                cat security-reports/trufflehog/trufflehog.json \
                  >> security-reports/trufflehog/trufflehog.html
                echo "</pre></body></html>" \
                  >> security-reports/trufflehog/trufflehog.html

                echo "===== TruffleHog JSON (first 50 lines) ====="
                head -50 security-reports/trufflehog/trufflehog.json || true
                '''
            }
        }

        stage('Source-Composition-Analysis') {
            steps {
                sh '''
                set -e
                mkdir -p security-reports/dependency-check

                docker run --rm \
                  -v "$WORKSPACE:/src" \
                  -v "$WORKSPACE/security-reports/dependency-check:/report" \
                  owasp/dependency-check:latest \
                  --scan /src \
                  --format JSON,HTML \
                  --out /report

                test -f security-reports/dependency-check/dependency-check-report.json

                echo "===== Dependency-Check JSON (first 50 lines) ====="
                head -50 security-reports/dependency-check/dependency-check-report.json
                '''
            }
        }

        stage('Upload to DefectDojo') {
            steps {
                withCredentials([
                    string(credentialsId: 'DEFECTDOJO_API_KEY', variable: 'DD_API_KEY'),
                    string(credentialsId: 'DEFECTDOJO_URL', variable: 'DD_URL')
                ]) {
                    sh '''
                    set -e

                    command -v jq >/dev/null || { echo "jq is required"; exit 1; }

                    echo "Finding or creating product..."
                    PRODUCT_ID=$(curl -s -H "Authorization: Token $DD_API_KEY" \
                      "$DD_URL/api/v2/products/?name=$PRODUCT_NAME" | jq -r '.results[0].id')

                    if [ "$PRODUCT_ID" = "null" ] || [ -z "$PRODUCT_ID" ]; then
                      PRODUCT_ID=$(curl -s -X POST \
                        -H "Authorization: Token $DD_API_KEY" \
                        -H "Content-Type: application/json" \
                        -d "{\"name\":\"$PRODUCT_NAME\",\"prod_type\":1}" \
                        "$DD_URL/api/v2/products/" | jq -r '.id')
                    fi

                    echo "Product ID: $PRODUCT_ID"

                    echo "Creating engagement..."
                    ENGAGEMENT_ID=$(curl -s -X POST \
                      -H "Authorization: Token $DD_API_KEY" \
                      -H "Content-Type: application/json" \
                      -d "{\"name\":\"$ENGAGEMENT_NAME\",\"product\":$PRODUCT_ID,\"engagement_type\":\"CI/CD\",\"status\":\"In Progress\"}" \
                      "$DD_URL/api/v2/engagements/" | jq -r '.id')

                    echo "Engagement ID: $ENGAGEMENT_ID"

                    echo "Uploading Dependency-Check report..."
                    curl -s -X POST "$DD_URL/api/v2/import-scan/" \
                      -H "Authorization: Token $DD_API_KEY" \
                      -F "scan_type=Dependency Check Scan" \
                      -F "file=@security-reports/dependency-check/dependency-check-report.json" \
                      -F "engagement=$ENGAGEMENT_ID" \
                      -F "active=true" \
                      -F "verified=false" \
                      -F "close_old_findings=true"

                    echo "DefectDojo upload successful."
                    '''
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying the application'
            }
        }

        stage('Archive Reports') {
            steps {
                archiveArtifacts artifacts: 'security-reports/**', fingerprint: true
            }
        }
    }
}
