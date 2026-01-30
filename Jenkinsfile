pipeline {
    agent any

    stages {

        stage('Secret Scan') {
            steps {
                sh '''
                mkdir -p security-reports/trufflehog
                trufflehog filesystem . --json > security-reports/trufflehog/trufflehog.json || true

                # Simple HTML wrapper
                echo "<html><body><h1>TruffleHog Report</h1><pre>" \
                  > security-reports/trufflehog/trufflehog.html
                cat security-reports/trufflehog/trufflehog.json \
                  >> security-reports/trufflehog/trufflehog.html
                echo "</pre></body></html>" \
                  >> security-reports/trufflehog/trufflehog.html
                '''
            }
        }

        stage('Source-Composition-Analysis') {
            steps {
                sh '''
                mkdir -p security-reports/dependency-check

                rm -f owasp-dependency-check.sh
                wget https://raw.githubusercontent.com/devopssecure/webapp/master/owasp-dependency-check.sh
                chmod +x owasp-dependency-check.sh

                docker run --rm \
                  -v "$WORKSPACE:/src" \
                  -v "$WORKSPACE/security-reports/dependency-check:/report" \
                  owasp/dependency-check:latest \
                  --scan /src \
                  --format HTML \
                  --out /report
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

        stage('Archive Security Reports') {
            steps {
                archiveArtifacts artifacts: 'security-reports/**', fingerprint: true
            }
        }
    }
}
