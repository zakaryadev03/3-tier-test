pipeline {
    agent any
    tools {
        nodejs 'nodejs23'
    }
    environment{
        SCANNER_HOME = tool "sonar-scanner"
    }
    stages {
        stage('Git checkout') {
            steps {
                git branch: 'dev', url: 'https://github.com/zakaryadev03/3-tier-test.git'
            }
        }
        
        stage('Frontend compilation') {
            steps {
                dir('client'){
                    sh 'find . -name "*.js" -exec node -c {} +'
                }
            }
        }
        
        stage('Backend compilation') {
            steps {
                dir('api'){
                    sh 'find . -name "*.js" -exec node -c {} +'
                }
            }
        }
        
        stage('Gitleaks scan') {
            steps {
                sh 'gitleaks detect --source ./client --exit-code 1'
                sh 'gitleaks detect --source ./api --exit-code 1'
            }
        }

        stage('SonarQube analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=NodeJS-Project \
                            -Dsonar.projectKey=NodeJS-Project '''
                }
            }
        }
        
        stage('Quality gate check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        
        stage('Trivy Fs Scan') {
            steps {
                sh 'trivy fs --format table -o fs-report.html .'
            }
        }
        
        stage('Build Docker Backend Image') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        dir('api'){
                            sh 'docker build -t zakaryab2003/backend-devsecops:latest .'
                            sh 'trivy image --format table -o Backend-image-report.html zakaryab2003/backend-devsecops:latest'
                            sh 'docker push zakaryab2003/backend-devsecops:latest'
                        }
                    }
                }
            }
        }
        
        stage('Build Docker Frontend Image') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        dir('client'){
                            sh 'docker build -t zakaryab2003/frontend-devsecops:latest .'
                            sh 'trivy image --format table -o Frontend-image-report.html zakaryab2003/frontend-devsecops:latest'
                            sh 'docker push zakaryab2003/frontend-devsecops:latest'
                        }
                    }
                }
            }
        }

        stage('K8-deploy') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'sudcloud-cluster', contextName: '', credentialsId: 'k8-token', namespace: 'dev', restrictKubeConfigAccess: false, serverUrl: 'https://042CDFB283295491DDF7G7A422346F3B.gr7.us-east-1.eks.amazonaws.com') {
                            sh 'kubectl apply -f k8s/sc.yaml -n dev'
                            sh 'kubectl apply -f k8s/mysql.yaml -n dev'
                            sh 'kubectl apply -f k8s/backend.yaml -n dev'
                            sh 'kubectl apply -f k8s/frontend.yaml -n dev'
                            sleep 30
                    }
                }
            }
        }
        
        stage('verify-K8-deploy') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'sudcloud-cluster', contextName: '', credentialsId: 'k8-token', namespace: 'dev', restrictKubeConfigAccess: false, serverUrl: 'https://042CDFB283295491DDF7G7A422346F3B.gr7.us-east-1.eks.amazonaws.com') {
                            sh 'kubectl get pods -n dev'
                            sh 'kubectl get svc -n dev'    
                    }
                }
            }
        }
        
    }
}
