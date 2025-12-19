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
                        // Получаем чистое имя репо
                        def repo = url.replace("https://github.com/", "").replace(/\/$/, "")
                        echo ">>> Checking: ${repo}"
                        def latestVersion = sh(script: "curl -s https://api.github.com/repos/${repo}/releases/latest | jq -r .tag_name", returnStdout: true).trim()
                        if (latestVersion == "null" || latestVersion == "") {
                            echo "Error checking ${repo}. Skipping."
                            return 
                        }

                        def currentVersion = sh(script: "grep '^${repo}|' ${env.LOG_FILE} | cut -d '|' -f 2 || echo '0'", returnStdout: true).trim()
                        if (latestVersion != currentVersion) {
                            echo "UPDATE FOUND for ${repo}: ${currentVersion} -> ${latestVersion}"
                            // Получаем СПИСОК ссылок (убрали head -n 1)
                            def downloadUrls = sh(script: """
                                curl -s https://api.github.com/repos/${repo}/releases/latest | \
                                jq -r --arg REGEX "${regex}" '.assets[] | select(.name | test(\$REGEX; "i")) | .browser_download_url'
                            """, returnStdout: true).trim()

                            if (downloadUrls && downloadUrls != "null") {
                                // Разбиваем список по строкам
                                def urls = downloadUrls.split('\n')
                                
                                urls.each { dUrl ->
                                    echo "   + Downloading: ${dUrl}"
                                    // Качаем каждый файл
                                    sh "wget -qP ${env.DL_DIR} ${dUrl}"
                                }
                                
                                // Обновляем лог только 1 раз после скачивания всех файлов
				sh "sed -i '\\#^${repo}|#d' ${env.LOG_FILE}"
				def dateNow = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
				sh "echo '${repo}|${latestVersion}|${dateNow}' >> ${env.LOG_FILE}"                                
                            } else {
                                echo "   ! Version changed, but NO files matched regex: ${regex}"
                            }
                        } else {
                            echo "No updates for ${repo}"
                        }
                    }
                }
            }
        }


	stage('Checking other sources') {
            steps {
                echo "Starting Web checks..."
                script {
                    // Используем полный путь из env или прямой путь, если env багует
                    def webFile = "repos/web.txt"
                    if (!fileExists(webFile)) {
                        error "File ${webFile} not found!"
                    }
                    
                    def lines = readFile(webFile).readLines()
                    
                    lines.each { line ->
                        if (!line.trim() || line.startsWith('#')) return
                        
                        def urls = []
                        def mode = "HASH"

                        if (line.contains("videolan.org")) {
                            echo ">>> VLC..."
                            def vlc64 = sh(script: "curl -s https://download.videolan.org/pub/videolan/vlc/last/win64/ | grep -oP 'vlc-.*-win64\\.exe' | head -n 1", returnStdout: true).trim()
                            def vlc32 = sh(script: "curl -s https://download.videolan.org/pub/videolan/vlc/last/win32/ | grep -oP 'vlc-.*-win32\\.exe' | head -n 1", returnStdout: true).trim
			    if (vlc64) urls.add("https://download.videolan.org/pub/videolan/vlc/last/win64/${vlc64}")
                            if (vlc32) urls.add("https://download.videolan.org/pub/videolan/vlc/last/win32/${vlc32}")
		            mode = "SMART"
                        } 
                        else if (line.contains("telegram.org")) {
                            echo ">>> Telegram..."
                            def tg64 = sh(script: "curl -s -I 'https://telegram.org/dl/desktop/win64' | grep -i 'location:' | awk '{print \$2}' | tr -d '\r'", returnStdout: true).trim()
                            def tg32 = sh(script: "curl -s -I 'https://telegram.org/dl/desktop/win' | grep -i 'location:' | awk '{print \$2}' | tr -d '\r'", returnStdout: true).trim()
			    if (tg64) urls.add(tg64)
		            if (tg32) urls.add(tg32)
                            mode = "SMART"
                        }
                        else if (line.contains("msi.com")) {
                            echo ">>> MSI Afterburner..."
                            def msiUrl = sh(script: "curl -s -A 'Mozilla/5.0' 'https://www.msi.com/Landing/afterburner/graphics-cards' | grep -o 'https://[^\"]*MSIAfterburnerSetup\\.zip[^\"]*' | head -n 1", returnStdout: true).trim()
                            if (msiUrl) urls.add(msiUrl)
                            mode = "HASH"
                        }
                        else if (line.contains("epicgames.com")) {
                        echo ">>> Epic Games..."
                            def epic = sh(script: "curl -s -I '${line.trim()}' | grep -i 'location:' | awk '{print \$2}' | tr -d '\r'", returnStdout: true).trim()
                            if (epic) urls.add(epic)
                            mode = "SMART"
                        }
			else if (line.contains("fraps.com")) {
                            echo ">>> Parsing Fraps..."
                            // Вытягиваем версию (например, 3.5.99) из текста страницы
                            def frapsVer = sh(script: "curl -s http://www.fraps.com/download.php | grep -oP 'Fraps \\K[0-9.]+' | head -n 1", returnStdout: true).trim()
                            if (frapsVer) {
                                // Fraps всегда отдает файл setup.exe, поэтому мы переименуем его для порядка
                                def frapsUrl = "https://www.fraps.com/free/setup.exe"
                                def customName = "FRAPS-${frapsVer}-setup.exe"
                                
                                if (fileExists("tmp/downloads/${customName}")) {
                                    echo "   [SKIP] Fraps ${frapsVer} already exists."
                                } else {
                                    echo "   [DOWN] Fraps new version: ${frapsVer}"
                                    sh "curl -L -s -o 'tmp/downloads/${customName}' '${frapsUrl}'"
                                    sh "echo 'WEB-SMART|${customName}|\\\$(date +%Y-%m-%d)' >> repos/web_log.log"
                                }
                            }
                        }
                        else {
                            urls.add(line.trim())
                            mode = "HASH"
                        }

                        urls.each { dUrl ->
                            if (!dUrl) return
                            def fname = dUrl.split('/').last().split('\\?')[0]
                            
                            if (mode == "SMART") {
                                if (fileExists("tmp/downloads/${fname}")) {
                                    echo "   [SKIP] ${fname}"
                                } else {
                                    echo "   [DOWN] ${fname}"
                                    sh "curl -L -s -o 'tmp/downloads/${fname}' '${dUrl}'"
				    sh "sed -i '\\#|${fname}|#d' ${env.LOG_FILE}"
				    def dateNow = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
				    sh "echo 'WEB-SMART|${fname}|${dateNow}' >> ${env.LOG_FILE}"
                                }
                            } else {
                                echo "   [HASH] ${fname}"
                                sh "curl -L -s -o 'tmp/downloads/${fname}.new' '${dUrl}'"
                                def newHash = sh(script: "sha256sum 'tmp/downloads/${fname}.new' | awk '{print \$1}'", returnStdout: true).trim()
                                def oldHash = sh(script: "grep '|${fname}|' repos/web_log.log | tail -n 1 | cut -d '|' -f 3 || echo 'none'", returnStdout: true).trim()
                                
                                if (newHash != oldHash) {
                                    sh "mv 'tmp/downloads/${fname}.new' 'tmp/downloads/${fname}'"
                                    sh "echo 'WEB-HASH|${fname}|${newHash}|\\\$(date +%Y-%m-%d)' >> repos/web_log.log"
                                } else {
                                    sh "rm 'tmp/downloads/${fname}.new'"
                                }
                            }
                        }
                    }
                }
            }
        }





    }
}
