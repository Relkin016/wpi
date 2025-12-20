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
        stage('Preparation') {
            steps {
                script {
                    echo "--- [PREP] Preparing workspace ---"
                    sh "mkdir -p ${env.DL_DIR}"
                    sh "mkdir -p tmp"
                    sh "rm -f ${env.DL_DIR}/*.new"
                    sh "rm -f ${env.DL_DIR}/*.html"
                    sh "rm -f tmp/*.json"
                    sh "rm -f ${env.QUEUE_FILE}"
                    
                    if (!fileExists(env.LOG_FILE)) {
                        sh "touch ${env.LOG_FILE}"
                    }
                }
            }
        }

        stage('Check Versions') {
            steps {
                echo "--- [CHECK] Comparing remote versions with local log ---"
                script {
                    def logLines = readFile(env.LOG_FILE).readLines()
                    def updatesFound = false
                    def addedToQueue = [] // Для предотвращения дублей в очереди (например, Steam)

                    // --- GITHUB ---
                    if (fileExists(env.GITHUB_LIST)) {
                        def ghLines = readFile(env.GITHUB_LIST).readLines()
                        ghLines.each { line ->
                            if (line.startsWith('#') || !line.trim()) return
                            try {
                                def parts = line.split(';')
                                def url = parts[0].trim()
                                def regex = parts[1].trim()
                                def repo = url.replace("https://github.com/", "").replace(/\/$/, "")

                                sh "curl -s -H User-Agent:${env.UA} https://api.github.com/repos/${repo}/releases/latest -o tmp/gh_${repo.replace('/', '_')}.json"
                                def jsonFile = "tmp/gh_${repo.replace('/', '_')}.json"
                                
                                if (readFile(jsonFile).contains("rate limit exceeded")) error "GITHUB LIMIT EXCEEDED"
                                
                                def latestVer = sh(script: "jq -r .tag_name ${jsonFile}", returnStdout: true).trim()
                                if (!latestVer || latestVer == "null") return

                                if (!logLines.any { it.contains("${repo}|${latestVer}|") }) {
                                    def dUrl = sh(script: "jq -r --arg REGEX \"${regex}\" '.assets[] | select(.name | test(\$REGEX; \"i\")) | .browser_download_url' ${jsonFile} | head -n 1", returnStdout: true).trim()
                                    if (dUrl && dUrl != "null") {
                                        echo "   [UPDATE] GitHub: ${repo} (${latestVer})"
                                        updatesFound = true
                                        sh "echo 'GITHUB|${repo}|${latestVer}|${dUrl}' >> ${env.QUEUE_FILE}"
                                    }
                                }
                            } catch (e) { echo "   [ERR] ${line}: ${e.message}" }
                        }
                    }

                    // --- WEB ---
                    if (fileExists(env.WEB_LIST)) {
                        def webLines = readFile(env.WEB_LIST).readLines()
                        webLines.each { line ->
                            if (line.startsWith('#') || !line.trim()) return
                            try {
                                def dUrls = []
                                def mode = "SMART"
                                
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
                                else { dUrls.add(line.trim()); mode = "HASH" }

                                dUrls.each { url ->
                                    if (!url || url == "null") return
                                    def fname = url.contains("?name=") ? url.split("\\?name=")[1] : java.net.URLDecoder.decode(url.split('/').last().split('\\?')[0], "UTF-8")
                                    if (fname == "client" || fname == "download" || addedToQueue.contains(fname)) return

                                    if (mode == "SMART") {
                                        if (!logLines.any { it.contains("|${fname}|") }) {
                                            echo "   [UPDATE] Web: ${fname}"
                                            updatesFound = true; addedToQueue.add(fname)
                                            sh "echo 'WEB|${fname}|SMART|${url}' >> ${env.QUEUE_FILE}"
                                        }
                                    } else {
                                        echo "   [HASH CHECK] ${fname}..."
                                        sh "curl -s -L -A ${env.UA} -o 'tmp/${fname}.check' '${url}'"
                                        def newHash = sh(script: "sha256sum 'tmp/${fname}.check' | awk '{print \$1}'", returnStdout: true).trim()
                                        if (!logLines.any { it.contains("|${fname}|${newHash}|") }) {
                                            echo "   [UPDATE] Hash mismatch: ${fname}"
                                            updatesFound = true; addedToQueue.add(fname)
                                            sh "echo 'HASH_MOVE|${fname}|${newHash}|tmp/${fname}.check' >> ${env.QUEUE_FILE}"
                                        } else { sh "rm 'tmp/${fname}.check'" }
                                    }
                                }
                            } catch (e) { echo "   [ERR] Web: ${e.message}" }
                        }
                    }
                    if (!updatesFound) { echo "NO UPDATES FOUND."; currentBuild.result = 'SUCCESS' }
                }
            }
        }

        stage('Download GitHub') {
            when { expression { fileExists(env.QUEUE_FILE) } }
            steps {
                script {
                    def queue = readFile(env.QUEUE_FILE).readLines()
                    def logLines = readFile(env.LOG_FILE).readLines()
                    queue.each { item ->
                        def parts = item.split('\\|')
                        if (parts[0] == 'GITHUB') {
                            def repo = parts[1], version = parts[2], url = parts[3]
                            def fname = url.split('/').last()
                            echo ">>> [DOWN] GitHub: ${fname}"
                            sh "wget -qP ${env.DL_DIR} '${url}'"
                            if (fname.endsWith(".zip")) {
                                def folder = fname.replace('.zip', '')
                                sh "mkdir -p ${env.DL_DIR}/${folder} && unzip -o ${env.DL_DIR}/${fname} -d ${env.DL_DIR}/${folder}/ && rm ${env.DL_DIR}/${fname}"
                            }
                            logLines.removeAll { it.startsWith("${repo}|") }
                            logLines.add("${repo}|${version}|${sh(script: 'date +%Y-%m-%d', returnStdout: true).trim()}")
                        }
                    }
                    writeFile file: env.LOG_FILE, text: logLines.join("\n") + "\n"
                }
            }
        }

        stage('Download Web') {
            when { expression { fileExists(env.QUEUE_FILE) } }
            steps {
                script {
                    def queue = readFile(env.QUEUE_FILE).readLines()
                    def logLines = readFile(env.LOG_FILE).readLines()
                    queue.each { item ->
                        def parts = item.split('\\|')
                        if (parts[0] == 'WEB') {
                            def fname = parts[1], url = parts[3]
                            echo ">>> [DOWN] Web: ${fname}"
                            def cleanUrl = url.contains("?name=") ? url.split("\\?name=")[0] : url
                            sh "curl -s -L -A ${env.UA} -o '${env.DL_DIR}/${fname}' '${cleanUrl}'"
                            logLines.removeAll { it.contains("|${fname}|") }
                            logLines.add("WEB-SMART|${fname}|${sh(script: 'date +%Y-%m-%d', returnStdout: true).trim()}")
                        }
                        else if (parts[0] == 'HASH_MOVE') {
                            def fname = parts[1], hash = parts[2], tmpPath = parts[3]
                            echo ">>> [SAVE] Hash Verified: ${fname}"
                            sh "mv '${tmpPath}' '${env.DL_DIR}/${fname}'"
                            logLines.removeAll { it.contains("|${fname}|") }
                            logLines.add("WEB-HASH|${fname}|${hash}|${sh(script: 'date +%Y-%m-%d', returnStdout: true).trim()}")
                        }
                    }
                    writeFile file: env.LOG_FILE, text: logLines.join("\n") + "\n"
                    sh "rm -f tmp/*.check tmp/sf_check.html" // Финальная чистка
                }
            }
        }
    }
}
