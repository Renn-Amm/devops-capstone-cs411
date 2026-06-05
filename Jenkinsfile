pipeline {
    agent any

    stages {
        stage('Install Dependencies') {
            steps {
                sh '''
                    curl -fsSL https://deb.nodesource.com/setup_24.x -o nodesource_setup.sh
                    sudo -E bash nodesource_setup.sh
                    sudo apt-get install -y nodejs
                    npm install
                '''
            }
        }

        stage('Test') {
            steps {
                sh 'node --test'
            }
        }

        stage('Docker Build and Push') {
            steps {
                sh '''
                    docker build -t ttl.sh/renn-amm-capstone:2h .
                    docker push ttl.sh/renn-amm-capstone:2h
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([string(credentialsId: 'k8s-token', variable: 'K8S_TOKEN')]) {
                    sh '''
                        kubectl config set-cluster k8s \
                            --server=https://kubernetes:6443 \
                            --insecure-skip-tls-verify=true
                        kubectl config set-credentials jenkins-robot \
                            --token=$K8S_TOKEN
                        kubectl config set-context k8s \
                            --cluster=k8s \
                            --user=jenkins-robot
                        kubectl config use-context k8s
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml
                        kubectl rollout status deployment/myapp --timeout=120s
                    '''
                }
            }
        }
        stage('Health Check') {
            steps {
                withCredentials([string(credentialsId: 'k8s-token', variable: 'K8S_TOKEN')]) {
                    sh '''
                        kubectl config use-context k8s
                        POD_IP=$(kubectl get pod -l app=myapp -o jsonpath='{.items[0].status.podIP}')
                        kubectl run curl-test \
                            --image=busybox \
                            --rm \
                            --restart=Never \
                            -it \
                            -- wget -qO- http://$POD_IP:4444/
                    '''
                }
            }
        }
    }
}
