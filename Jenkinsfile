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
        // ------------------------------------------
        // 1. PREPARATION
        // ------------------------------------------
        stage('Preparation') {
            steps {
                script {
                    echo "--- [PREP] Preparing workspace ---"
                    sh "mkdir -p ${env.DL_DIR}"
                    sh "mkdir -p tmp"
                    
                    // Полная очистка перед загрузкой
                    sh "rm -rf ${env.DL_DIR}/*"
                    sh "rm -f tmp/*.json"
                    sh "rm -f tmp/*.check"
                    sh "rm -f ${env.QUEUE_FILE}"
                    sh "rm -f config.js"
                    
                    if (!fileExists(env.LOG_FILE)) {
                        sh "touch ${env.LOG_FILE}"
                    }
                }
            }
        }

        // ------------------------------------------
        // 2. CHECK VERSIONS (GATEKEEPER)
        // ------------------------------------------
        stage('Check Versions') {
            steps {
                echo "--- [CHECK] Scanning all sources ---"
                script {
                    def logLines = readFile(env.LOG_FILE).readLines()
                    def globalUpdateNeeded = false
                    def fullQueueList = [] 
                    def addedNames = []

                    // --- A: GITHUB ---
                    if (fileExists(env.GITHUB_LIST)) {
                        def ghLines = readFile(env.GITHUB_LIST).readLines()
                        ghLines.each { line ->
                            if (line.startsWith('#') || !line.trim()) return
                            try {
                                def parts = line.split(';')
                                def url = parts[0].trim(); def regex = parts[1].trim()
                                def repo = url.replace("https://github.com/", "").replace(/\/$/, "")

                                sh "curl -s -H User-Agent:${env.UA} https://api.github.com/repos/${repo}/releases/latest -o tmp/gh_${repo.replace('/', '_')}.json"
                                def jsonFile = "tmp/gh_${repo.replace('/', '_')}.json"
                                
                                if (readFile(jsonFile).contains("rate limit exceeded")) error "GITHUB LIMIT EXCEEDED"
                                def latestVer = sh(script: "jq -r .tag_name ${jsonFile}", returnStdout: true).trim()
                                if (!latestVer || latestVer == "null") return

                                def dUrl = sh(script: "jq -r --arg REGEX \"${regex}\" '.assets[] | select(.name | test(\$REGEX; \"i\")) | .browser_download_url' ${jsonFile} | head -n 1", returnStdout: true).trim()
                                
                                if (dUrl && dUrl != "null") {
                                    fullQueueList.add("GITHUB|${repo}|${latestVer}|${dUrl}")
                                    if (!logLines.any { it.contains("${repo}|${latestVer}|") }) {
                                        echo "   [UPDATE] GitHub: ${repo} (${latestVer})"
                                        globalUpdateNeeded = true
                                    }
                                }
                            } catch (e) { echo "   [ERR] ${line}: ${e.message}" }
                        }
                    }

                    // --- B: WEB ---
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
                                    if (fname == "client" || fname == "download" || addedNames.contains(fname)) return

                                    if (mode == "SMART") {
                                        fullQueueList.add("WEB|${fname}|SMART|${url}")
                                        if (!logLines.any { it.contains("|${fname}|") }) {
                                            echo "   [UPDATE] Web: ${fname}"
                                            globalUpdateNeeded = true
                                        }
                                    } else {
                                        echo "   [CHECK HASH] ${fname}..."
                                        sh "curl -s -L -A ${env.UA} -o 'tmp/${fname}.check' '${url}'"
                                        def newHash = sh(script: "sha256sum 'tmp/${fname}.check' | awk '{print \$1}'", returnStdout: true).trim()
                                        fullQueueList.add("HASH_MOVE|${fname}|${newHash}|tmp/${fname}.check")
                                        if (!logLines.any { it.contains("|${fname}|${newHash}|") }) {
                                            echo "   [UPDATE] Hash mismatch: ${fname}"
                                            globalUpdateNeeded = true
                                        }
                                    }
                                    addedNames.add(fname)
                                }
                            } catch (e) { echo "   [ERR] Web: ${e.message}" }
                        }
                    }

                    if (!globalUpdateNeeded) {
                        echo "\n=== SYSTEM UP TO DATE ==="; currentBuild.result = 'SUCCESS'
                    } else {
                        echo "\n=== UPDATES DETECTED! ==="
                        writeFile file: env.QUEUE_FILE, text: fullQueueList.join("\n") + "\n"
                    }
                }
            }
        }

        // ------------------------------------------
        // 3. DOWNLOAD STATIC FILES (EOL)
        // ------------------------------------------
        stage('Download Static (EOL)') {
            when { expression { fileExists(env.QUEUE_FILE) } }
            steps {
                script {
                    echo ">>> [DOWN] Downloading Split Static Archives..."
                    sh "mkdir -p tmp/static_temp"
                    
                    // Скачиваем 7 частей с GitHub (Raw ссылки)
                    // Используем main ветку
                    for (int i = 1; i <= 7; i++) {
                        sh "wget -qP tmp/static_temp https://github.com/Relkin016/file/raw/main/EOL.7z.00${i}"
                    }
                    
                    echo ">>> [EXTRACT] Unpacking 7z..."
                    // Распаковываем первый том, 7z подхватит остальные.
                    // -y : отвечать Да
                    // -o : папка назначения (слитно с флагом)
                    // ТРЕБУЕТСЯ: apt install p7zip-full
                    sh "7z x tmp/static_temp/EOL.7z.001 -o${env.DL_DIR} -y"
                    
                    // Чистим временные архивы
                    sh "rm -rf tmp/static_temp"
                    
                    // Проверка, что файлы появились (для лога)
                    sh "ls -lh ${env.DL_DIR} | grep -E 'DirectX|Unlocker|unchecky|Net'"
                }
            }
        }

        // ------------------------------------------
        // 4. DOWNLOAD GITHUB
        // ------------------------------------------
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

        // ------------------------------------------
        // 5. DOWNLOAD WEB
        // ------------------------------------------
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
                    sh "rm -f tmp/*.check tmp/sf_check.html"
                }
            }
        }

        // ------------------------------------------
        // 6. GENERATE CONFIG
        // ------------------------------------------
        stage('Generate Config') {
            when { expression { fileExists(env.QUEUE_FILE) } }
            steps {
                script {
                    def pythonScript = """
import os
import re
import datetime

DOWNLOADS_DIR = "tmp/downloads"
WPI_INSTALL_PATH = r"%wpipath%\\\\Install" 

APP_RULES = {
    # --- СТАТИЧЕСКИЕ (ИЗ АРХИВА) ---
    "unlocker":  {"name": "Unlocker 1.9.2",  "cat": "Системные", "uid": "UNLOCKER", "default": True,  "match": r"Unlocker_.*\\.exe"},
    "unchecky":  {"name": "Unchecky",        "cat": "Защита",    "uid": "UNCHECKY", "default": True,  "match": r"unchecky_setup\\.exe"},
    "directx":   {"name": "DirectX",         "cat": "Системные", "uid": "DIRECTX",  "default": True,  "match": r"DirectX.*\\.exe"},
    "dotnet":    {"name": ".NET Framework",  "cat": "Системные", "uid": "DOTNET",   "default": True,  "match": r"Netframework\\.exe"},

    # --- ДИНАМИЧЕСКИЕ ---
    "7zip":      {"name": "7-Zip",           "cat": "Системные", "uid": "7ZIP",      "default": False, "match": r"7z.*\\.exe"},
    "hwinfo":    {"name": "HWiNFO64",        "cat": "Системные", "uid": "HWINFO",    "default": True,  "match": r"hwi64.*\\.exe"},
    "cpu_fix":   {"name": "WuaCpuFix",       "cat": "Системные", "uid": "CPUFIX",    "default": True,  "match": r"WuaCpuFix.*\\.zip"},
    "hash_tab":  {"name": "OpenHashTab",     "cat": "Системные", "uid": "HASHTAB",   "default": True,  "match": r"OpenHashTab.*\\.msi"},
    "vlc":       {"name": "VLC Media Player","cat": "Медиа",     "uid": "VLC",       "default": True,  "match": r"vlc.*\\.exe"},
    "qpview":    {"name": "QuickPicViewer",  "cat": "Медиа",     "uid": "QPVIEW",    "default": False, "match": r"QuickPictureViewer.*\\.exe"},
    "qbittorrent":{"name": "qBittorrent",    "cat": "Интернет",  "uid": "QBIT",      "default": True,  "match": r"qbittorrent.*\\.exe"},
    "motrix":    {"name": "Motrix",          "cat": "Интернет",  "uid": "MOTRIX",    "default": False, "match": r"Motrix.*\\.exe"},
    "firefox":   {"name": "Firefox",         "cat": "Интернет",  "uid": "FIREFOX",   "default": True,  "match": r"Firefox.*\\.exe"},
    "openvpn":   {"name": "OpenVPN",         "cat": "Интернет",  "uid": "OPENVPN",   "default": False, "match": r"openvpn.*\\.msi"},
    "localsend": {"name": "LocalSend",       "cat": "Интернет",  "uid": "LOCALSEND", "default": False, "match": r"LocalSend.*\\.exe"},
    "rustdesk":  {"name": "RustDesk",        "cat": "Интернет",  "uid": "RUSTDESK",  "default": True,  "match": r"rustdesk.*\\.exe"},
    "steam":     {"name": "Steam",           "cat": "Для игр",   "uid": "STEAM",     "default": False, "match": r"SteamSetup\\.exe"},
    "epic":      {"name": "Epic Games",      "cat": "Для игр",   "uid": "EPIC",      "default": False, "match": r"EpicInstaller.*\\.msi"},
    "fraps":     {"name": "Fraps",           "cat": "Для игр",   "uid": "FRAPS",     "default": False, "match": r"fraps.*\\.exe"},
    "telegram":  {"name": "Telegram",        "cat": "Applications", "uid": "TELEGRAM", "default": True,  "match": r"tsetup.*\\.exe"},
    "notepad":   {"name": "Notepad++",       "cat": "Applications", "uid": "NOTEPAD",  "default": True,  "match": r"npp.*\\.exe"},
    "wincdemu":  {"name": "WinCDEmu",        "cat": "Iso",          "uid": "WINCDEMU", "default": False, "match": r"WinCDEmu.*\\.exe"},
}

HEADER = \"\"\"// WPI Config Generated by Jenkins
CheckOnLoad='default';
Configurations=['Silent','Default'];
ShowMultiDefault=true;
SortOrder=['Защита','Системные','Медиа','Iso','Интернет','Applications','Для игр'];
ConfigSortBy=2;
ConfigSortAscDes='asc';

pn=1;
\"\"\"

def generate():
    if not os.path.exists(DOWNLOADS_DIR):
        print("Error: Downloads dir not found")
        return

    files = sorted(os.listdir(DOWNLOADS_DIR))
    found_items = {}

    for f in files:
        matched = False
        for key, rules in APP_RULES.items():
            if re.match(rules["match"], f, re.IGNORECASE):
                if key not in found_items:
                    found_items[key] = {"files": []}
                
                arch = "{x86}"
                if any(x in f.lower() for x in ["64", "x64", "win64"]):
                    arch = "{x64}"
                elif any(x in f.lower() for x in ["x86", "win32"]):
                    arch = "{x86}"
                
                found_items[key]["files"].append( (arch, f) )
                matched = True
                break
        
        if not matched:
            print(f"SKIPPED: {f}")

    output = HEADER
    dq = chr(34)
    
    for key, data in found_items.items():
        rule = APP_RULES[key]
        cmd_list = []
        files_sorted = sorted(data["files"], key=lambda x: x[0]) 
        for arch, filename in files_sorted:
            path = f"{WPI_INSTALL_PATH}\\\\\\\\{filename}"
            cmd_str = f"'{arch} {dq}{path}{dq}'"
            cmd_list.append(cmd_str)
        cmds_js = ",".join(cmd_list)
        is_default = 'yes' if rule.get('default', False) else 'no'

        block = f\"\"\"
prog[pn]=['{rule['name']}'];
uid[pn]=['{rule['uid']}'];
dflt[pn]=['{is_default}'];
forc[pn]=['no'];
cat[pn]=['{rule['cat']}'];
pfro[pn]=['no'];
cmds[pn]=[{cmds_js}];
desc[pn]=['{rule['name']}'];
pn++;
\"\"\"
        output += block
    output += "\\n// End of generated config"

    with open("config.js", "w", encoding="utf-8") as f:
        f.write(output)
    print(output)

if __name__ == "__main__":
    generate()
"""
                    writeFile file: 'gen_config.py', text: pythonScript
                    sh "python3 gen_config.py"
                    archiveArtifacts artifacts: 'config.js', allowEmptyArchive: true
                }
            }
        }
    }
}
