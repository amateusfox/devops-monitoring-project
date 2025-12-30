pipeline {
    agent any
    
    environment {
        DOMAIN_PROM = 'prom.amateusfox.online'
        DOMAIN_GRAFANA = 'grafana.amateusfox.online'
        MONITORING_VM_IP = '194.67.124.52'
        CONTROL_VM_IP = '95.163.226.71'
    }
    
    stages {
        stage('Checkout') {
            steps {
                cleanWs()
                checkout scm
                sh 'ls -la'
            }
        }
        
        stage('Prepare Environment') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'ansible-ssh-key',
                    keyFileVariable: 'SSH_KEY_FILE',
                    usernameVariable: 'SSH_USER'
                )]) {
                    sh '''
                        mkdir -p ~/.ssh
                        cp "${SSH_KEY_FILE}" ~/.ssh/id_ansible_ed25519
                        chmod 600 ~/.ssh/id_ansible_ed25519
                        echo "SSH key ready for user: ${SSH_USER}"
                    '''
                }
                
                sh '''
                    cat > inventory.ini << EOF
[monitoring]
monitoring-vm ansible_host=${MONITORING_VM_IP} ansible_user=devops ansible_ssh_private_key_file=~/.ssh/id_ansible_ed25519 ansible_ssh_common_args='-o StrictHostKeyChecking=no'

[control]
control-vm ansible_host=${CONTROL_VM_IP} ansible_user=devops ansible_ssh_private_key_file=~/.ssh/id_ansible_ed25519 ansible_ssh_common_args='-o StrictHostKeyChecking=no'

[all:children]
monitoring
control
EOF
                    echo "Inventory created"
                    cat inventory.ini
                '''
                
                sh '''
                    if ! command -v ansible &> /dev/null; then
                        echo "Installing Ansible..."
                        sudo apt-get update
                        sudo apt-get install -y ansible
                    fi
                '''
            }
        }
        
        stage('Deploy with Ansible') {
            steps {
                withCredentials([
                    string(credentialsId: 'grafana-admin-pass', variable: 'GRAFANA_PASS'),
                    string(credentialsId: 'certbot-email', variable: 'CERTBOT_EMAIL')
                ]) {
                    sh '''
                        echo "Starting deployment..."
                        echo "Prometheus: ${DOMAIN_PROM}"
                        echo "Grafana: ${DOMAIN_GRAFANA}"
                        
                        export DOMAIN_PROM="${DOMAIN_PROM}"
                        export DOMAIN_GRAFANA="${DOMAIN_GRAFANA}"
                        export CERTBOT_EMAIL="${CERTBOT_EMAIL}"
                        export CONTROL_VM_IP="${CONTROL_VM_IP}"
                        export GRAFANA_ADMIN_PASSWORD="${GRAFANA_PASS}"
                        export ANSIBLE_HOST_KEY_CHECKING=False
                        
                        ansible-playbook -i inventory.ini playbook.yml -v
                        
                        if [ $? -eq 0 ]; then
                            echo "Deployment successful"
                        else
                            echo "Deployment failed"
                            exit 1
                        fi
                    '''
                }
            }
        }
        
        stage('Health Checks') {
            steps {
                sleep 30
                
                // 1. Check Prometheus
                sh '''
                    echo "=== Checking Prometheus ==="
                    if curl -f -L --max-time 30 https://${DOMAIN_PROM}/-/healthy; then
                        echo "Prometheus: HEALTHY"
                    else
                        echo "Prometheus: UNHEALTHY"
                        exit 1
                    fi
                '''
                
                // 2. Check Grafana
                sh '''
                    echo "=== Checking Grafana ==="
                    if curl -f -L --max-time 30 https://${DOMAIN_GRAFANA}/api/health; then
                        echo "Grafana: HEALTHY"
                    else
                        echo "Grafana: UNHEALTHY"
                        exit 1
                    fi
                '''
                
                // 3. Check SSL
                sh '''
                    echo "=== Checking SSL ==="
                    echo "Testing certificate for ${DOMAIN_PROM}"
                    if openssl s_client -connect ${DOMAIN_PROM}:443 -servername ${DOMAIN_PROM} < /dev/null 2>/dev/null | openssl x509 -noout -checkend 0; then
                        echo "SSL certificate: VALID"
                    else
                        echo "SSL certificate: INVALID"
                        exit 1
                    fi
                '''
                
                // 4. Check Node Exporter metrics (SIMPLIFIED - без квадратных скобок)
                sh '''
                    echo "=== Checking Metrics ==="
                    echo "Querying Prometheus for metrics..."
                    
                    # Простой запрос к Prometheus API
                    RESPONSE=$(curl -s https://${DOMAIN_PROM}/api/v1/query?query=up)
                    
                    # Проверяем что ответ содержит данные (без сложных регулярных выражений)
                    if echo "$RESPONSE" | grep -q "node_exporter_control"; then
                        echo "Node Exporter metrics: FOUND"
                    else
                        echo "Node Exporter metrics: NOT FOUND"
                        echo "Response sample:"
                        echo "$RESPONSE" | head -5
                        exit 1
                    fi
                    
                    # Проверяем raw metrics
                    echo "Checking raw metrics endpoint..."
                    curl -s https://${DOMAIN_PROM}/metrics | grep -E "node_cpu|node_memory" | head -2
                '''
                
                // 5. Check Docker containers
                sh '''
                    echo "=== Checking Docker Containers ==="
                    ssh -i ~/.ssh/id_ansible_ed25519 -o StrictHostKeyChecking=no devops@${MONITORING_VM_IP} '
                        echo "Docker containers on monitoring-vm:"
                        docker ps --format "{{.Names}}: {{.Status}}"
                        
                        COUNT=$(docker ps -q | wc -l)
                        echo "Running containers: $COUNT"
                        
                        if [ $COUNT -ge 2 ]; then
                            echo "Docker: OK"
                        else
                            echo "Docker: ERROR - not enough containers"
                            exit 1
                        fi
                    '
                '''
            }
        }
        
        stage('Final Verification') {
            steps {
                sh '''
                    echo "=== Final Verification ==="
                    echo "All services should be accessible:"
                    echo "1. Prometheus: https://${DOMAIN_PROM}"
                    echo "2. Grafana: https://${DOMAIN_GRAFANA}"
                    echo ""
                    echo "Testing connectivity..."
                    
                    # Quick connectivity test
                    curl -s -o /dev/null -w "Prometheus: %{http_code}\n" https://${DOMAIN_PROM}/graph
                    curl -s -o /dev/null -w "Grafana: %{http_code}\n" https://${DOMAIN_GRAFANA}/login
                    
                    echo ""
                    echo "Deployment completed successfully!"
                '''
            }
        }
    }
    
    post {
        success {
            echo '✅ Pipeline SUCCESS - Monitoring stack deployed'
            sh '''
                echo "=========================================="
                echo "MONITORING STACK DEPLOYED SUCCESSFULLY"
                echo "=========================================="
                echo "Prometheus: https://${DOMAIN_PROM}"
                echo "Grafana:    https://${DOMAIN_GRAFANA}"
                echo "Credentials: admin / (check Jenkins secrets)"
                echo "=========================================="
            '''
        }
        failure {
            echo '❌ Pipeline FAILED - Check logs above'
        }
        always {
            sh '''
                rm -f ~/.ssh/id_ansible_ed25519
                echo "Cleanup completed"
            '''
        }
    }
}
