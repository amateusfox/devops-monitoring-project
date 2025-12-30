pipeline {
    agent any
    
    environment {
        // Переменные окружения - можно вынести в .env файл позже
        MONITORING_VM = 'monitoring-vm'  // Имя вашей второй ВМ в инвентаре
        PROMETHEUS_URL = 'https://prom.your-domain.com'  // Ваш домен Prometheus
        GRAFANA_URL = 'http://monitoring-vm:3000'
        NODE_EXPORTER_PORT = '9100'
    }
    
    stages {
        // ==================== ЭТАП 1: Клонирование репозитория ====================
        stage('Checkout Repository') {
            steps {
                echo "🔽 Клонируем репозиторий из GitHub..."
                git(
                    url: 'git@github.com:amateusfox/devops-monitoring-project.git',
                    branch: 'main'
                )
                
                // Проверяем, что файлы склонировались
                sh '''
                    echo "📂 Содержимое репозитория:"
                    ls -la
                    echo ""
                    echo "📁 Ansible роль monitoring:"
                    ls -la roles/monitoring/
                '''
            }
        }
        
        // ==================== ЭТАП 2: Загрузка переменных (опционально) ====================
        stage('Load Environment Variables') {
            steps {
                script {
                    if (fileExists('.env')) {
                        echo "📄 Загружаем переменные из .env файла"
                        def props = readProperties file: '.env'
                        props.each { key, value ->
                            env[key] = value
                            echo "  ${key}=${value}"
                        }
                    } else {
                        echo "ℹ️ .env файл не найден, используем переменные по умолчанию"
                    }
                }
            }
        }
        
        // ==================== ЭТАП 3: Развёртывание стека мониторинга ====================
        stage('Deploy Monitoring Stack') {
            steps {
                echo "🚀 Разворачиваем стек мониторинга на ${MONITORING_VM}..."
                
                sh """
                    cd ${WORKSPACE}
                    echo "Запускаем Ansible playbook..."
                    # Запускаем вашу роль monitoring на целевой ВМ
                    ansible-playbook -i inventory.ini playbook.yml --tags "monitoring" --limit ${MONITORING_VM}
                    
                    if [ \$? -eq 0 ]; then
                        echo "✅ Ansible выполнен успешно"
                    else
                        echo "❌ Ошибка в Ansible"
                        exit 1
                    fi
                """
            }
        }
        
        // ==================== ЭТАП 4: Проверка откликов UI ====================
        stage('Verify UI Availability') {
            steps {
                echo "🌐 Проверяем доступность интерфейсов..."
                
                // Проверка Prometheus через Nginx (HTTPS)
                sh """
                    echo "Проверяем Prometheus (${PROMETHEUS_URL})..."
                    HTTP_CODE=\$(curl -s -o /dev/null -w "%{http_code}" -k ${PROMETHEUS_URL}/-/healthy --connect-timeout 10)
                    
                    if [ "\$HTTP_CODE" = "200" ]; then
                        echo "✅ Prometheus доступен (код: \$HTTP_CODE)"
                    else
                        echo "❌ Prometheus недоступен (код: \$HTTP_CODE)"
                        exit 1
                    fi
                """
                
                // Проверка Grafana
                sh """
                    echo "Проверяем Grafana (${GRAFANA_URL})..."
                    HTTP_CODE=\$(curl -s -o /dev/null -w "%{http_code}" ${GRAFANA_URL}/api/health --connect-timeout 10)
                    
                    if [ "\$HTTP_CODE" = "200" ]; then
                        echo "✅ Grafana доступна (код: \$HTTP_CODE)"
                    else
                        echo "❌ Grafana недоступна (код: \$HTTP_CODE)"
                        exit 1
                    fi
                """
            }
        }
        
        // ==================== ЭТАП 5: Проверка SSL-сертификатов ====================
        stage('Verify SSL Certificates') {
            steps {
                echo "🔐 Проверяем SSL-сертификаты..."
                
                sh """
                    echo "Проверяем сертификат для ${PROMETHEUS_URL}..."
                    
                    # Проверяем срок действия сертификата
                    CERT_INFO=\$(echo | openssl s_client -connect ${PROMETHEUS_URL#https://}:443 -servername ${PROMETHEUS_URL#https://} 2>/dev/null | openssl x509 -noout -dates 2>/dev/null || echo "ERROR")
                    
                    if [ "\$CERT_INFO" != "ERROR" ]; then
                        echo "✅ Сертификат валиден:"
                        echo "\$CERT_INFO"
                        
                        # Проверяем, не истёк ли срок
                        NOT_AFTER=\$(echo "\$CERT_INFO" | grep "notAfter" | cut -d= -f2)
                        echo "Срок действия до: \$NOT_AFTER"
                    else
                        echo "❌ Не удалось проверить сертификат"
                        exit 1
                    fi
                """
            }
        }
        
        // ==================== ЭТАП 6: Проверка метрик ====================
        stage('Verify Metrics Endpoint') {
            steps {
                echo "📊 Проверяем метрики..."
                
                // Проверка Node Exporter на monitoring-vm
                sh """
                    echo "Проверяем Node Exporter на ${MONITORING_VM}:${NODE_EXPORTER_PORT}..."
                    
                    METRICS_OUTPUT=\$(curl -s --connect-timeout 10 http://${MONITORING_VM}:${NODE_EXPORTER_PORT}/metrics || echo "ERROR")
                    
                    if echo "\$METRICS_OUTPUT" | grep -q "node_cpu_seconds_total"; then
                        echo "✅ Node Exporter отдаёт метрики CPU"
                    else
                        echo "❌ Node Exporter не отдаёт метрики CPU"
                        exit 1
                    fi
                    
                    if echo "\$METRICS_OUTPUT" | grep -q "node_memory_MemAvailable_bytes"; then
                        echo "✅ Node Exporter отдаёт метрики памяти"
                    else
                        echo "❌ Node Exporter не отдаёт метрики памяти"
                        exit 1
                    fi
                """
                
                // Проверка, что Prometheus видит таргет
                sh """
                    echo "Проверяем, видит ли Prometheus таргет..."
                    
                    TARGETS_JSON=\$(curl -s -k ${PROMETHEUS_URL}/api/v1/targets || echo "ERROR")
                    
                    if echo "\$TARGETS_JSON" | grep -q '"health":"up"'; then
                        echo "✅ Prometheus видит активные таргеты"
                    else
                        echo "❌ Prometheus не видит активные таргеты"
                        echo "Ответ от API:"
                        echo "\$TARGETS_JSON" | head -5
                        exit 1
                    fi
                """
            }
        }
    }
    
    post {
        always {
            echo "=========================================="
            echo "🏁 Сборка #${BUILD_NUMBER} завершена"
            echo "Статус: ${currentBuild.result ?: 'SUCCESS'}"
            echo "=========================================="
            
            // Очистка (опционально)
            sh '''
                echo "Очищаем временные файлы..."
                # Можно добавить очистку кэша и т.д.
            '''
        }
        
        success {
            echo "🎉 Все этапы выполнены успешно!"
            echo "Стек мониторинга развёрнут и проверен"
            
            // Можно добавить уведомления (email, telegram, slack)
            // emailext (
            //     subject: "SUCCESS: Pipeline '${JOB_NAME}' #${BUILD_NUMBER}",
            //     body: "Все этапы выполнены успешно.",
            //     to: "your-email@example.com"
            // )
        }
        
        failure {
            echo "💥 Сборка завершилась с ошибкой"
            echo "Проверьте логи для диагностики"
            
            // Можно добавить уведомления об ошибке
            // emailext (
            //     subject: "FAILED: Pipeline '${JOB_NAME}' #${BUILD_NUMBER}",
            //     body: "Пайплайн завершился с ошибкой.",
            //     to: "your-email@example.com"
            // )
        }
    }
}
