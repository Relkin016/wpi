pipeline {
    agent any

    environment {
        DL_DIR = "tmp/downloads"
        LOG_FILE = "repos/web_log.log"
        GITHUB_LIST = "repos/github.txt"
        WEB_LIST = "repos/web.txt"
        // Глобальный User-Agent (обязателен для SourceForge, HWInfo, Epic)
        UA = "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"
    }

    stages {
        stage('Cleanup') {
            steps {
                script {
                    echo "Cleaning up workspace..."
                    sh "mkdir -p ${env.DL_DIR}"
                    // Чистим временные файлы
                    sh "rm -f ${env.DL_DIR}/*.new"
                    sh "rm -f ${env.DL_DIR}/*.html"
                    if (!fileExists(env.LOG_FILE)) {
                        sh "touch ${env.LOG_FILE}"
                    }
                }
            }
        }

        stage('Checking GitHub') {
            steps {
                echo "Starting GitHub checks..."
                script {
                    if (!fileExists(env.GITHUB_LIST)) error "File ${env.GITHUB_LIST} not found!"
                    
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
                                    
                                    // АВТО-РАСПАКОВКА ZIP
                                    if (fileName.endsWith(".zip")) {
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
                    if (!fileExists(env.WEB_LIST)) error "File ${env.WEB_LIST} not found!"

                    def lines = readFile(env.WEB_LIST).readLines()
                    
                    lines.each { line ->
                        if (line.startsWith('#') || !line.trim()) return
                        
                        def urls = [] 
                        def mode = "HASH"

                        // ===========================
                        //        БЛОК ПАРСЕРОВ
                        // ===========================

                        // --- 1. VLC (x64 + x86) ---
                        if (line.contains("videolan.org")) {
                            echo ">>> Parsing VLC..."
                            // Win64
                            def v64 = sh(script: "curl -s https://download.videolan.org/pub/videolan/vlc/last/win64/ | grep -oP 'href=\"\\Kvlc-[0-9.]+-win64\\.exe' | head -n 1", returnStdout: true).trim()
                            if (v64) urls.add("https://download.videolan.org/pub/videolan/vlc/last/win64/${v64}")
                            
                            // Win32 (x86) - ВЕРНУЛ НА МЕСТО
                            def v32 = sh(script: "curl -s https://download.videolan.org/pub/videolan/vlc/last/win32/ | grep -oP 'href=\"\\Kvlc-[0-9.]+-win32\\.exe' | head -n 1", returnStdout: true).trim()
                            if (v32) urls.add("https://download.videolan.org/pub/videolan/vlc/last/win32/${v32}")
                            
                            mode = "SMART"
                        } 

                        // --- 2. TELEGRAM (x64 + x86) ---
                        else if (line.contains("telegram.org")) {
                            echo ">>> Parsing Telegram..."
                            // Win64
                            def tg64 = sh(script: "curl -s -o /dev/null -w '%{redirect_url}' 'https://telegram.org/dl/desktop/win64'", returnStdout: true).trim()
                            if (tg64) urls.add(tg64)
                            // Win32
                            def tg32 = sh(script: "curl -s -o /dev/null -w '%{redirect_url}' 'https://telegram.org/dl/desktop/win'", returnStdout: true).trim()
                            if (tg32) urls.add(tg32)
                            
                            mode = "SMART"
                        }

                        // --- 3. EPIC GAMES ---
                        else if (line.contains("epicgames.com")) {
                            echo ">>> Parsing Epic Games..."
                            def rawUrl = sh(script: "curl -s -A '${env.UA}' -o /dev/null -w '%{redirect_url}' '${line}'", returnStdout: true).trim()
                            if (rawUrl) {
                                urls.add(rawUrl.split('\\?')[0])
                                mode = "SMART"
                            }
                        }

                        // --- 4. FIREFOX ---
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

                        // --- 5. FRAPS ---
                        else if (line.contains("fraps.com")) {
                             echo ">>> Parsing Fraps..."
                             def frapsVer = sh(script: "curl -s https://fraps.com/download.php | grep -oP 'Fraps \\K[0-9.]+' | head -n 1", returnStdout: true).trim()
                             if (frapsVer) {
                                 urls.add("https://beepa.com/free/setup.exe?name=fraps-${frapsVer}-setup.exe")
                                 mode = "SMART"
                             }
                        }

                        // --- 6. OPENVPN (ОТЛОЖЕНО) ---
                        /* else if (line.contains("openvpn.net")) {
                           // Логика отключена по запросу
                        }
                        */

                        // --- 7. SOURCEFORGE (qBittorrent + HWInfo) ---
                        else if (line.contains("qbittorrent.org") || line.contains("hwinfo.com")) {
                            echo ">>> Parsing SourceForge (qBit/HWInfo)..."
                            
                            def rssLink = ""
                            if (line.contains("qbittorrent")) rssLink = "https://sourceforge.net/projects/qbittorrent/rss?path=/qbittorrent-win32"
                            if (line.contains("hwinfo")) rssLink = "https://sourceforge.net/projects/hwinfo/rss"

                            def webUrl = sh(script: "curl -s '${rssLink}' | grep -o 'https://[^\"<]*\\(x64_setup\\|hwi64_[0-9]\\+\\)\\.exe/download' | head -n 1", returnStdout: true).trim()
                            
                            if (webUrl) {
                                sh "curl -L -s -A '${env.UA}' -o 'tmp/sf_temp.html' '${webUrl}'"
                                def directUrl = sh(script: "grep -oP 'https://downloads\\.sourceforge\\.net/[^\"]+' tmp/sf_temp.html | head -n 1", returnStdout: true).trim()
                                sh "rm -f tmp/sf_temp.html"

                                if (directUrl) {
                                    def fName = webUrl.replace("/download", "").split('/').last()
                                    // HWInfo fix (remove dots from name body, keep extension)
                                    if (line.contains("hwinfo")) {
                                         fName = fName.replace(".exe", "").replace(".", "") + ".exe"
                                    }
                                    
                                    urls.add("${directUrl}?name=${fName}")
                                    mode = "SMART"
                                }
                            }
                        }

                        // --- 8. HASH CHECK (Остальные + OpenVPN попадет сюда) ---
                        else {
                            urls.add(line.trim())
                            mode = "HASH"
                        }

                        // ===========================
                        //       БЛОК ЗАГРУЗКИ
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

                            def headers = "-A '${env.UA}' -L"
                            if (cleanUrl.contains("techpowerup.com")) {
                                headers += " -e 'https://www.techpowerup.com/'"
                            }

                            if (mode == "SMART") {
                                if (fileExists("${env.DL_DIR}/${fname}")) {
                                    echo "   [SKIP] ${fname} already exists."
                                } else {
                                    echo "   [DOWN] Downloading ${fname}..."
                                    sh "curl -s ${headers} -o '${env.DL_DIR}/${fname}' '${cleanUrl}'"
                                    
                                    def fSize = sh(script: "wc -c < '${env.DL_DIR}/${fname}'", returnStdout: true).trim()
                                    if (fSize == "0" || fSize.toInteger() < 10000) {
                                        echo "   [FAIL] File too small (${fSize} bytes). Deleting."
                                        sh "rm '${env.DL_DIR}/${fname}'"
                                    } else {
                                        def dateNow = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
                                        sh "sed -i '\\#|${fname}|#d' ${env.LOG_FILE}"
                                        sh "echo 'WEB-SMART|${fname}|${dateNow}' >> ${env.LOG_FILE}"
                                    }
                                }
                            } else {
                                // HASH CHECK MODE
                                echo "   [HASH CHECK] ${fname}..."
                                sh "curl -s ${headers} -o '${env.DL_DIR}/${fname}.new' '${cleanUrl}'"
                                
                                def fSize = sh(script: "wc -c < '${env.DL_DIR}/${fname}.new'", returnStdout: true).trim()
                                if (fSize == "0" || fSize.toInteger() < 10000) {
                                    sh "rm '${env.DL_DIR}/${fname}.new'"
                                } else {
                                    def newHash = sh(script: "sha256sum '${env.DL_DIR}/${fname}.new' | awk '{print \$1}'", returnStdout: true).trim()
                                    def oldHash = sh(script: "grep '|${fname}|' ${env.LOG_FILE} | tail -n 1 | cut -d '|' -f 3 || echo 'none'", returnStdout: true).trim()
                                    
                                    if (newHash != oldHash) {
                                        echo "   [UPDATE] Hash mismatch! Updating..."
                                        sh "mv '${env.DL_DIR}/${fname}.new' '${env.DL_DIR}/${fname}'"
                                        def dateNow = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
                                        sh "sed -i '\\#|${fname}|#d' ${env.LOG_FILE}"
                                        sh "echo 'WEB-HASH|${fname}|${newHash}|${dateNow}' >> ${env.LOG_FILE}"
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
}
