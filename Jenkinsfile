pipeline {
    agent any

    environment {
        DL_DIR = "tmp/downloads"
        LOG_FILE = "repos/web_log.log"
        GITHUB_LIST = "repos/github.txt"
        WEB_LIST = "repos/web.txt"
        // UA без пробелов и скобок
        UA = "Mozilla/5.0_Windows_NT_10.0_Win64_x64"
    }

    stages {
        stage('Cleanup') {
            steps {
                script {
                    echo "--- [START] Cleaning workspace ---"
                    sh "mkdir -p ${env.DL_DIR}"
                    // Чистим только временные файлы
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
                    // Читаем лог как список строк
                    def logLines = readFile(env.LOG_FILE).readLines()

                    lines.each { line ->
                        if (line.startsWith('#') || !line.trim()) return

                        def parts = line.split(';')
                        def url = parts[0].trim()
                        def regex = parts[1].trim()
                        def repo = url.replace("https://github.com/", "").replace(/\/$/, "")

                        echo ">>> Processing: ${repo}"

                        // Try-catch чтобы не падать из-за одного репо
                        try {
                            // 1. Качаем JSON
                            sh "curl -s -H User-Agent:${env.UA} https://api.github.com/repos/${repo}/releases/latest -o tmp/gh_response.json"
                            
                            def jsonCheck = readFile("tmp/gh_response.json").trim()
                            if (jsonCheck.contains("rate limit exceeded")) error "GITHUB LIMIT EXCEEDED"
                            
                            def latestVersion = sh(script: "jq -r .tag_name tmp/gh_response.json", returnStdout: true).trim()

                            if (!latestVersion || latestVersion == "null") error "Failed to get version"

                            // 2. ПРОВЕРКА ПО ЛОГУ (ИСПРАВЛЕНО)
                            // Ищем строку, содержащую "repo|latestVersion|"
                            def isUpToDate = logLines.any { it.contains("${repo}|${latestVersion}|") }

                            if (isUpToDate) {
                                echo "   [SKIP] Up to date: ${latestVersion}"
                                return // Пропускаем, не качаем
                            }

                            echo "   [UPDATE] New version found: ${latestVersion}"

                            // 3. Скачивание
                            def downloadUrls = sh(script: """
                                jq -r --arg REGEX "${regex}" '.assets[] | select(.name | test(\$REGEX; "i")) | .browser_download_url' tmp/gh_response.json
                            """, returnStdout: true).trim()

                            if (downloadUrls && downloadUrls != "null") {
                                def urls = downloadUrls.split('\n')
                                urls.each { dUrl ->
                                    def fileName = dUrl.split('/').last()
                                    echo "      + Downloading: ${fileName}"
                                    sh "wget -qP ${env.DL_DIR} ${dUrl}"
                                    
                                    if (fileName.endsWith(".zip")) {
                                        def folderName = fileName.replace('.zip', '')
                                        sh "mkdir -p ${env.DL_DIR}/${folderName}"
                                        sh "unzip -o ${env.DL_DIR}/${fileName} -d ${env.DL_DIR}/${folderName}/"
                                        sh "rm ${env.DL_DIR}/${fileName}"
                                    }
                                }

                                // 4. ОБНОВЛЕНИЕ ЛОГА
                                // Удаляем старые записи этого репо из памяти
                                logLines.removeAll { it.startsWith("${repo}|") }
                                // Добавляем новую
                                def dateNow = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
                                logLines.add("${repo}|${latestVersion}|${dateNow}")
                                // Перезаписываем файл целиком
                                writeFile file: env.LOG_FILE, text: logLines.join("\n") + "\n"

                            } else {
                                error "Regex mismatch: '${regex}'"
                            }

                        } catch (Exception e) {
                            echo "   [ERROR] ${repo}: ${e.getMessage()}"
                            currentBuild.result = 'UNSTABLE'
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
                    // Перечитываем лог (GitHub мог его обновить)
                    def logLines = readFile(env.LOG_FILE).readLines()
                    
                    lines.each { line ->
                        if (line.startsWith('#') || !line.trim()) return
                        
                        try {
                            def urls = [] 
                            def mode = "HASH"

                            // --- ПАРСЕРЫ ---
                            if (line.contains("videolan.org")) {
                                echo ">>> Checking VLC..."
                                def v64 = sh(script: "curl -s https://download.videolan.org/pub/videolan/vlc/last/win64/ | grep -oP 'href=\"\\Kvlc-[0-9.]+-win64\\.exe' | head -n 1", returnStdout: true).trim()
                                if (v64) urls.add("https://download.videolan.org/pub/videolan/vlc/last/win64/${v64}")
                                def v32 = sh(script: "curl -s https://download.videolan.org/pub/videolan/vlc/last/win32/ | grep -oP 'href=\"\\Kvlc-[0-9.]+-win32\\.exe' | head -n 1", returnStdout: true).trim()
                                if (v32) urls.add("https://download.videolan.org/pub/videolan/vlc/last/win32/${v32}")
                                mode = "SMART"
                            } 
                            else if (line.contains("telegram.org")) {
                                echo ">>> Checking Telegram..."
                                def tg64 = sh(script: "curl -s -o /dev/null -w '%{redirect_url}' 'https://telegram.org/dl/desktop/win64'", returnStdout: true).trim()
                                if (tg64) urls.add(tg64)
                                def tg32 = sh(script: "curl -s -o /dev/null -w '%{redirect_url}' 'https://telegram.org/dl/desktop/win'", returnStdout: true).trim()
                                if (tg32) urls.add(tg32)
                                mode = "SMART"
                            }
                            else if (line.contains("epicgames.com")) {
                                echo ">>> Checking Epic..."
                                def rawUrl = sh(script: "curl -s -A ${env.UA} -o /dev/null -w '%{redirect_url}' '${line}'", returnStdout: true).trim()
                                if (rawUrl) { urls.add(rawUrl.split('\\?')[0]); mode = "SMART" }
                            }
                            else if (line.contains("mozilla.org")) {
                                echo ">>> Checking Firefox..."
                                def rawUrl = sh(script: "curl -s -o /dev/null -w '%{redirect_url}' 'https://download.mozilla.org/?product=firefox-latest-ssl&os=win64&lang=ru'", returnStdout: true).trim()
                                if (rawUrl) {
                                    def oldName = rawUrl.split('/').last()
                                    def newName = sh(script: "echo '${oldName}' | sed 's/%20/./g'", returnStdout: true).trim()
                                    urls.add("${rawUrl}?name=${newName}")
                                    mode = "SMART"
                                }
                            }
                            else if (line.contains("fraps.com")) {
                                echo ">>> Checking Fraps..."
                                def ver = sh(script: "curl -s https://fraps.com/download.php | grep -oP 'Fraps \\K[0-9.]+' | head -n 1", returnStdout: true).trim()
                                if (ver) { urls.add("https://beepa.com/free/setup.exe?name=fraps-${ver}-setup.exe"); mode = "SMART" }
                            }
                            else if (line.contains("qbittorrent.org") || line.contains("hwinfo.com")) {
                                echo ">>> Checking SourceForge..."
                                def rssLink = line.contains("qbittorrent") ? "https://sourceforge.net/projects/qbittorrent/rss?path=/qbittorrent-win32" : "https://sourceforge.net/projects/hwinfo/rss"
                                def webUrl = sh(script: "curl -s '${rssLink}' | grep -o 'https://[^\"<]*\\(x64_setup\\|hwi64_[0-9]\\+\\)\\.exe/download' | head -n 1", returnStdout: true).trim()
                                if (webUrl) {
                                    sh "curl -L -s -A ${env.UA} -o 'tmp/sf_temp.html' '${webUrl}'"
                                    def directUrl = sh(script: "grep -oP 'https://downloads\\.sourceforge\\.net/[^\"]+' tmp/sf_temp.html | head -n 1", returnStdout: true).trim()
                                    sh "rm -f tmp/sf_temp.html"
                                    if (directUrl) {
                                        def fName = webUrl.replace("/download", "").split('/').last()
                                        if (line.contains("hwinfo")) fName = fName.replace(".exe", "").replace(".", "") + ".exe"
                                        urls.add("${directUrl}?name=${fName}")
                                        mode = "SMART"
                                    }
                                }
                            }
                            else {
                                urls.add(line.trim())
                                mode = "HASH"
                            }

                            // --- СКАЧИВАНИЕ ---
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

                                if (fname == "client" || fname == "download" || fname == "") return

                                // >>> SMART CHECK (ИСПРАВЛЕНО) <<<
                                if (mode == "SMART") {
                                    // Ищем в списке строк ЛЮБОЕ вхождение "|имя_файла|"
                                    def exists = logLines.any { it.contains("|${fname}|") }
                                    if (exists) {
                                        echo "   [SKIP] Already exists: ${fname}"
                                        return
                                    }
                                }

                                def headers = "-A ${env.UA} -L"
                                if (cleanUrl.contains("techpowerup.com")) headers += " -e 'https://www.techpowerup.com/'"

                                if (mode == "SMART") {
                                    echo "   [DOWN] Downloading ${fname}..."
                                    sh "curl -s ${headers} -o '${env.DL_DIR}/${fname}' '${cleanUrl}'"
                                    
                                    // Обновляем лог
                                    logLines.removeAll { it.contains("|${fname}|") }
                                    def dateNow = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
                                    logLines.add("WEB-SMART|${fname}|${dateNow}")
                                    
                                } else {
                                    // HASH CHECK
                                    echo "   [CHECK] Verifying hash for ${fname}..."
                                    sh "curl -s ${headers} -o '${env.DL_DIR}/${fname}.new' '${cleanUrl}'"
                                    
                                    def newHash = sh(script: "sha256sum '${env.DL_DIR}/${fname}.new' | awk '{print \$1}'", returnStdout: true).trim()
                                    
                                    // Проверка хеша (ИСПРАВЛЕНО)
                                    // Ищем строку, содержащую "|имя|хеш|"
                                    def hashMatch = logLines.any { it.contains("|${fname}|${newHash}|") }
                                    
                                    if (hashMatch) {
                                        echo "   [SKIP] Hash match: ${fname}"
                                        sh "rm '${env.DL_DIR}/${fname}.new'"
                                    } else {
                                        echo "   [UPDATE] Hash mismatch for ${fname}"
                                        sh "mv '${env.DL_DIR}/${fname}.new' '${env.DL_DIR}/${fname}'"
                                        
                                        logLines.removeAll { it.contains("|${fname}|") }
                                        def dateNow = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
                                        logLines.add("WEB-HASH|${fname}|${newHash}|${dateNow}")
                                    }
                                }
                                
                                // Сохраняем лог
                                writeFile file: env.LOG_FILE, text: logLines.join("\n") + "\n"
                            }

                        } catch (Exception e) {
                            echo "   [ERROR] Line failed: ${line}. ${e.getMessage()}"
                            currentBuild.result = 'UNSTABLE'
                        }
                    }
                }
            }
        }
    }
}
