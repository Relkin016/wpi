pipeline {
    agent any

    environment {
        DL_DIR = "tmp/downloads"
        // ВЕРНУЛ КАК ТЫ ПРОСИЛ: Файл внутри репо, чтобы потом комитить
        LOG_FILE = "repos/web_log.log"
        
        GITHUB_LIST = "repos/github.txt"
        WEB_LIST = "repos/web.txt"
        QUEUE_FILE = "tmp/download_queue.txt"
        
        UA = "Mozilla/5.0_Windows_NT_10.0_Win64_x64"
    }

    stages {
        // ==========================================
        // 1. ПОДГОТОВИТЕЛЬНЫЙ ЭТАП
        // ==========================================
        stage('Preparation') {
            steps {
                script {
                    echo "--- [PREP] Preparing workspace ---"
                    sh "mkdir -p ${env.DL_DIR}"
                    sh "mkdir -p tmp"
                    
                    // Чистим временные файлы
                    sh "rm -f ${env.DL_DIR}/*.new"
                    sh "rm -f ${env.DL_DIR}/*.html"
                    sh "rm -f tmp/*.json"
                    sh "rm -f ${env.QUEUE_FILE}"
                    
                    if (!fileExists(env.LOG_FILE)) {
                        echo "Log file not found in repo. Creating empty one."
                        sh "touch ${env.LOG_FILE}"
                    }
                }
            }
        }

        // ==========================================
        // 2. CHECKING VERSIONS (GATEKEEPER)
        // ==========================================
        stage('Check Versions') {
            steps {
                echo "--- [CHECK] Comparing remote versions with local log ---"
                script {
                    def logLines = readFile(env.LOG_FILE).readLines()
                    def updatesFound = false

                    // --- ЧАСТЬ А: ПРОВЕРКА GITHUB ---
                    if (fileExists(env.GITHUB_LIST)) {
                        def ghLines = readFile(env.GITHUB_LIST).readLines()
                        ghLines.each { line ->
                            if (line.startsWith('#') || !line.trim()) return
                            
                            try {
                                def parts = line.split(';')
                                def url = parts[0].trim()
                                def regex = parts[1].trim()
                                def repo = url.replace("https://github.com/", "").replace(/\/$/, "")

                                echo ">>> Checking GitHub: ${repo}"

                                sh "curl -s -H User-Agent:${env.UA} https://api.github.com/repos/${repo}/releases/latest -o tmp/gh_${repo.replace('/', '_')}.json"
                                def jsonFile = "tmp/gh_${repo.replace('/', '_')}.json"
                                
                                if (readFile(jsonFile).contains("rate limit exceeded")) error "GITHUB LIMIT EXCEEDED"
                                
                                def latestVer = sh(script: "jq -r .tag_name ${jsonFile}", returnStdout: true).trim()
                                if (!latestVer || latestVer == "null") error "Failed to parse version"

                                // Сравниваем с логом
                                def isFresh = logLines.any { it.contains("${repo}|${latestVer}|") }

                                if (!isFresh) {
                                    echo "   [UPDATE DETECTED] ${repo}: New version ${latestVer}"
                                    updatesFound = true
                                    
                                    // Парсим ссылку
                                    def dUrl = sh(script: "jq -r --arg REGEX \"${regex}\" '.assets[] | select(.name | test(\$REGEX; \"i\")) | .browser_download_url' ${jsonFile} | head -n 1", returnStdout: true).trim()
                                    
                                    if (dUrl && dUrl != "null") {
                                        // Пишем в очередь
                                        sh "echo 'GITHUB|${repo}|${latestVer}|${dUrl}' >> ${env.QUEUE_FILE}"
                                    } else {
                                        echo "   [ERROR] Regex mismatch for ${repo}"
                                        // Можно сделать currentBuild.result = 'UNSTABLE'
                                    }
                                } else {
                                    echo "   [OK] Up to date: ${latestVer}"
                                }
                            } catch (e) {
                                echo "   [ERROR] GitHub Check Failed: ${e.getMessage()}"
                            }
                        }
                    }

                    // --- ЧАСТЬ Б: ПРОВЕРКА WEB ---
                    if (fileExists(env.WEB_LIST)) {
                        def webLines = readFile(env.WEB_LIST).readLines()
                        webLines.each { line ->
                            if (line.startsWith('#') || !line.trim()) return

                            try {
                                def dUrl = ""
                                def mode = "SMART"
                                
                                // --- ПАРСИНГ ---
                                if (line.contains("videolan.org")) {
                                    def raw = sh(script: "curl -s https://download.videolan.org/pub/videolan/vlc/last/win64/ | grep -oP 'href=\"\\Kvlc-[0-9.]+-win64\\.exe' | head -n 1", returnStdout: true).trim()
                                    if (raw) dUrl = "https://download.videolan.org/pub/videolan/vlc/last/win64/${raw} https://download.videolan.org/pub/videolan/vlc/last/win32/${raw.replace('win64', 'win32')}"
                                }
                                else if (line.contains("telegram.org")) {
                                    def t64 = sh(script: "curl -s -o /dev/null -w '%{redirect_url}' 'https://telegram.org/dl/desktop/win64'", returnStdout: true).trim()
                                    def t32 = sh(script: "curl -s -o /dev/null -w '%{redirect_url}' 'https://telegram.org/dl/desktop/win'", returnStdout: true).trim()
                                    if (t64) dUrl = "${t64} ${t32}"
                                }
                                else if (line.contains("epicgames.com")) {
                                    def raw = sh(script: "curl -s -A ${env.UA} -o /dev/null -w '%{redirect_url}' '${line}'", returnStdout: true).trim()
                                    if (raw) dUrl = raw.split('\\?')[0]
                                }
                                else if (line.contains("mozilla.org")) {
                                    def raw = sh(script: "curl -s -o /dev/null -w '%{redirect_url}' 'https://download.mozilla.org/?product=firefox-latest-ssl&os=win64&lang=ru'", returnStdout: true).trim()
                                    if (raw) {
                                        def oldN = raw.split('/').last()
                                        def newN = sh(script: "echo '${oldN}' | sed 's/%20/./g'", returnStdout: true).trim()
                                        dUrl = "${raw}?name=${newN}"
                                    }
                                }
                                else if (line.contains("fraps.com")) {
                                    def ver = sh(script: "curl -s https://fraps.com/download.php | grep -oP 'Fraps \\K[0-9.]+' | head -n 1", returnStdout: true).trim()
                                    if (ver) dUrl = "https://beepa.com/free/setup.exe?name=fraps-${ver}-setup.exe"
                                }
                                else if (line.contains("qbittorrent.org") || line.contains("hwinfo.com")) {
                                    def rss = line.contains("qbittorrent") ? "https://sourceforge.net/projects/qbittorrent/rss?path=/qbittorrent-win32" : "https://sourceforge.net/projects/hwinfo/rss"
                                    def web = sh(script: "curl -s '${rss}' | grep -o 'https://[^\"<]*\\(x64_setup\\|hwi64_[0-9]\\+\\)\\.exe/download' | head -n 1", returnStdout: true).trim()
                                    if (web) {
                                        sh "curl -L -s -A ${env.UA} -o 'tmp/sf_check.html' '${web}'"
                                        def direct = sh(script: "grep -oP 'https://downloads\\.sourceforge\\.net/[^\"]+' tmp/sf_check.html | head -n 1", returnStdout: true).trim()
                                        sh "rm -f tmp/sf_check.html"
                                        if (direct) {
                                            def fn = web.replace("/download", "").split('/').last()
                                            if (line.contains("hwinfo")) fn = fn.replace(".exe", "").replace(".", "") + ".exe"
                                            dUrl = "${direct}?name=${fn}"
                                        }
                                    }
                                }
                                else {
                                    dUrl = line.trim()
                                    mode = "HASH"
                                }

                                if (!dUrl) return

                                // --- ПРОВЕРКА ---
                                dUrl.split(' ').each { url ->
                                    def fname = ""
                                    def cleanUrl = ""
                                    if (url.contains("?name=")) {
                                        def p = url.split("\\?name=")
                                        cleanUrl = p[0]; fname = p[1]
                                    } else {
                                        cleanUrl = url
                                        def rawName = cleanUrl.split('/').last().split('\\?')[0]
                                        fname = java.net.URLDecoder.decode(rawName, "UTF-8")
                                    }

                                    if (fname == "client" || fname == "download" || fname == "") return

                                    if (mode == "SMART") {
                                        if (!logLines.any { it.contains("|${fname}|") }) {
                                            echo "   [UPDATE DETECTED] Web App: ${fname}"
                                            updatesFound = true
                                            sh "echo 'WEB|${fname}|SMART|${url}' >> ${env.QUEUE_FILE}"
                                        } else {
                                            echo "   [OK] Web App: ${fname}"
                                        }
                                    } else {
                                        // HASH MODE
                                        echo "   [CHECKING HASH] ${fname}..."
                                        def headers = "-A ${env.UA} -L"
                                        if (cleanUrl.contains("techpowerup")) headers += " -e 'https://www.techpowerup.com/'"
                                        
                                        sh "curl -s ${headers} -o 'tmp/${fname}.check' '${cleanUrl}'"
                                        def newHash = sh(script: "sha256sum 'tmp/${fname}.check' | awk '{print \$1}'", returnStdout: true).trim()
                                        
                                        if (!logLines.any { it.contains("|${fname}|${newHash}|") }) {
                                            echo "   [UPDATE DETECTED] Hash mismatch: ${fname}"
                                            updatesFound = true
                                            sh "echo 'HASH_MOVE|${fname}|${newHash}|tmp/${fname}.check' >> ${env.QUEUE_FILE}"
                                        } else {
                                            echo "   [OK] Hash match: ${fname}"
                                            sh "rm 'tmp/${fname}.check'"
                                        }
                                    }
                                }

                            } catch (e) {
                                echo "   [ERROR] Web Check Failed: ${e.getMessage()}"
                            }
                        }
                    }

                    if (!updatesFound) {
                        echo "\n=============================================="
                        echo "NO UPDATES FOUND."
                        echo "Skipping Download stages."
                        echo "=============================================="
                    } else {
                        echo "\n>>> UPDATES FOUND! Proceeding..."
                    }
                }
            }
        }

        // ==========================================
        // 3. DOWNLOAD GITHUB
        // ==========================================
        stage('Download GitHub') {
            when { expression { fileExists(env.QUEUE_FILE) } }
            steps {
                script {
                    def queue = readFile(env.QUEUE_FILE).readLines()
                    def logLines = readFile(env.LOG_FILE).readLines() // Читаем актуальный лог

                    queue.each { item ->
                        def parts = item.split('\\|')
                        if (parts[0] == 'GITHUB') {
                            def repo = parts[1]
                            def version = parts[2]
                            def url = parts[3]
                            def fname = url.split('/').last()

                            echo ">>> [DOWNLOADING] GitHub: ${fname}"
                            sh "wget -qP ${env.DL_DIR} ${url}"

                            if (fname.endsWith(".zip")) {
                                def folder = fname.replace('.zip', '')
                                sh "mkdir -p ${env.DL_DIR}/${folder}"
                                sh "unzip -o ${env.DL_DIR}/${fileName} -d ${env.DL_DIR}/${folder}/"
                                sh "rm ${env.DL_DIR}/${fileName}"
                            }

                            // Обновляем лог
                            logLines.removeAll { it.startsWith("${repo}|") }
                            def dateNow = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
                            logLines.add("${repo}|${version}|${dateNow}")
                        }
                    }
                    writeFile file: env.LOG_FILE, text: logLines.join("\n") + "\n"
                }
            }
        }

        // ==========================================
        // 4. DOWNLOAD WEB
        // ==========================================
        stage('Download Web') {
            when { expression { fileExists(env.QUEUE_FILE) } }
            steps {
                script {
                    def queue = readFile(env.QUEUE_FILE).readLines()
                    def logLines = readFile(env.LOG_FILE).readLines()

                    queue.each { item ->
                        def parts = item.split('\\|')
                        def type = parts[0]
                        
                        if (type == 'WEB') {
                            def fname = parts[1]
                            def url = parts[3]
                            def cleanUrl = url.contains("?name=") ? url.split("\\?name=")[0] : url

                            echo ">>> [DOWNLOADING] Web App: ${fname}"
                            def headers = "-A ${env.UA} -L"
                            if (cleanUrl.contains("techpowerup")) headers += " -e 'https://www.techpowerup.com/'"
                            
                            sh "curl -s ${headers} -o '${env.DL_DIR}/${fname}' '${cleanUrl}'"
                            
                            logLines.removeAll { it.contains("|${fname}|") }
                            def dateNow = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
                            logLines.add("WEB-SMART|${fname}|${dateNow}")
                        }
                        else if (type == 'HASH_MOVE') {
                            def fname = parts[1]
                            def hash = parts[2]
                            def tmpPath = parts[3]

                            echo ">>> [SAVING] Verified Hash File: ${fname}"
                            sh "mv '${tmpPath}' '${env.DL_DIR}/${fname}'"
                            
                            logLines.removeAll { it.contains("|${fname}|") }
                            def dateNow = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
                            logLines.add("WEB-HASH|${fname}|${hash}|${dateNow}")
                        }
                    }
                    writeFile file: env.LOG_FILE, text: logLines.join("\n") + "\n"
                }
            }
        }
    }
}
