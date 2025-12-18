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
                    def lines = readFile(env.GITHUB_LIST).readLines()

                    lines.each { line ->
                        if (line.startsWith('#') || !line.trim()) return

                        def parts = line.split(';')
                        def url = parts[0].trim()
                        def regex = parts[1].trim()
                        
                        // Чистим URL
                        def repo = url.replace("https://github.com/", "").replace(/\/$/, "")

                        echo "Checking: ${repo}"

                        // Получаем версию
                        def latestVersion = sh(script: "curl -s https://api.github.com/repos/${repo}/releases/latest | jq -r .tag_name", returnStdout: true).trim()

                        if (latestVersion == "null" || latestVersion == "") {
                            echo "Error checking ${repo}. Skipping."
                            return 
                        }

                        // Читаем старую версию
                        def currentVersion = sh(script: "grep '^${repo}|' ${env.LOG_FILE} | cut -d '|' -f 2 || echo '0'", returnStdout: true).trim()

                        if (latestVersion != currentVersion) {
                            echo "UPDATE FOUND for ${repo}: ${currentVersion} -> ${latestVersion}"
                            
                            // --- ФИКС ЗДЕСЬ ---
                            // 1. --arg REGEX "${regex}" передает строку внутрь jq безопасно
                            // 2. \$REGEX используется внутри фильтра (экранируем $ для Groovy)
                            def downloadUrl = sh(script: """
                                curl -s https://api.github.com/repos/${repo}/releases/latest | \
                                jq -r --arg REGEX "${regex}" '.assets[] | select(.name | test(\$REGEX; "i")) | .browser_download_url' | head -n 1
                            """, returnStdout: true).trim()

                            if (downloadUrl && downloadUrl != "null") {
                                echo "Downloading: ${downloadUrl}"
                                sh "wget -qP ${env.DL_DIR} ${downloadUrl}"
                                
                                // Обновляем лог
                                sh "sed -i '\\#^${repo}|#d' ${env.LOG_FILE}"
                                sh "echo '${repo}|${latestVersion}|\\\$(date +%Y-%m-%d)' >> ${env.LOG_FILE}"
                            } else {
                                echo "Version changed, but file matching regex not found!"
                            }
                        } else {
                            echo "No updates for ${repo}"
                        }
                    }
                }
            }
        }


    }
}
