pipeline {
    agent any

    environment {
        DL_DIR = "tmp/downloads"
        LOG_FILE = "repos/web_log.log"
        GITHUB_LIST = "repos/github.txt"
        WEB_LIST = "repos/web.txt"
        QUEUE_FILE = "tmp/download_queue.txt"
        UA = "Mozilla/5.0_Windows_NT_10.0_Win64_x64"
    }

    stages {
        // ==========================================
        // 1. ПРЕДВАРИТЕЛЬНАЯ ОЧИСТКА
        // ==========================================
        stage('Preparation') {
            steps {
                script {
                    echo "--- [PREP] Preparing workspace ---"
                    // ВАЖНО: При полной пересборке мы чистим папку загрузок,
                    // чтобы там не осталось старого мусора, если версия понизилась.
                    sh "mkdir -p ${env.DL_DIR}"
                    sh "mkdir -p tmp"
                    
                    // Удаляем всё из папки загрузок, так как мы планируем качать всё заново
                    // (Если обновлений не будет - папка останется пустой, и это ок, так как сборки ISO не будет)
                    sh "rm -rf ${env.DL_DIR}/*"
                    
                    sh "rm -f tmp/*.json"
                    sh "rm -f tmp/*.check"
                    sh "rm -f ${env.QUEUE_FILE}"
                    
                    if (!fileExists(env.LOG_FILE)) {
                        sh "touch ${env.LOG_FILE}"
                    }
                }
            }
        }

        // ==========================================
        // 2. CHECK VERSIONS (СБОР ДАННЫХ И ПРОВЕРКА)
        // ==========================================
        stage('Check Versions') {
            steps {
                echo "--- [CHECK] Scanning all sources ---"
                script {
                    def logLines = readFile(env.LOG_FILE).readLines()
                    
                    // Флаг: нужно ли запускать скачивание?
                    def globalUpdateNeeded = false
                    
                    // В этот список мы собираем ВСЕ ссылки на ВСЕ программы.
                    // Если globalUpdateNeeded станет true, мы скачаем весь этот список.
                    def fullQueueList = [] 
                    
                    def addedNames = [] // Чтобы избегать дублей (напр. Steam)

                    // ------------------------------------------------
                    // ЧАСТЬ A: GITHUB (Собираем инфо по всем репо)
                    // ------------------------------------------------
                    if (fileExists(env.GITHUB_LIST)) {
                        def ghLines = readFile(env.GITHUB_LIST).readLines()
                        ghLines.each { line ->
                            if (line.startsWith('#') || !line.trim()) return
                            try {
                                def parts = line.split(';')
                                def url = parts[0].trim()
                                def regex = parts[1].trim()
                                def repo = url.replace("https://github.com/", "").replace(/\/$/, "")

                                // Качаем JSON
                                sh "curl -s -H User-Agent:${env.UA} https://api.github.com/repos/${repo}/releases/latest -o tmp/gh_${repo.replace('/', '_')}.json"
                                def jsonFile = "tmp/gh_${repo.replace('/', '_')}.json"
                                
                                if (readFile(jsonFile).contains("rate limit exceeded")) error "GITHUB LIMIT EXCEEDED"
                                
                                def latestVer = sh(script: "jq -r .tag_name ${jsonFile}", returnStdout: true).trim()
                                if (!latestVer || latestVer == "null") return

                                // Получаем ссылку ВСЕГДА (даже если версия совпадает)
                                def dUrl = sh(script: "jq -r --arg REGEX \"${regex}\" '.assets[] | select(.name | test(\$REGEX; \"i\")) | .browser_download_url' ${jsonFile} | head -n 1", returnStdout: true).trim()
                                
                                if (dUrl && dUrl != "null") {
                                    // Добавляем в общий список
                                    fullQueueList.add("GITHUB|${repo}|${latestVer}|${dUrl}")

                                    // ПРОВЕРКА: Отличается ли версия от лога?
                                    if (!logLines.any { it.contains("${repo}|${latestVer}|") }) {
                                        echo "   [UPDATE DETECTED] GitHub: ${repo} (${latestVer})"
                                        globalUpdateNeeded = true
                                    } else {
                                        echo "   [OK] GitHub: ${repo} (${latestVer})"
                                    }
                                }
                            } catch (e) { echo "   [ERR] ${line}: ${e.message}" }
                        }
                    }

                    // ------------------------------------------------
                    // ЧАСТЬ B: WEB (Собираем инфо по всем сайтам)
                    // ------------------------------------------------
                    if (fileExists(env.WEB_LIST)) {
                        def webLines = readFile(env.WEB_LIST).readLines()
                        webLines.each { line ->
                            if (line.startsWith('#') || !line.trim()) return
                            try {
                                def dUrls = []
                                def mode = "SMART"
                                
                                // --- ПАРСИНГ ---
                                if (line.contains("videolan.org")) {
                                    def raw = sh(script: "curl -s https://download.videolan.org/pub/videolan/vlc/last/win64/ | grep -oP 'href=\"\\Kvlc-[0-9.]+-win64\\.exe' | head -n 1", returnStdout: true).trim()
                                    if (raw) {
                                        dUrls.add("https://download.videolan.org/pub/videolan/vlc/last/win64/${raw}")
                                        dUrls.add("https://download.videolan.org/pub/videolan/vlc/last/win32/${raw.replace('win64', 'win32')}")
                                    }
                                }
                                else if (line.contains("telegram.org")) {
                                    dUrls.add(sh(script: "curl -s -o /dev/null -w '%{redirect_url}' 'https://telegram.org/dl/desktop/win64'", returnStdout: true).trim())
                                    dUrls.add(sh(script: "curl -s -o /dev/null -w '%{redirect_url}' 'https://telegram.org/dl/desktop/win'", returnStdout: true).trim())
                                }
                                else if (line.contains("epicgames.com")) {
                                    dUrls.add(sh(script: "curl -s -A ${env.UA} -o /dev/null -w '%{redirect_url}' '${line}'", returnStdout: true).trim().split('\\?')[0])
                                }
                                else if (line.contains("mozilla.org")) {
                                    def raw = sh(script: "curl -s -o /dev/null -w '%{redirect_url}' 'https://download.mozilla.org/?product=firefox-latest-ssl&os=win64&lang=ru'", returnStdout: true).trim()
                                    if (raw) {
                                        def newN = sh(script: "echo '${raw.split('/').last()}' | sed s/%20/./g", returnStdout: true).trim()
                                        dUrls.add("${raw}?name=${newN}")
                                    }
                                }
                                else if (line.contains("fraps.com")) {
                                    def ver = sh(script: "curl -s https://fraps.com/download.php | grep -oP 'Fraps \\K[0-9.]+' | head -n 1", returnStdout: true).trim()
                                    if (ver) dUrls.add("https://beepa.com/free/setup.exe?name=fraps-${ver}-setup.exe")
                                }
                                else if (line.contains("qbittorrent.org") || line.contains("hwinfo.com")) {
                                    def rss = line.contains("qbittorrent") ? "https://sourceforge.net/projects/qbittorrent/rss?path=/qbittorrent-win32" : "https://sourceforge.net/projects/hwinfo/rss"
                                    def web = sh(script: "curl -s '${rss}' | grep -o 'https://[^\"<]*\\(x64_setup\\|hwi64_[0-9]\\+\\)\\.exe/download' | head -n 1", returnStdout: true).trim()
                                    if (web) {
                                        sh "curl -L -s -A ${env.UA} -o 'tmp/sf_check.html' '${web}'"
                                        def direct = sh(script: "grep -oP 'https://downloads\\.sourceforge\\.net/[^\"]+' tmp/sf_check.html | head -n 1", returnStdout: true).trim()
                                        if (direct) {
                                            def fn = web.replace("/download", "").split('/').last()
                                            if (line.contains("hwinfo")) fn = fn.replace(".exe", "").replace(".", "") + ".exe"
                                            dUrls.add("${direct}?name=${fn}")
                                        }
                                    }
                                }
                                else { 
                                    dUrls.add(line.trim())
                                    mode = "HASH" 
                                }

                                // --- ОБРАБОТКА ССЫЛОК ---
                                dUrls.each { url ->
                                    if (!url || url == "null") return
                                    
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

                                    if (fname == "client" || fname == "download" || addedNames.contains(fname)) return

                                    if (mode == "SMART") {
                                        // Добавляем в список ВСЕГДА
                                        fullQueueList.add("WEB|${fname}|SMART|${url}")
                                        
                                        // Проверяем по логу
                                        if (!logLines.any { it.contains("|${fname}|") }) {
                                            echo "   [UPDATE DETECTED] Web: ${fname}"
                                            globalUpdateNeeded = true
                                        } else {
                                            echo "   [OK] Web: ${fname}"
                                        }
                                    } else {
                                        // HASH CHECK MODE (Steam, OpenVPN)
                                        // Нам нужно скачать файл для проверки хеша.
                                        // Если глобальный апдейт УЖЕ нужен, можно не качать? Нет, нам нужен актуальный хеш.
                                        
                                        echo "   [CHECKING HASH] ${fname}..."
                                        def headers = "-A ${env.UA} -L"
                                        if (cleanUrl.contains("techpowerup")) headers += " -e 'https://www.techpowerup.com/'"
                                        
                                        sh "curl -s ${headers} -o 'tmp/${fname}.check' '${cleanUrl}'"
                                        def newHash = sh(script: "sha256sum 'tmp/${fname}.check' | awk '{print \$1}'", returnStdout: true).trim()
                                        
                                        // ВАЖНО: Добавляем в очередь как "MOVE", так как мы уже скачали файл в .check
                                        // Это сэкономит трафик при полной загрузке
                                        fullQueueList.add("HASH_MOVE|${fname}|${newHash}|tmp/${fname}.check")
                                        
                                        if (!logLines.any { it.contains("|${fname}|${newHash}|") }) {
                                            echo "   [UPDATE DETECTED] Hash mismatch: ${fname}"
                                            globalUpdateNeeded = true
                                        } else {
                                            echo "   [OK] Hash match: ${fname}"
                                        }
                                    }
                                    addedNames.add(fname)
                                }
                            } catch (e) { echo "   [ERR] Web: ${e.message}" }
                        }
                    }

                    // ------------------------------------------------
                    // ФИНАЛЬНОЕ РЕШЕНИЕ
                    // ------------------------------------------------
                    if (!globalUpdateNeeded) {
                        echo "\n=============================================="
                        echo "NO UPDATES FOUND. SYSTEM IS UP TO DATE."
                        echo "Exiting pipeline early. No downloads needed."
                        echo "=============================================="
                        currentBuild.result = 'SUCCESS'
                    } else {
                        echo "\n>>> UPDATES DETECTED! TRIGGERING FULL DOWNLOAD..."
                        echo ">>> Queue size: ${fullQueueList.size()} items."
                        
                        // Записываем ПОЛНЫЙ список в файл очереди
                        writeFile file: env.QUEUE_FILE, text: fullQueueList.join("\n") + "\n"
                    }
                }
            }
        }

        // ==========================================
        // 3. DOWNLOAD GITHUB (ПОЛНАЯ ЗАГРУЗКА)
        // ==========================================
        stage('Download GitHub') {
            when { expression { fileExists(env.QUEUE_FILE) } }
            steps {
                script {
                    def queue = readFile(env.QUEUE_FILE).readLines()
                    def logLines = readFile(env.LOG_FILE).readLines()

                    queue.each { item ->
                        def parts = item.split('\\|')
                        if (parts[0] == 'GITHUB') {
                            def repo = parts[1]
                            def version = parts[2]
                            def url = parts[3]
                            def fname = url.split('/').last()

                            echo ">>> [DOWNLOADING] GitHub: ${fname}"
                            sh "wget -qP ${env.DL_DIR} '${url}'"

                            if (fname.endsWith(".zip")) {
                                def folder = fname.replace('.zip', '')
                                sh "mkdir -p ${env.DL_DIR}/${folder}"
                                sh "unzip -o ${env.DL_DIR}/${fname} -d ${env.DL_DIR}/${folder}/"
                                sh "rm ${env.DL_DIR}/${fname}"
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
        // 4. DOWNLOAD WEB (ПОЛНАЯ ЗАГРУЗКА)
        // ==========================================
        stage('Download Web') {
            when { expression { fileExists(env.QUEUE_FILE) } }
            steps {
                script {
                    def queue = readFile(env.QUEUE_FILE).readLines()
                    def logLines = readFile(env.LOG_FILE).readLines()

                    queue.each { item ->
                        def parts = item.split('\\|')
                        
                        if (parts[0] == 'WEB') {
                            def fname = parts[1]
                            def url = parts[3]
                            def cleanUrl = url.contains("?name=") ? url.split("\\?name=")[0] : url

                            echo ">>> [DOWNLOADING] Web: ${fname}"
                            def headers = "-A ${env.UA} -L"
                            if (cleanUrl.contains("techpowerup")) headers += " -e 'https://www.techpowerup.com/'"
                            
                            sh "curl -s ${headers} -o '${env.DL_DIR}/${fname}' '${cleanUrl}'"
                            
                            logLines.removeAll { it.contains("|${fname}|") }
                            def dateNow = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
                            logLines.add("WEB-SMART|${fname}|${dateNow}")
                        }
                        else if (parts[0] == 'HASH_MOVE') {
                            def fname = parts[1]
                            def hash = parts[2]
                            def tmpPath = parts[3]

                            echo ">>> [SAVING] From Check Cache: ${fname}"
                            sh "mv '${tmpPath}' '${env.DL_DIR}/${fname}'"
                            
                            logLines.removeAll { it.contains("|${fname}|") }
                            def dateNow = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
                            logLines.add("WEB-HASH|${fname}|${hash}|${dateNow}")
                        }
                    }
                    writeFile file: env.LOG_FILE, text: logLines.join("\n") + "\n"
                    // Финальная чистка
                    sh "rm -f tmp/*.check tmp/sf_check.html"
                }
            }
        }
    }
}
