pipeline {
    agent any
    tools {
        maven 'maven3'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git', url: 'https://github.com/VanishaParwal/bank-app.git'
            }
        }
        stage('Compile') {
            steps {
                sh 'mvn compile'
                echo "compile complete"
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test -Dspring.profiles.active=test'
                echo "test complete"
            }
        }
        stage('Trivy Fs Scan') {
            steps {
                sh 'trivy fs --format table -o fs-report.html .'
                echo "trivy scan"
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=GCBank \
                        -Dsonar.projectKey=GCBank \
                        -Dsonar.java.binaries=target
                    '''
                    echo 'sonarqube analysis complete'
                }
            }
        }
        stage('Quality Gate Check') {
            steps {
                echo 'Quality check starts here'
                timeout (time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        stage('Build') {
            steps {
                sh 'mvn package'
            }
        }
        stage('Publish Artifacts') {
            steps {
                withMaven(globalMavenSettingsConfig: 'vani', jdk: '', maven: 'maven3') {
                    sh 'mvn deploy'
                }
            }
        }
        stage('Build and Tag Docker Images') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dcker-cred') {
                        sh "docker build -t vanishaparwal/bankapp:${IMAGE_TAG} ."
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --format table -o image-report.html vanishaparwal/bankapp:${IMAGE_TAG}"
                echo "trivy scan"
            }
        }
        stage('Push Docker Images') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dcker-cred') {
                        sh "docker push vanishaparwal/bankapp:${IMAGE_TAG}"
                    }
                }
            }
        }
        stage('Update manifest.yaml and push to GitHub') {
            steps {
                script {
                    // Update the image tag in manifest.yaml
                    sh """
                        sed -i 's|image: vanishaparwal/bankapp:.*|image: vanishaparwal/bankapp:${IMAGE_TAG}|g' manifest.yaml
                    """

                    // Configure Git user if needed (use a generic CI email)
                    sh '''
                        git config --global user.email "jenkins@cicd.local"
                        git config --global user.name "Jenkins CI"
                    '''

                    // Add, commit, and push the changes
                    sh """
                        git add manifest.yaml
                        git commit -m "Update manifest.yaml image tag to ${IMAGE_TAG} [ci skip]" || echo "No changes to commit"
                        git push origin main
                    """
                }
            }
        }
    }
}
