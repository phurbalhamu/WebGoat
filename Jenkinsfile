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
