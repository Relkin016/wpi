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

                        if (!latestVersion || latestVersion == "null") {
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
                                    def fileName = dUrl.split('/').last()
                                    echo "   + Downloading: ${fileName}"
                                    sh "wget -qP ${env.DL_DIR} ${dUrl}"
                                    
                                    // ЛОГИКА РАСПАКОВКИ В ПАПКУ
                                    if (fileName.endsWith(".zip")) {
                                        // Имя папки = имя файла без .zip
                                        def folderName = fileName.replace('.zip', '')
                                        echo "   [UNZIP] Extracting into ${folderName}..."
                                        sh "mkdir -p ${env.DL_DIR}/${folderName}"
                                        sh "unzip -o ${env.DL_DIR}/${fileName} -d ${env.DL_DIR}/${folderName}/"
                                        sh "rm ${env.DL_DIR}/${fileName}"
                                    }
                                }
                                
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

	stage('Checking Web') {
            steps {
                echo "Starting Web checks..."
                script {
                    if (!fileExists("repos/web.txt")) error "File repos/web.txt not found!"

                    def lines = readFile("repos/web.txt").readLines()
                    
                    lines.each { line ->
                        if (line.startsWith('#') || !line.trim()) return
                        
                        def urls = [] 
                        def mode = "HASH"

                        // --- ПАРСЕРЫ ---

                        if (line.contains("videolan.org")) {
                            echo ">>> Parsing VLC..."
                            // ИСПРАВЛЕНО: Регулярка теперь берет строго цифры и точки, без лишнего мусора
                            // \K отбрасывает href="
                            def v64 = sh(script: "curl -s https://download.videolan.org/pub/videolan/vlc/last/win64/ | grep -oP 'href=\"\\Kvlc-[0-9.]+-win64\\.exe' | head -n 1", returnStdout: true)
                            def v32 = sh(script: "curl -s https://download.videolan.org/pub/videolan/vlc/last/win32/ | grep -oP 'href=\"\\Kvlc-[0-9.]+-win32\\.exe' | head -n 1", returnStdout: true)
                            
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
                        else if (line.contains("hwinfo.com")) {
                            echo ">>> Parsing HWInfo (RSS)..."
                            // Берем версию из RSS
                            def ver = sh(script: "curl -s 'https://sourceforge.net/projects/hwinfo/rss' | grep -o 'hwi64_[0-9]\\+' | head -n 1 | cut -d'_' -f2", returnStdout: true)
                            if (ver) {
                                urls.add("https://www.hwinfo.com/files/hwi64_${ver.trim()}.exe")
                                mode = "SMART"
                            }
                        }
                        else if (line.contains("qbittorrent.org")) {
                            echo ">>> Parsing QBittorrent (RSS)..."
                            def qbit = sh(script: "curl -s 'https://sourceforge.net/projects/qbittorrent/rss?path=/qbittorrent-win32' | grep -o 'https://.*_x64_setup.exe/download' | head -n 1", returnStdout: true)
                            if (qbit) {
                                urls.add(qbit.trim())
                                mode = "SMART"
                            }
                        }
                        else if (line.contains("nvcleanstall")) {
                            echo ">>> Parsing NVCleanstall..."
                            // 1. Ищем строку вида "NVCleanstall v1.19.0" в заголовке
                            def ver = sh(script: "curl -s -A 'Mozilla/5.0' '${line}' | grep -oP 'NVCleanstall v\\K[0-9.]+' | head -n 1", returnStdout: true)
                            
                            if (ver) {
                                def cleanVer = ver.trim()
                                def targetName = "NVCleanstall_${cleanVer}.exe"
                                // 2. Формируем "виртуальный" URL с именем файла, чтобы передать в загрузчик
                                // Нам все равно нужно выцеплять ID для скачивания, но проверять будем по ИМЕНИ
                                def tpuID = sh(script: "curl -s -A 'Mozilla/5.0' '${line}' | grep -oP 'href=\"/download/start/\\?id=\\d+' | head -n 1 | cut -d'\"' -f2", returnStdout: true)
                                if (tpuID) {
                                    urls.add("https://www.techpowerup.com${tpuID.trim()}?name=${targetName}")
                                    mode = "SMART"
                                }
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
                            // Ищем ссылку, игнорируем регистр, ищем расширение msi
                            def vpn = sh(script: "curl -s -A 'Mozilla/5.0' 'https://openvpn.net/client/' | grep -oP 'https://swupdate.openvpn.net/downloads/connect/openvpn-connect-[0-9.]+_signed-x64\\.msi' | head -n 1", returnStdout: true)
                            if (vpn) urls.add(vpn.trim())
                            mode = "SMART"
                        }
                        else if (line.contains("fraps.com")) {
                             echo ">>> Parsing Fraps..."
                             def frapsVer = sh(script: "curl -s http://www.fraps.com/download.php | grep -oP 'Fraps \\K[0-9.]+' | head -n 1", returnStdout: true)
                             if (frapsVer) {
                                 urls.add("https://www.fraps.com/free/setup.exe?name=FRAPS-${frapsVer.trim()}-setup.exe")
                                 mode = "SMART"
                             }
                        }
                        else {
                            // Прямые ссылки (Steam, Revo) - тут хеш неизбежен, версии нет в URL
                            urls.add(line.trim())
                            mode = "HASH"
                        }

                        // --- ЗАГРУЗКА ---
                        urls.each { dUrl ->
                            if (!dUrl) return
                            
                            def fname = ""
                            def cleanUrl = ""
                            
                            // Обработка параметра ?name= (для Fraps и NVCleanstall)
                            if (dUrl.contains("?name=")) {
                                def parts = dUrl.split("\\?name=")
                                cleanUrl = parts[0]
                                fname = parts[1]
                            } else {
                                cleanUrl = dUrl
                                fname = dUrl.split('/').last().split('\\?')[0]
                            }
                            
                            if (mode == "SMART") {
                                if (fileExists("${env.DL_DIR}/${fname}")) {
                                    echo "   [SKIP] ${fname} already exists."
                                } else {
                                    echo "   [DOWN] Downloading ${fname}..."
                                    // Добавляем Referer для TechPowerUp, остальным он не мешает
                                    sh "curl -L -s -A 'Mozilla/5.0' -e 'https://www.techpowerup.com/' -o '${env.DL_DIR}/${fname}' '${cleanUrl}'"
                                    
                                    // Проверка что файл не пустой (TPU любит отдавать 0 байт если что-то не так)
                                    def fSize = sh(script: "wc -c < '${env.DL_DIR}/${fname}'", returnStdout: true).trim()
                                    if (fSize == "0") {
                                        echo "   [ERROR] Download failed (0 bytes) for ${fname}"
                                        sh "rm '${env.DL_DIR}/${fname}'"
                                    } else {
                                        def dateNow = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
                                        sh "sed -i '\\#|${fname}|#d' repos/web_log.log"
                                        sh "echo 'WEB-SMART|${fname}|${dateNow}' >> repos/web_log.log"
                                    }
                                }
                            } else {
                                // HASH MODE (Только для Steam/Revo где нет выбора)
                                echo "   [HASH CHECK] ${fname}..."
                                sh "curl -L -s -A 'Mozilla/5.0' -o '${env.DL_DIR}/${fname}.new' '${cleanUrl}'"
                                
                                def newHash = sh(script: "sha256sum '${env.DL_DIR}/${fname}.new' | awk '{print \$1}'", returnStdout: true).trim()
                                def oldHash = sh(script: "grep '|${fname}|' repos/web_log.log | tail -n 1 | cut -d '|' -f 3 || echo 'none'", returnStdout: true).trim()
                                
                                if (newHash != oldHash) {
                                    echo "   [UPDATE] Hash mismatch! Updating ${fname}"
                                    sh "mv '${env.DL_DIR}/${fname}.new' '${env.DL_DIR}/${fname}'"
                                    
                                    def dateNow = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
                                    sh "sed -i '\\#|${fname}|#d' repos/web_log.log"
                                    sh "echo 'WEB-HASH|${fname}|${newHash}|${dateNow}' >> repos/web_log.log"
                                } else {
                                    sh "rm '${env.DL_DIR}/${fname}.new'"
                                }
                            }
                        }
                    }
                }
            }
        }



    }
}
