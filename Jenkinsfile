pipeline {
    agent any

    environment {
        // Задаем рабочие папки через переменные
        DL_DIR = "tmp/downloads"
        LOG_FILE = "repos/web_log.log"
        GITHUB_LIST = "repos/github.txt"
    }

    stages {
        stage('Setup') {
            steps {
                // Создаем папки и пустой лог, если его нет
                sh "mkdir -p ${DL_DIR}"
                sh "touch ${LOG_FILE}"
            }
        }

        stage('Checking GitHub') {
            steps {
                echo "Starting GitHub checks..."
                script {
                    // Читаем файл строка за строкой
                    def lines = readFile(env.GITHUB_LIST).readLines()

                    lines.each { line ->
                        // Пропускаем комментарии и пустые строки
                        if (line.startsWith('#') || !line.trim()) return

                        // 1. Парсим строку: Ссылка;Regex
                        def parts = line.split(';')
                        def url = parts[0].trim()
                        def regex = parts[1].trim()
                        
                        // Получаем чистое имя репо (user/repo)
                        def repo = url.replace("https://github.com/", "").replace(/\/$/, "")

                        echo "Checking: ${repo}"

                        // 2. Получаем последнюю версию (tag_name) через API
                        // Используем jq для надежности. Если jq нет, вернемся на grep.
                        def latestVersion = sh(script: "curl -s https://api.github.com/repos/${repo}/releases/latest | jq -r .tag_name", returnStdout: true).trim()

                        // Защита от сбоя API (если вернулось null или пустота)
                        if (latestVersion == "null" || latestVersion == "") {
                            echo "Error checking ${repo}. Skipping."
                            return // переходим к следующему
                        }

                        // 3. Читаем старую версию из лога
                        // Ищем строку, начинающуюся с имени репо, и берем поле после |
                        def currentVersion = sh(script: "grep '^${repo}|' ${env.LOG_FILE} | cut -d '|' -f 2 || echo '0'", returnStdout: true).trim()

                        // 4. Сравнение
                        if (latestVersion != currentVersion) {
                            echo "UPDATE FOUND for ${repo}: ${currentVersion} -> ${latestVersion}"
                            
                            // --- БЛОК СКАЧИВАНИЯ ---
                            // Тут мы используем твой Regex для поиска ссылки на скачивание
                            def downloadUrl = sh(script: """
                                curl -s https://api.github.com/repos/${repo}/releases/latest | \
                                jq -r '.assets[] | select(.name | test("${regex}"; "i")) | .browser_download_url' | head -n 1
                            """, returnStdout: true).trim()

                            if (downloadUrl && downloadUrl != "null") {
                                echo "Downloading: ${downloadUrl}"
                                // Скачиваем файл в папку
                                sh "wget -qP ${env.DL_DIR} ${downloadUrl}"
                                
                                // Обновляем лог: удаляем старую запись и пишем новую
                                // Формат: Repo|Version|Date
                                sh "sed -i '/^${repo}|/d' ${env.LOG_FILE}"
                                sh "echo '${repo}|${latestVersion}|$(date +%Y-%m-%d)' >> ${env.LOG_FILE}"
                            } else {
                                echo "Version changed, but file matching regex '${regex}' not found!"
                            }
                            // -----------------------
                        } else {
                            echo "No updates for ${repo} (${currentVersion})"
                        }
                    }
                }
            }
        }
    }
}
