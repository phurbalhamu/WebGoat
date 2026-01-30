pipeline {
    agent any

    stages {

        stage('Secret Scan') {
            steps {
                sh '''
                mkdir -p security-reports/trufflehog

                trufflehog filesystem . --json \
                  > security-reports/trufflehog/trufflehog.json || true

                echo "<html><body><h1>TruffleHog Report</h1><pre>" \
                  > security-reports/trufflehog/trufflehog.html
                cat security-reports/trufflehog/trufflehog.json \
                  >> security-reports/trufflehog/trufflehog.html
                echo "</pre></body></html>" \
                  >> security-reports/trufflehog/trufflehog.html

                echo "===== TruffleHog JSON Report (first 200 lines) ====="
                head -200 security-reports/trufflehog/trufflehog.json || true
                '''
            }
        }

        stage('Source-Composition-Analysis') {
            steps {
                sh '''
                mkdir -p security-reports/dependency-check

                docker run --rm \
                  -v "$WORKSPACE:/src" \
                  -v "$WORKSPACE/security-reports/dependency-check:/report" \
                  owasp/dependency-check:latest \
                  --scan /src \
                  --format ALL \
                  --out /report

                echo "===== Dependency-Check JSON Report (first 200 lines) ====="
                head -200 security-reports/dependency-check/dependency-check-report.json || true
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

            echo "Creating product (if not exists)..."
            PRODUCT_ID=$(curl -s -H "Authorization: Token $DD_API_KEY" \
              -H "Content-Type: application/json" \
              -d '{"name":"Jenkins-CICD-App","description":"Created by Jenkins","prod_type":1}' \
              $DD_URL/api/v2/products/ | jq -r '.id')

            echo "Product ID: $PRODUCT_ID"

            echo "Creating engagement..."
            ENGAGEMENT_ID=$(curl -s -H "Authorization: Token $DD_API_KEY" \
              -H "Content-Type: application/json" \
              -d "{\"name\":\"CI Scan\",\"product\":$PRODUCT_ID,\"status\":\"In Progress\",\"engagement_type\":\"CI/CD\"}" \
              $DD_URL/api/v2/engagements/ | jq -r '.id')

            echo "Engagement ID: $ENGAGEMENT_ID"

            echo "Uploading Dependency-Check report..."
            curl -X POST "$DD_URL/api/v2/import-scan/" \
              -H "Authorization: Token $DD_API_KEY" \
              -F "scan_type=Dependency Check Scan" \
              -F "file=@security-reports/dependency-check/dependency-check-report.json" \
              -F "engagement=$ENGAGEMENT_ID" \
              -F "active=true" \
              -F "verified=false" \
              -F "close_old_findings=true"

            echo "Upload complete."
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
                archiveArtifacts artifacts: 'security-reports/**'
            }
        }
    }
}
