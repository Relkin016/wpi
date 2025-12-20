pipeline {
    agent any

    environment {
        DL_DIR = "tmp/downloads"
        LOG_FILE = "repos/web_log.log"
        GITHUB_LIST = "repos/github.txt"
        WEB_LIST = "repos/web.txt"
        // БЕЗОПАСНЫЙ UA: Без пробелов и скобок. Работает везде, не ломает bash.
        UA = "Mozilla/5.0_Windows_NT_10.0_Win64_x64"
    }

    stages {
        stage('Cleanup') {
            steps {
                script {
                    echo "--- [START] Cleaning workspace ---"
                    sh "mkdir -p ${env.DL_DIR}"
                    // Удаляем временные файлы проверок, но оставляем готовые инсталляторы
                    sh "rm -f ${env.DL_DIR}/*.new"
                    sh "rm -f ${env.DL_DIR}/*.html"
                    sh "rm -f tmp/*.json"
                    
                    if (!fileExists(env.LOG_FILE)) {
                        sh "touch ${env.LOG_FILE}"
                    }
                }
            }
        }

        stage('Checking GitHub') {
            steps {
                echo "--- [GITHUB] Checking for updates ---"
                script {
                    if (!fileExists(env.GITHUB_LIST)) error "File ${env.GITHUB_LIST} not found!"
                    
                    def lines = readFile(env.GITHUB_LIST).readLines()
                    // Читаем лог один раз в память
                    def logContent = readFile(env.LOG_FILE)

                    lines.each { line ->
                        if (line.startsWith('#') || !line.trim()) return

                        def parts = line.split(';')
                        def url = parts[0].trim()
                        def regex = parts[1].trim()
                        def repo = url.replace("https://github.com/", "").replace(/\/$/, "")

                        echo ">>> Processing: ${repo}"

                        // 1. Качаем JSON в файл (это спасает от проблем с кавычками)
                        sh "curl -s -H User-Agent:${env.UA} https://api.github.com/repos/${repo}/releases/latest -o tmp/gh_response.json"
                        
                        def jsonContent = readFile("tmp/gh_response.json").trim()
                        if (jsonContent.contains("rate limit exceeded")) {
                            error "GITHUB API LIMIT EXCEEDED! Wait 1h."
                        }

                        // 2. Достаем версию через jq
                        def latestVersion = sh(script: "jq -r .tag_name tmp/gh_response.json", returnStdout: true).trim()

                        if (!latestVersion || latestVersion == "null") {
                            error "Failed to get version for ${repo}."
                        }

                        // 3. ПРОВЕРКА ПО ЛОГУ
                        // Если в логе есть строка "repo|версия|", значит файл уже скачан
                        if (logContent.contains("${repo}|${latestVersion}|")) {
                            echo "   [SKIP] Up to date: ${latestVersion}"
                            return // Пропускаем этот репозиторий
                        }

                        echo "   [UPDATE] New version found: ${latestVersion}"
                        
                        // 4. Парсим URL загрузки
                        def downloadUrls = sh(script: """
                            jq -r --arg REGEX "${regex}" '.assets[] | select(.name | test(\$REGEX; "i")) | .browser_download_url' tmp/gh_response.json
                        """, returnStdout: true).trim()

                        if (downloadUrls && downloadUrls != "null") {
                            def urls = downloadUrls.split('\n')
                            urls.each { dUrl ->
                                def fileName = dUrl.split('/').last()
                                echo "      + Downloading: ${fileName}"
                                sh "wget -qP ${env.DL_DIR} ${dUrl}"
                                
                                // Авто-распаковка ZIP
                                if (fileName.endsWith(".zip")) {
                                    def folderName = fileName.replace('.zip', '')
                                    sh "mkdir -p ${env.DL_DIR}/${folderName}"
                                    sh "unzip -o ${env.DL_DIR}/${fileName} -d ${env.DL_DIR}/${folderName}/"
                                    sh "rm ${env.DL_DIR}/${fileName}"
                                }
                            }
                            
                            // Обновляем лог: удаляем старую запись и добавляем новую
                            sh "sed -i '\\#^${repo}|#d' ${env.LOG_FILE}"
                            def dateNow = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
                            sh "echo '${repo}|${latestVersion}|${dateNow}' >> ${env.LOG_FILE}"
                            
                        } else {
                            error "CRITICAL: Version changed for ${repo}, but NO files matched regex: '${regex}'"
                        }
                    }
                }
            }
        }

        stage('Checking Web') {
            steps {
                echo "--- [WEB] Checking for updates ---"
                script {
                    if (!fileExists(env.WEB_LIST)) error "File ${env.WEB_LIST} not found!"

                    def lines = readFile(env.WEB_LIST).readLines()
                    def logContent = readFile(env.LOG_FILE) // Обновляем данные лога (там уже есть записи от GitHub)
                    
                    lines.each { line ->
                        if (line.startsWith('#') || !line.trim()) return
                        
                        def urls = [] 
                        def mode = "HASH"

                        // ===========================
                        //     PARSING LOGIC
                        // ===========================

                        // 1. VLC (x64 & x86)
                        if (line.contains("videolan.org")) {
                            echo ">>> Parsing VLC..."
                            def v64 = sh(script: "curl -s https://download.videolan.org/pub/videolan/vlc/last/win64/ | grep -oP 'href=\"\\Kvlc-[0-9.]+-win64\\.exe' | head -n 1", returnStdout: true).trim()
                            if (v64) urls.add("https://download.videolan.org/pub/videolan/vlc/last/win64/${v64}")
                            
                            def v32 = sh(script: "curl -s https://download.videolan.org/pub/videolan/vlc/last/win32/ | grep -oP 'href=\"\\Kvlc-[0-9.]+-win32\\.exe' | head -n 1", returnStdout: true).trim()
                            if (v32) urls.add("https://download.videolan.org/pub/videolan/vlc/last/win32/${v32}")
                            mode = "SMART"
                        } 
                        // 2. TELEGRAM (x64 & x86)
                        else if (line.contains("telegram.org")) {
                            echo ">>> Parsing Telegram..."
                            def tg64 = sh(script: "curl -s -o /dev/null -w '%{redirect_url}' 'https://telegram.org/dl/desktop/win64'", returnStdout: true).trim()
                            if (tg64) urls.add(tg64)
                            def tg32 = sh(script: "curl -s -o /dev/null -w '%{redirect_url}' 'https://telegram.org/dl/desktop/win'", returnStdout: true).trim()
                            if (tg32) urls.add(tg32)
                            mode = "SMART"
                        }
                        // 3. EPIC GAMES
                        else if (line.contains("epicgames.com")) {
                            echo ">>> Parsing Epic Games..."
                            // UA без кавычек (переменная безопасная)
                            def rawUrl = sh(script: "curl -s -A ${env.UA} -o /dev/null -w '%{redirect_url}' '${line}'", returnStdout: true).trim()
                            if (rawUrl) {
                                urls.add(rawUrl.split('\\?')[0])
                                mode = "SMART"
                            }
                        }
                        // 4. FIREFOX
                        else if (line.contains("mozilla.org")) {
                            echo ">>> Parsing Firefox..."
                            def rawUrl = sh(script: "curl -s -o /dev/null -w '%{redirect_url}' 'https://download.mozilla.org/?product=firefox-latest-ssl&os=win64&lang=ru'", returnStdout: true).trim()
                            if (rawUrl) {
                                def oldName = rawUrl.split('/').last()
                                def newName = sh(script: "echo '${oldName}' | sed 's/%20/./g'", returnStdout: true).trim()
                                urls.add("${rawUrl}?name=${newName}")
                                mode = "SMART"
                            }
                        }
                        // 5. FRAPS
                        else if (line.contains("fraps.com")) {
                             echo ">>> Parsing Fraps..."
                             def frapsVer = sh(script: "curl -s https://fraps.com/download.php | grep -oP 'Fraps \\K[0-9.]+' | head -n 1", returnStdout: true).trim()
                             if (frapsVer) {
                                 urls.add("https://beepa.com/free/setup.exe?name=fraps-${frapsVer}-setup.exe")
                                 mode = "SMART"
                             }
                        }
                        // 6. SOURCEFORGE (qBit + HWInfo)
                        else if (line.contains("qbittorrent.org") || line.contains("hwinfo.com")) {
                            echo ">>> Parsing SourceForge..."
                            def rssLink = ""
                            if (line.contains("qbittorrent")) rssLink = "https://sourceforge.net/projects/qbittorrent/rss?path=/qbittorrent-win32"
                            if (line.contains("hwinfo")) rssLink = "https://sourceforge.net/projects/hwinfo/rss"

                            def webUrl = sh(script: "curl -s '${rssLink}' | grep -o 'https://[^\"<]*\\(x64_setup\\|hwi64_[0-9]\\+\\)\\.exe/download' | head -n 1", returnStdout: true).trim()
                            
                            if (webUrl) {
                                sh "curl -L -s -A ${env.UA} -o 'tmp/sf_temp.html' '${webUrl}'"
                                def directUrl = sh(script: "grep -oP 'https://downloads\\.sourceforge\\.net/[^\"]+' tmp/sf_temp.html | head -n 1", returnStdout: true).trim()
                                sh "rm -f tmp/sf_temp.html"

                                if (directUrl) {
                                    def fName = webUrl.replace("/download", "").split('/').last()
                                    // HWInfo fix: 8.34.exe -> 834.exe (убираем лишние точки в версии)
                                    if (line.contains("hwinfo")) fName = fName.replace(".exe", "").replace(".", "") + ".exe"
                                    urls.add("${directUrl}?name=${fName}")
                                    mode = "SMART"
                                }
                            }
                        }
                        // 7. HASH CHECK (Steam, OpenVPN прямая ссылка)
                        else {
                            urls.add(line.trim())
                            mode = "HASH"
                        }

                        // ===========================
                        //     DOWNLOAD & VERIFY
                        // ===========================
                        urls.each { dUrl ->
                            if (!dUrl) return
                            
                            def fname = ""
                            def cleanUrl = ""
                            
                            if (dUrl.contains("?name=")) {
                                def parts = dUrl.split("\\?name=")
                                cleanUrl = parts[0]
                                fname = parts[1]
                            } else {
                                cleanUrl = dUrl
                                def rawName = cleanUrl.split('/').last().split('\\?')[0]
                                fname = java.net.URLDecoder.decode(rawName, "UTF-8")
                            }

                            // ЗАЩИТА: Пропускаем мусорные имена
                            if (fname == "client" || fname == "download" || fname == "") {
                                echo "   [SKIP] Garbage filename detected: '${fname}'"
                                return
                            }

                            // >>> SMART CHECK (Проверка по имени) <<<
                            if (mode == "SMART") {
                                // Если файл с таким именем уже есть в логе — значит версия актуальна
                                if (logContent.contains("|${fname}|")) {
                                    echo "   [SKIP] Already up to date: ${fname}"
                                    return
                                }
                            }

                            // Формируем заголовки
                            def headers = "-A ${env.UA} -L"
                            if (cleanUrl.contains("techpowerup.com")) headers += " -e 'https://www.techpowerup.com/'"

                            // >>> DOWNLOADING <<<
                            if (mode == "SMART") {
                                // Для SMART качаем сразу в чистовик, так как мы уже проверили лог выше
                                echo "   [DOWN] Downloading new version: ${fname}..."
                                sh "curl -s ${headers} -o '${env.DL_DIR}/${fname}' '${cleanUrl}'"
                                
                                def fSize = sh(script: "wc -c < '${env.DL_DIR}/${fname}'", returnStdout: true).trim()
                                if (fSize == "0" || fSize.toInteger() < 10000) {
                                    echo "   [FAIL] File too small. Deleting."
                                    sh "rm '${env.DL_DIR}/${fname}'"
                                } else {
                                    def dateNow = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
                                    // Удаляем старую запись (если была) и пишем новую
                                    sh "sed -i '\\#|${fname}|#d' ${env.LOG_FILE}"
                                    sh "echo 'WEB-SMART|${fname}|${dateNow}' >> ${env.LOG_FILE}"
                                }
                            } else {
                                // >>> HASH CHECK MODE <<< 
                                // (Steam, OpenVPN) - всегда качаем во временный файл для сверки хеша
                                echo "   [CHECK] Verifying hash for ${fname}..."
                                sh "curl -s ${headers} -o '${env.DL_DIR}/${fname}.new' '${cleanUrl}'"
                                
                                def fSize = sh(script: "wc -c < '${env.DL_DIR}/${fname}.new'", returnStdout: true).trim()
                                if (fSize == "0" || fSize.toInteger() < 10000) {
                                    sh "rm '${env.DL_DIR}/${fname}.new'"
                                } else {
                                    def newHash = sh(script: "sha256sum '${env.DL_DIR}/${fname}.new' | awk '{print \$1}'", returnStdout: true).trim()
                                    
                                    // Проверяем: есть ли в логе ЭТО имя с ЭТИМ хешем?
                                    if (logContent.contains("|${fname}|${newHash}|")) {
                                        echo "   [SKIP] Hash match (Up to date): ${fname}"
                                        sh "rm '${env.DL_DIR}/${fname}.new'" // Удаляем, версия та же
                                    } else {
                                        echo "   [UPDATE] Hash mismatch! Updating ${fname}..."
                                        sh "mv '${env.DL_DIR}/${fname}.new' '${env.DL_DIR}/${fname}'"
                                        def dateNow = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
                                        sh "sed -i '\\#|${fname}|#d' ${env.LOG_FILE}"
                                        sh "echo 'WEB-HASH|${fname}|${newHash}|${dateNow}' >> ${env.LOG_FILE}"
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
