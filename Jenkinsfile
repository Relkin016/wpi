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

                        // 1. Получаем версию
                        def latestVersion = sh(script: "curl -s https://api.github.com/repos/${repo}/releases/latest | jq -r .tag_name", returnStdout: true).trim()

                        if (latestVersion == "null" || latestVersion == "") {
                            echo "Error checking ${repo}. Skipping."
                            return 
                        }

                        // 2. Читаем старую версию
                        def currentVersion = sh(script: "grep '^${repo}|' ${env.LOG_FILE} | cut -d '|' -f 2 || echo '0'", returnStdout: true).trim()

                        // 3. Если версии отличаются - качаем
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
                                sh "echo '${repo}|${latestVersion}|\\\$(date +%Y-%m-%d)' >> ${env.LOG_FILE}"
                                
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
                    def lines = readFile(env.WEB_LIST).readLines()
                    
                    lines.each { line ->
                        if (line.startsWith('#') || !line.trim()) return
                        
                        def urls = [] // Список для ссылок (если нужно качать x32 + x64)
                        def mode = "HASH" // По умолчанию сверяем хеш

                        // --- ПАРСЕРЫ ---
                        
                        if (line.contains("videolan.org")) {
                            echo ">>> Parsing VLC..."
                            // Берем сразу оба: x64 и x32
                            def vlc64 = sh(script: "curl -s https://download.videolan.org/pub/videolan/vlc/last/win64/ | grep -oP 'vlc-.*-win64\\.exe' | head -n 1", returnStdout: true).trim()
                            def vlc32 = sh(script: "curl -s https://download.videolan.org/pub/videolan/vlc/last/win32/ | grep -oP 'vlc-.*-win32\\.exe' | head -n 1", returnStdout: true).trim()
                            
                            if (vlc64) urls.add("https://download.videolan.org/pub/videolan/vlc/last/win64/${vlc64}")
                            if (vlc32) urls.add("https://download.videolan.org/pub/videolan/vlc/last/win32/${vlc32}")
                            mode = "SMART"
                        } 
                        else if (line.contains("telegram.org")) {
                            echo ">>> Parsing Telegram..."
                            def tg64 = sh(script: "curl -s -I 'https://telegram.org/dl/desktop/win64' | grep -i 'location:' | awk '{print \$2}' | tr -d '\r'", returnStdout: true).trim()
                            def tg32 = sh(script: "curl -s -I 'https://telegram.org/dl/desktop/win' | grep -i 'location:' | awk '{print \$2}' | tr -d '\r'", returnStdout: true).trim()
                            if (tg64) urls.add(tg64)
                            if (tg32) urls.add(tg32)
                            mode = "SMART"
                        }
                        else if (line.contains("epicgames.com")) {
                            echo ">>> Parsing Epic Games..."
                            def epic = sh(script: "curl -s -I 'https://launcher-public-service-prod06.ol.epicgames.com/launcher/api/installer/download/EpicGamesLauncherInstaller.msi' | grep -i 'location:' | awk '{print \$2}' | tr -d '\r'", returnStdout: true).trim()
                            if (epic) urls.add(epic)
                            mode = "SMART"
                        }
                        else if (line.contains("hwinfo.com")) {
                            echo ">>> Parsing HWInfo..."
                            def ver = sh(script: "curl -s 'https://sourceforge.net/projects/hwinfo/rss' | grep -o 'hwi64_[0-9]\\+' | head -n 1 | cut -d'_' -f2", returnStdout: true).trim()
                            if (ver) urls.add("https://www.hwinfo.com/files/hwi64_${ver}.exe")
                            mode = "SMART"
                        }
                        else if (line.contains("qbittorrent")) {
                            echo ">>> Parsing QBittorrent..."
                            def qbit = sh(script: "curl -s 'https://sourceforge.net/projects/qbittorrent/rss?path=/qbittorrent-win32' | grep -o 'https://.*_x64_setup.exe/download' | head -n 1", returnStdout: true).trim()
                            if (qbit) urls.add(qbit)
                            mode = "SMART"
                        }
                        else {
                            // Для Steam, Revo, Fraps просто берем ссылку из файла
                            urls.add(line.trim())
                            mode = "HASH"
                        }

                        // --- ОБРАБОТКА ЗАГРУЗКИ ---
                        
                        urls.each { dUrl ->
                            def fname = dUrl.split('/').last().split('\\?')[0]
                            
                            if (mode == "SMART") {
                                // Smart: проверка по имени файла
                                if (fileExists("${env.DL_DIR}/${fname}")) {
                                    echo "   [SKIP] ${fname} already exists."
                                } else {
                                    echo "   [DOWN] New version: ${fname}"
                                    sh "curl -L -s -o '${env.DL_DIR}/${fname}' '${dUrl}'"
                                    sh "echo 'WEB-SMART|${fname}|\\\$(date +%Y-%m-%d)' >> ${env.LOG_FILE}"
                                }
                            } 
                            else {
                                // Hash: качаем .new и сверяем
                                echo "   [HASH CHECK] ${fname}..."
                                sh "curl -L -s -o '${env.DL_DIR}/${fname}.new' '${dUrl}'"
                                
                                def newHash = sh(script: "sha256sum '${env.DL_DIR}/${fname}.new' | awk '{print \$1}'", returnStdout: true).trim()
                                def oldHash = sh(script: "grep '|${fname}|' ${env.LOG_FILE} | tail -n 1 | cut -d '|' -f 3 || echo 'none'", returnStdout: true).trim()
                                
                                if (newHash != oldHash) {
                                    echo "   [UPDATE] ${fname} changed!"
                                    sh "mv '${env.DL_DIR}/${fname}.new' '${env.DL_DIR}/${fname}'"
                                    sh "sed -i '\\#|${fname}|#d' ${env.LOG_FILE}"
                                    sh "echo 'WEB-HASH|${fname}|${newHash}|\\\$(date +%Y-%m-%d)' >> ${env.LOG_FILE}"
                                } else {
                                    echo "   [SKIP] ${fname} matches old hash."
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
