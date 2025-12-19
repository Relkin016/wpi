pipeline {
    agent any

    environment {
        // Пути
        DL_DIR = "tmp/downloads"
        LOG_FILE = "repos/web_log.log"
        GITHUB_LIST = "repos/github.txt"
        WEB_LIST = "repos/web.txt"
    }

    stages {
        stage('Cleanup Old Data') {
            steps {
                echo "Cleaning up workspace..."
                // Удаляем временные файлы загрузок (.new), но оставляем скачанное
                sh "mkdir -p ${DL_DIR}"
                sh "rm -f ${DL_DIR}/*.new"
                // Если лога нет, создаем
                sh "touch ${LOG_FILE}"
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
                                    
                                    // РАСПАКОВКА ZIP (WuaCpuFix и др)
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

                        // --- 1. VLC ---
                        if (line.contains("videolan.org")) {
                            echo ">>> Parsing VLC..."
                            def v64 = sh(script: "curl -s https://download.videolan.org/pub/videolan/vlc/last/win64/ | grep -oP 'href=\"\\Kvlc-[0-9.]+-win64\\.exe' | head -n 1", returnStdout: true)
                            def v32 = sh(script: "curl -s https://download.videolan.org/pub/videolan/vlc/last/win32/ | grep -oP 'href=\"\\Kvlc-[0-9.]+-win32\\.exe' | head -n 1", returnStdout: true)
                            
                            if (v64) urls.add("https://download.videolan.org/pub/videolan/vlc/last/win64/${v64.trim()}")
                            if (v32) urls.add("https://download.videolan.org/pub/videolan/vlc/last/win32/${v32.trim()}")
                            mode = "SMART"
                        } 

                        // --- 2. Telegram ---
                        else if (line.contains("telegram.org")) {
                            echo ">>> Parsing Telegram..."
                            def tg = sh(script: "curl -s -I 'https://telegram.org/dl/desktop/win64' | grep -i 'location:' | awk '{print \$2}' | tr -d '\r'", returnStdout: true)
                            if (tg) urls.add(tg.trim())
                            mode = "SMART"
                        }
                        // --- 3. FRAPS ---
                        else if (line.contains("fraps.com")) {
                             echo ">>> Parsing Fraps..."
                             def frapsVer = sh(script: "curl -s https://fraps.com/download.php | grep -oP 'Fraps \\K[0-9.]+' | head -n 1", returnStdout: true)
                             if (frapsVer) {
                                 def fVer = frapsVer.trim()
                                 urls.add("https://beepa.com/free/setup.exe?name=FRAPS-${fVer}-setup.exe")
                                 mode = "SMART"
                             }
                        }
                        // --- 4. TechPowerUp ---
                        else if (line.contains("techpowerup.com")) {
                            echo ">>> Parsing TechPowerUp..."
                            def ua = "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"
                            
                            def tpuID = sh(script: "curl -s -A '${ua}' '${line}' | grep -oP 'href=\"/download/start/\\?id=\\K\\d+' | head -n 1", returnStdout: true).trim()
                            
                            if (tpuID) {
                                def ver = ""
                                def finalName = ""
                                
                                // NVCleanstall
                                if (line.contains("nvcleanstall")) {
                                    ver = sh(script: "curl -s -A '${ua}' '${line}' | grep -oP 'NVCleanstall v\\K[0-9.]+' | head -n 1", returnStdout: true).trim()
                                    if (ver) finalName = "NVCleanstall_${ver}.exe"
                                } 
                                // Visual C++
                                else if (line.contains("visual-c")) {
                                    def rawDate = sh(script: "curl -s -A '${ua}' '${line}' | grep -oP '[A-Z][a-z]+ [0-9]{1,2}(st|nd|rd|th), [0-9]{4}' | head -n 1", returnStdout: true).trim()
                                    if (rawDate) {
                                        ver = sh(script: "echo '${rawDate}' | sed -E 's/(st|nd|rd|th),/,/g' | xargs -I {} date -d '{}' +%Y-%m-%d", returnStdout: true).trim()
                                        finalName = "Visual-C-Runtimes-AIO_${ver}.zip"
                                    }
                                }

                                if (finalName) {
                                    echo "   [TPU] Detected: ${finalName}"
                                    def startUrl = "https://www.techpowerup.com/download/start/?id=${tpuID}"
                                    def mirrorUrl = sh(script: "curl -s -A '${ua}' -e '${line}' '${startUrl}' | grep -oP 'action=\"\\K[^\"]+' | head -n 1", returnStdout: true).trim()
                                    
                                    if (mirrorUrl) {
                                        if (mirrorUrl.startsWith("//")) mirrorUrl = "https:" + mirrorUrl
                                        urls.add("${mirrorUrl}?name=${finalName}")
                                        mode = "SMART"
                                    }
                                }
                            }
                        }
                        // --- 5. Firefox ---
                        else if (line.contains("mozilla.org")) {
                            echo ">>> Parsing Firefox..."
                            def ff = sh(script: "curl -s -I 'https://download.mozilla.org/?product=firefox-latest-ssl&os=win64&lang=ru' | grep -i 'location:' | awk '{print \$2}' | tr -d '\r'", returnStdout: true)
                            if (ff) urls.add(ff.trim())
                            mode = "SMART"
                        }
                        // --- 6. OpenVPN ---
                        else if (line.contains("openvpn.net")) {
                            echo ">>> Parsing OpenVPN..."
                            def vpn = sh(script: "curl -s -A 'Mozilla/5.0' 'https://openvpn.net/client/' | grep -oP 'https://swupdate.openvpn.net/downloads/connect/openvpn-connect-[0-9.]+_signed-x64\\.msi' | head -n 1", returnStdout: true)
                            if (vpn) urls.add(vpn.trim())
                            mode = "SMART"
                        }
                        // --- 7. HWInfo (RSS) ---
                        else if (line.contains("hwinfo.com")) {
                            echo ">>> Parsing HWInfo..."
                            def ver = sh(script: "curl -s 'https://sourceforge.net/projects/hwinfo/rss' | grep -o 'hwi64_[0-9]\\+' | head -n 1 | cut -d'_' -f2", returnStdout: true)
                            if (ver) {
                                urls.add("https://www.hwinfo.com/files/hwi64_${ver.trim()}.exe")
                                mode = "SMART"
                            }
                        }
                        // --- 8. QBittorrent (RSS) ---
                        else if (line.contains("qbittorrent.org")) {
                            echo ">>> Parsing QBittorrent..."
                            def qbit = sh(script: "curl -s 'https://sourceforge.net/projects/qbittorrent/rss?path=/qbittorrent-win32' | grep -o 'https://.*_x64_setup.exe/download' | head -n 1", returnStdout: true)
                            if (qbit) {
                                urls.add(qbit.trim())
                                mode = "SMART"
                            }
                        }
                        // --- Остальные (Hash Check) ---
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
                                fname = dUrl.split('/').last().split('\\?')[0]
                            }

                            // Заголовки (Критично для TPU)
                            def headers = "-A 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)'"
                            if (cleanUrl.contains("techpowerup.com")) {
                                headers += " -e 'https://www.techpowerup.com/'"
                            }

                            if (mode == "SMART") {
                                if (fileExists("${env.DL_DIR}/${fname}")) {
                                    echo "   [SKIP] ${fname} already exists."
                                } else {
                                    echo "   [DOWN] Downloading ${fname}..."
                                    sh "curl -L -s ${headers} -o '${env.DL_DIR}/${fname}' '${cleanUrl}'"
                                    
                                    def fSize = sh(script: "wc -c < '${env.DL_DIR}/${fname}'", returnStdout: true).trim()
                                    if (fSize == "0") {
                                        echo "   [FAIL] 0 bytes downloaded. Check URL/Protection."
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
                                sh "curl -L -s ${headers} -o '${env.DL_DIR}/${fname}.new' '${cleanUrl}'"
                                
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
