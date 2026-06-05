pipeline {
    agent any

    stages {
        stage('Install Dependencies') {
            steps {
                sh '''
                    curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
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

        stage('Deploy to Target') {
            steps {
                sh '''
                    mkdir -p ~/.ssh
                    ssh-keyscan -H target >> ~/.ssh/known_hosts
                    ssh -i ~/.ssh/id_ed25519 laborant@target "
                        sudo mkdir -p /opt/myapp/node_modules &&
                        sudo chown -R laborant:laborant /opt/myapp
                    "
                    scp -i ~/.ssh/id_ed25519 index.js laborant@target:/opt/myapp/index.js
                    scp -i ~/.ssh/id_ed25519 -r node_modules/. laborant@target:/opt/myapp/node_modules/
                    ssh -i ~/.ssh/id_ed25519 laborant@target "
                        sudo chown -R myapp:myapp /opt/myapp &&
                        sudo systemctl restart myapp
                    "
                '''
            }
        }

        stage('Deploy to Docker') {
            steps {
                sh '''
                    ssh-keyscan -H docker >> ~/.ssh/known_hosts
                    ssh -i ~/.ssh/id_ed25519 laborant@docker "
                        docker pull ttl.sh/renn-amm-capstone:2h &&
                        docker rm -f myapp 2>/dev/null || true &&
                        docker run -d --name myapp -p 4444:4444 ttl.sh/renn-amm-capstone:2h
                    "
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
                        kubectl delete pod myapp --ignore-not-found
                        kubectl apply -f k8s/pod.yaml
                        kubectl wait --for=condition=Ready pod/myapp --timeout=120s
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
