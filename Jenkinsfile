// Compile ISO: genisoimage -o имя_образа.iso -R -J /путь/к/исходной/папке
// GetInfoFromGitHub
// GetInfoFromWeb


pipeline {
    // 1. Указываем, где будет выполняться код
    agent any

    // 2. Определяем переменные и настройки
    environment {
        // Устанавливаем рабочую директорию, если нужно
        BUILD_DIR = 'target'
    }

    // 3. Стадии выполнения
    stages {
        stage('Checkout') {
            steps {
                // Jenkins автоматически клонирует репозиторий, 
                // если задание настроено как 'Pipeline script from SCM'
                echo "Код склонирован из GitHub"
            }
        }

        stage('Build') {
            steps {
                // Пример: Выполнение команды сборки проекта
                sh 'npm install'
                sh 'npm run build'
            }
            // Отправка уведомления на GitHub
            post {
                success {
                    // Используется для установки статуса коммита в GitHub
                    // Requires the GitHub plugin
                    githubNotify status: 'SUCCESS'
                }
            }
        }

        stage('Test') {
            steps {
                
            }
        }
    }

        stage('Deploy') {
            steps {
                // Copy to Drive
                // Get drive link
                // Merge to site - update index.html
            
                
            }
        }
    }

    // 4. Пост-действия (выполняются всегда или при определенном статусе)
    post {
        always {
            echo 'Очистка рабочей директории...'
            // cleanWs() - удаляет рабочую директорию после выполнения
        }
    }
}

