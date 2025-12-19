pipeline {
    agent any
    environment {
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
                            def downloadUrls = sh(script: """
                                curl -s https://api.github.com/repos/${repo}/releases/latest | \
                                jq -r --arg REGEX "${regex}" '.assets[] | select(.name | test(\$REGEX; "i")) | .browser_download_url'
                            """, returnStdout: true).trim()

                            if (downloadUrls && downloadUrls != "null") {
                                def urls = downloadUrls.split('\n')
                                
				urls.each { dUrl ->
                                    echo "   + Downloading: ${dUrl}"
                                    sh "wget -qP ${env.DL_DIR} ${dUrl}"
                                    
                                    if (dUrl.endsWith(".zip")) {
                                        echo "   [UNZIP] Extracting ${dUrl.split('/').last()}..."
                                        sh "unzip -o ${env.DL_DIR}/${dUrl.split('/').last()} -d ${env.DL_DIR} && rm ${env.DL_DIR}/${dUrl.split('/').last()}"
                                    }
                                }                                
				sh "sed -i '\\#^${repo}|#d' ${env.LOG_FILE}"
				def dateNow = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
				sh "echo '${repo}|${latestVersion}|${dateNow}' >> ${env.LOG_FILE}"                                
                            } else {
                                echo "   ! Version changed, but NO files matched regex: ${regex}"
                            	exit 1
				}
                        } else {
                            echo "No updates for ${repo}"
                        }
                    }
                }
            }
        }


	stage('Checking Web') {
            steps {
                echo "Starting Web checks..."
                script {
                    // Проверяем наличие файла
                    if (!fileExists("repos/web.txt")) {
                        error "File repos/web.txt not found!"
                    }

                    def lines = readFile("repos/web.txt").readLines()
                    
                    lines.each { line ->
                        // Пропускаем пустые строки и комментарии
                        if (line.startsWith('#') || !line.trim()) return
                        
                        def urls = [] 
                        def mode = "HASH" // По умолчанию HASH, если парсер не переопределит

                        // --- ПАРСЕРЫ ---

                        if (line.contains("videolan.org")) {
                            echo ">>> Parsing VLC..."
                            def v64 = sh(script: "curl -s https://download.videolan.org/pub/videolan/vlc/last/win64/ | grep -oP 'vlc-.*-win64\\.exe' | head -n 1", returnStdout: true)
                            def v32 = sh(script: "curl -s https://download.videolan.org/pub/videolan/vlc/last/win32/ | grep -oP 'vlc-.*-win32\\.exe' | head -n 1", returnStdout: true)
                            
                            // .trim() вызываем только если результат не null
                            if (v64) urls.add("https://download.videolan.org/pub/videolan/vlc/last/win64/${v64.trim()}")
                            if (v32) urls.add("https://download.videolan.org/pub/videolan/vlc/last/win32/${v32.trim()}")
                            mode = "SMART"
                        } 
                        else if (line.contains("telegram.org")) {
                            echo ">>> Parsing Telegram..."
                            def tg = sh(script: "curl -s -I 'https://telegram.org/dl/desktop/win64' | grep -i 'location:' | awk '{print \$2}' | tr -d '\r'", returnStdout: true)
                            if (tg) urls.add(tg.trim())
                            mode = "SMART"
                        }
                        else if (line.contains("techpowerup.com")) {
                            echo ">>> Parsing TechPowerUp..."
                            // Обязательно нужен User-Agent, иначе вернут 403
                            def tpuID = sh(script: "curl -s -A 'Mozilla/5.0' '${line}' | grep -oP 'href=\"/download/start/\\?id=\\d+' | head -n 1 | cut -d'\"' -f2", returnStdout: true)
                            if (tpuID) {
                                urls.add("https://www.techpowerup.com${tpuID.trim()}")
                                mode = "HASH" // У TPU имя файла статичное при скачивании
                            }
                        }
                        else if (line.contains("mozilla.org")) {
                            echo ">>> Parsing Firefox..."
                            def ff = sh(script: "curl -s -I 'https://download.mozilla.org/?product=firefox-latest-ssl&os=win64&lang=ru' | grep -i 'location:' | awk '{print \$2}' | tr -d '\r'", returnStdout: true)
                            if (ff) urls.add(ff.trim())
                            mode = "SMART"
                        }
                        else if (line.contains("openvpn.net")) {
                            echo ">>> Parsing OpenVPN..."
                            def vpn = sh(script: "curl -s 'https://openvpn.net/client/' | grep -oP 'https://swupdate.openvpn.net/downloads/connect/openvpn-connect-[^\"]*-x64\\.msi' | head -n 1", returnStdout: true)
                            if (vpn) urls.add(vpn.trim())
                            mode = "SMART"
                        }
                        else if (line.contains("fraps.com")) {
                             echo ">>> Parsing Fraps..."
                             def frapsVer = sh(script: "curl -s http://www.fraps.com/download.php | grep -oP 'Fraps \\K[0-9.]+' | head -n 1", returnStdout: true)
                             if (frapsVer) {
                                 // Качаем и сразу даем правильное имя с версией
                                 def fVer = frapsVer.trim()
                                 // Добавляем фейковый URL с параметром, чтобы передать имя файла в цикл загрузки
                                 urls.add("https://www.fraps.com/free/setup.exe?name=FRAPS-${fVer}-setup.exe")
                                 mode = "SMART"
                             }
                        }
                        else {
                            // Прямые ссылки (Steam, Revo и т.д.)
                            urls.add(line.trim())
                            mode = "HASH"
                        }

                        // --- ЗАГРУЗКА ---
                        urls.each { dUrl ->
                            if (!dUrl) return
                            
                            // Определяем имя файла
                            def fname = ""
                            def cleanUrl = ""
                            
                            if (dUrl.contains("?name=")) {
                                // Если мы передали кастомное имя (как для Fraps)
                                def parts = dUrl.split("\\?name=")
                                cleanUrl = parts[0]
                                fname = parts[1]
                            } else {
                                // Обычный режим
                                cleanUrl = dUrl
                                fname = dUrl.split('/').last().split('\\?')[0]
                            }
                            
                            if (mode == "SMART") {
                                if (fileExists("tmp/downloads/${fname}")) {
                                    echo "   [SKIP] ${fname} already exists."
                                } else {
                                    echo "   [DOWN] Downloading ${fname}..."
                                    sh "curl -L -s -o 'tmp/downloads/${fname}' '${cleanUrl}'"
                                    
                                    // Запись в лог
                                    def dateNow = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
                                    sh "sed -i '\\#|${fname}|#d' repos/web_log.log"
                                    sh "echo 'WEB-SMART|${fname}|${dateNow}' >> repos/web_log.log"
                                }
                            } else {
                                // HASH MODE (Steam, TechPowerUp)
                                echo "   [HASH CHECK] ${fname}..."
                                // Используем Referer для TPU, иначе не скачает
                                sh "curl -L -s -A 'Mozilla/5.0' -e 'https://www.techpowerup.com/' -o 'tmp/downloads/${fname}.new' '${cleanUrl}'"
                                
                                // Проверка, скачался ли файл
                                def fileSize = sh(script: "wc -c < 'tmp/downloads/${fname}.new'", returnStdout: true).trim()
                                if (fileSize == "0") {
                                    echo "   [ERROR] Download failed (empty file) for ${fname}"
                                    sh "rm 'tmp/downloads/${fname}.new'"
                                    return
                                }

                                def newHash = sh(script: "sha256sum 'tmp/downloads/${fname}.new' | awk '{print \$1}'", returnStdout: true).trim()
                                def oldHash = sh(script: "grep '|${fname}|' repos/web_log.log | tail -n 1 | cut -d '|' -f 3 || echo 'none'", returnStdout: true).trim()
                                
                                if (newHash != oldHash) {
                                    echo "   [UPDATE] Hash mismatch! Updating ${fname}"
                                    sh "mv 'tmp/downloads/${fname}.new' 'tmp/downloads/${fname}'"
                                    
                                    def dateNow = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
                                    sh "sed -i '\\#|${fname}|#d' repos/web_log.log"
                                    sh "echo 'WEB-HASH|${fname}|${newHash}|${dateNow}' >> repos/web_log.log"
                                } else {
                                    echo "   [SKIP] Hash matches for ${fname}"
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
