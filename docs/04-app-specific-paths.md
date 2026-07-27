# 04 — Credenciais em Aplicações de Terceiros

## Clientes SSH/FTP

| Aplicação | Local das credenciais | Proteção |
|---|---|---|
| **WinSCP** | `HKCU\Software\Martin Prikryl\WinSCP 2\Sessions` | Ofuscação reversível (sem chave secreta real) |
| **FileZilla** | `%APPDATA%\FileZilla\sitemanager.xml`<br>`%APPDATA%\FileZilla\recentservers.xml` | **Texto claro ou Base64** |
| **mRemoteNG** | `%APPDATA%\mRemoteNG\confCons.xml` | AES com senha mestre (padrão fraco/conhecido em versões antigas) |
| **Xshell/Xftp** | `%USERPROFILE%\Documents\NetSarang Computer\` | Ofuscação própria |
| **MobaXterm** | Registro + `MobaXterm.ini` | AES com chave derivada de hostname+usuário |

## Tokens de sessão — mensageria e colaboração

Tokens de sessão valem tanto quanto senhas — e costumam ter **menos** proteção e revogação mais difícil. Quem possui o token assume a conta sem senha e, em geral, sem 2FA, até revogação manual.

### Discord

| Item | Caminho |
|---|---|
| Local Storage (token) | `%APPDATA%\discord\Local Storage\leveldb\*.ldb` e `*.log` |
| Chave de criptografia | `%APPDATA%\discord\Local State` → campo `os_crypt.encrypted_key` |
| Discord Canary | `%APPDATA%\discordcanary\Local Storage\leveldb\` |
| Discord PTB | `%APPDATA%\discordptb\Local Storage\leveldb\` |
| Discord (via navegador) | `Local Storage\leveldb\` do perfil do navegador usado |

**Formato do token**

- **Token de usuário**: regex `[a-zA-Z0-9_-]{24}\.[a-zA-Z0-9_-]{6}\.[a-zA-Z0-9_-]{27,}` (Base64 de `user_id.timestamp.hmac`)
- **Token com MFA**: prefixo `mfa.` seguido de ~84 caracteres
- Nos arquivos `.ldb`/`.log`, aparece como valor da chave `token` no LevelDB

**Proteção (mudança importante de 2022)**

| Versão | Proteção |
|---|---|
| Antes de ~ago/2022 | **Texto claro** nos arquivos LevelDB — leitura direta |
| Após ~ago/2022 | Token criptografado via **Electron safeStorage** (AES-GCM com chave DPAPI em `Local State`) — mesmo modelo dos navegadores Chromium |

Como a chave DPAPI é do **próprio usuário**, qualquer processo rodando como o usuário pode decriptar — a criptografia protege apenas contra acesso offline/outro usuário.

**Relevância**: 🔴 T1528 — vetor massivo de phishing e account takeover desde 2021. 🔵 monitorar leituras de `leveldb\` (Sysmon EID 11), exfiltração de `.ldb`, logons de IPs impossíveis.

### Slack

| Item | Caminho |
|---|---|
| Cookies e Local Storage | `%APPDATA%\Slack\Local Storage\leveldb\`<br>`%APPDATA%\Slack\Network\Cookies` |
| Chave de criptografia | `%APPDATA%\Slack\Local State` → `os_crypt.encrypted_key` (safeStorage/DPAPI) |
| Cache de workspaces | `%APPDATA%\Slack\storage\slack-*` |

- O token de sessão (`xoxc-`) + cookie `d` (`xoxd-`) juntos autenticam o workspace inteiro.
- 🔴 T1528/T1539 — roubo de sessão Slack dá acesso a canais, DMs e arquivos corporativos.

### Microsoft Teams

| Item | Caminho |
|---|---|
| Teams clássico | `%APPDATA%\Microsoft\Teams\` (`Local Storage`, `Cookies`, `desktop-config.json`) |
| New Teams (MSIX) | `%LOCALAPPDATA%\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\` |
| Credenciais de conta | Credential Manager (`MicrosoftAccount:*`, `MSO*`) + cache MSAL |

- Tokens de acesso/refresh MSAL no cache — reutilizáveis até expiração/revogação condicional.
- 🔴 session hijacking de Teams é vetor ativo de phishing corporativo (mensagens internas têm confiança implícita).

### Telegram Desktop

| Item | Caminho |
|---|---|
| Sessão | `%APPDATA%\Telegram Desktop\tdata\` (`D877F783D5D3EF8C*` e `map*` / `key_datas`) |
| Instalação portátil | `<pasta do Telegram>\tdata\` |

- A pasta `tdata` inteira **é** a sessão: copiá-la para outra máquina (com o mesmo nome de pasta) assume a conta, sem 2FA local.
- 🔴 T1528 — exfiltração de `tdata` é alvo padrão de infostealers; 🔵 revogar sessões ativas em *Configurações → Dispositivos* em IR.

### WhatsApp Desktop

| Item | Caminho |
|---|---|
| WhatsApp (Store/MSIX) | `%LOCALAPPDATA%\Packages\5319275A.WhatsAppDesktop_*\LocalState\` |
| WhatsApp (instalador direto) | `%APPDATA%\WhatsApp\Local Storage\leveldb\` |
| Sessão web | cookies/Local Storage do navegador |

- 🔵 sessões vinculadas são revogáveis em *Aparelhos conectados* no celular — incluir no playbook de IR.

## Tokens de sessão — jogos e mídia

| Plataforma | Caminho | Proteção |
|---|---|---|
| **Steam** | `C:\Program Files (x86)\Steam\config\loginusers.vdf`<br>`C:\Program Files (x86)\Steam\ssfn*` | `ssfn` (Steam Guard file) + token = login sem código de e-mail |
| **Epic Games** | `%LOCALAPPDATA%\EpicGamesLauncher\Saved\Config\Windows\` e `Saved\webcache*\` | Tokens OAuth no webcache |
| **Spotify** | `%APPDATA%\Spotify\` (Local Storage) | safeStorage/DPAPI |
| **Riot Games** | `%LOCALAPPDATA%\Riot Games\` | Tokens de sessão no client |

## Clientes de e-mail

### Microsoft Outlook (clássico)

| Item | Caminho | Conteúdo |
|---|---|---|
| Perfis | `HKCU\Software\Microsoft\Office\16.0\Outlook\Profiles\` | Contas configuradas, servidores |
| Credenciais de conta | Credential Manager — `MS.Outlook:*`, `MicrosoftOffice16_Data:ADAL:*` | Senhas IMAP/SMTP/POP e tokens OAuth (DPAPI) |
| Cache de caixa | `%LOCALAPPDATA%\Microsoft\Outlook\*.ost` | **Conteúdo integral dos e-mails** — legível com ferramentas forenses mesmo sem senha |
| PST arquivados | onde o usuário salvou (comum: `Documents\Outlook Files\`) | Mesmo conteúdo, proteção de senha PST é fraca/reversível |

### New Outlook (Monarch)

| Item | Caminho |
|---|---|
| Dados e cache | `%LOCALAPPDATA%\Microsoft\Olk\` |
| Tokens | Cache MSAL (`%LOCALAPPDATA%\Microsoft\IdentityCache\`) + WebView2 em `%LOCALAPPDATA%\Microsoft\Olk\EBWebView\` |

### Thunderbird

- `%APPDATA%\Thunderbird\Profiles\<perfil>\logins.json` + `key4.db` — **senhas de e-mail** no formato NSS (ver seção Gecko). Sem senha mestre = decriptável offline.
- Caixa local: `Mail\` e `ImapMail\` no perfil (mbox/maildir em texto claro).

### eM Client

| Item | Caminho | Proteção |
|---|---|---|
| Banco de dados | `%APPDATA%\eM Client\` (`accounts.dat`, `mail_data.dat`) | Senhas criptografadas por máquina (DPAPI-like) |
| Backups | `%USERPROFILE%\Documents\eM Client\` | Arquivos `.zip` de backup com a base completa |

### Mailbird

- `%APPDATA%\Mailbird\` (SQLite + Local Storage) — tokens OAuth e credenciais IMAP/SMTP.

### Relevância

- 🔴 T1114 (Email Collection): OST/PST contêm a **caixa inteira** — ideal para coleta silenciosa e busca por "senha", "password", "credentials" no histórico de e-mails.
- 🔴 Credenciais IMAP/SMTP em `MS.Outlook:*` permitem acesso persistente ao mailbox sem interação com MFA moderno (protocolos legados).
- 🔵 Desabilitar protocolos legados (IMAP/POP basic auth), monitorar acesso a `.ost`/`.pst` por processos estranhos, e lembrar que trocar a senha **não** invalida OST já exfiltrado.

## IDEs e editores

### VS Code

| Item | Caminho | Conteúdo |
|---|---|---|
| Estado global (SQLite) | `%APPDATA%\Code\User\globalStorage\state.vscdb` | Dados de extensões, alguns tokens em claro |
| Secrets de extensões | Credential Manager — alvos `vscode*` (ex.: `vscode.github-authentication`) | Tokens OAuth de GitHub, Azure, Remote |
| Chave safeStorage | `%APPDATA%\Code\Local State` → `os_crypt.encrypted_key` | DPAPI (mesmo modelo Chromium) |
| Settings sync | `%APPDATA%\Code\User\sync\` | Configurações sincronizadas |
| Storage por extensão | `%APPDATA%\Code\User\globalStorage\<extensão>\` | Alguns extensões salvam tokens em arquivos próprios |
| Variantes | `%APPDATA%\Code - Insiders\`, `%APPDATA%\VSCodium\` | Mesma estrutura |

- **Remote-SSH**: reutiliza `%USERPROFILE%\.ssh\` (doc 03) — chaves e config.
- 🔴 T1552.001 — tokens de extensões dão acesso a repositórios e cloud do dev; 🔵 tratar `state.vscdb` como arquivo sensível em EDR/DLP.

### JetBrains (IntelliJ, PyCharm, WebStorm, DataGrip...)

| Item | Caminho | Conteúdo |
|---|---|---|
| Config por IDE | `%APPDATA%\JetBrains\<IDE><versão>\` | `options\`, plugins, datasources |
| Password Safe (padrão) | **Credential Manager** (native keychain) | Senhas de DB, Git, deployment |
| Password Safe (modo KeePass) | Arquivo `.kdbx` no diretório de config da IDE | Protegido por senha mestre |
| Configuração do safe | `%APPDATA%\JetBrains\<IDE><versão>\options\security.xml` | Define o backend usado |
| DataSources | `%APPDATA%\JetBrains\<IDE><versão>\dataSources\` | Conexões de banco (usuários/hosts) |

### Visual Studio

| Item | Caminho |
|---|---|
| Credenciais de conta | Credential Manager (`MicrosoftAccount`, `VS:*`) |
| Config | `%LOCALAPPDATA%\Microsoft\VisualStudio\<versão>\` |
| Publish profiles | `Properties\PublishProfiles\*.pubxml` nos projetos — podem conter senha de deploy em claro |

## Histórico de shell e artefatos de linha de comando

| Item | Caminho | Por que importa |
|---|---|---|
| Histórico do PowerShell | `%APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt` | **Senhas digitadas em comandos** (`net use`, `ssh`, `mysql -p`, connection strings) ficam em texto claro — achado clássico em DFIR e red team |
| Histórico do cmd | `doskey /history` (memória) + setas de comando | Sessão atual apenas |
| `cmdkey /list` | Credential Manager | Enumera credenciais salvas sem elevação |
| Transcrições | `Start-Transcript` (onde o usuário salvou) | Logs completos de sessão |

## Clientes de banco de dados

| Aplicação | Caminho | Proteção |
|---|---|---|
| **DBeaver** | `%APPDATA%\DBeaverData\workspace6\General\.dbeaver\credentials-config.json` + `data-sources.json` | AES com chave padrão conhecida (sem senha mestre) |
| **SSMS** | `%APPDATA%\Microsoft\SQL Server Management Studio\<versão>\SqlStudio.bin` | DPAPI do usuário |
| **HeidiSQL** | `HKCU\Software\HeidiSQL\Servers\<nome>` | Ofuscação reversível |
| **pgAdmin** | `%APPDATA%\pgAdmin\pgadmin4.db` + `%USERPROFILE%\.pgpass` | `.pgpass` em **texto claro** |
| **MySQL Workbench** | `%APPDATA%\MySQL\Workbench\workbench_user_data.dat` | Vault próprio (DPAPI) |
| **Azure Data Studio** | `%APPDATA%\azuredatastudio\` + Credential Manager | safeStorage/DPAPI |

## Arquivos de configuração esquecidos

| Arquivo | Local típico | Conteúdo |
|---|---|---|
| `unattend.xml` / `autounattend.xml` | `C:\Windows\Panther\`<br>`C:\Windows\System32\sysprep\` | **Senha de admin local em texto claro** de imagens de deploy — remover após sysprep! |
| `web.config` | pastas de sites IIS (`C:\inetpub\wwwroot\`) | Connection strings com usuário/senha de SQL |
| `.env` | pastas de projetos dev | Secrets de API, DB e cloud em texto claro — frequentemente commitados por acidente |
| `appsettings.json` | projetos .NET | Connection strings e chaves de API |
| `php.ini` / `wp-config.php` | servidores web | Credenciais de banco |

## Navegadores

### Família Chromium

Todos os Chromium-based seguem o mesmo modelo de armazenamento:

- **Logins**: `<User Data>\Default\Login Data` (SQLite, senhas AES-GCM)
- **Cookies/sessões**: `<User Data>\Default\Network\Cookies` (SQLite)
- **Chave de criptografia**: `<User Data>\Local State` → campo `os_crypt.encrypted_key` (DPAPI)
- Chrome/Edge/Brave 127+ adicionam **App-Bound Encryption** (vincula a chave ao binário legítimo)

| # | Navegador | Caminho do perfil (`User Data`) |
|---|---|---|
| 1 | Google Chrome | `%LOCALAPPDATA%\Google\Chrome\User Data\` |
| 2 | Microsoft Edge | `%LOCALAPPDATA%\Microsoft\Edge\User Data\` |
| 3 | Brave | `%LOCALAPPDATA%\BraveSoftware\Brave-Browser\User Data\` |
| 4 | Opera | `%APPDATA%\Opera Software\Opera Stable\` *(Roaming, sem subpasta User Data)* |
| 5 | Opera GX | `%APPDATA%\Opera Software\Opera GX Stable\` |
| 6 | Vivaldi | `%LOCALAPPDATA%\Vivaldi\User Data\` |
| 7 | Arc | `%LOCALAPPDATA%\Arc\User Data\` |
| 8 | Yandex Browser | `%LOCALAPPDATA%\Yandex\YandexBrowser\User Data\` |
| 9 | Epic Privacy Browser | `%LOCALAPPDATA%\Epic Privacy Browser\User Data\` |
| 10 | Slimjet | `%LOCALAPPDATA%\Slimjet\User Data\` |
| 11 | Torch | `%LOCALAPPDATA%\Torch\User Data\` |
| 12 | Naver Whale | `%LOCALAPPDATA%\Naver\Naver Whale\User Data\` |
| 13 | Coc Coc | `%LOCALAPPDATA%\CocCoc\Browser\User Data\` |
| 14 | Avast Secure Browser | `%LOCALAPPDATA%\AVAST Software\Browser\User Data\` |
| 15 | AVG Secure Browser | `%LOCALAPPDATA%\AVG\Browser\User Data\` |
| 16 | 360 Secure Browser (Qihoo) | `%LOCALAPPDATA%\360Chrome\Chrome\User Data\` |
| 17 | Cent Browser | `%LOCALAPPDATA%\CentBrowser\User Data\` |
| 18 | UC Browser | `%LOCALAPPDATA%\UCBrowser\` |
| 19 | Maxthon (v6+) | `%APPDATA%\Maxthon6\` |
| 20 | Chromium (open-source) | `%LOCALAPPDATA%\Chromium\User Data\` |

> **Nota**: perfis adicionais ficam em `Profile 1`, `Profile 2`, etc. (em vez de `Default`) — ferramentas de extração devem iterar todos. Opera/Opera GX são exceção: usam `%APPDATA%` diretamente.

### Família Gecko (Firefox e forks)

Modelo comum: `logins.json` (credenciais criptografadas) + `key4.db` (chave NSS) + `cookies.sqlite`. **Sem senha mestre, tudo é decriptável offline.**

| # | Navegador | Caminho do perfil |
|---|---|---|
| 1 | Mozilla Firefox | `%APPDATA%\Mozilla\Firefox\Profiles\<perfil>\` |
| 2 | Tor Browser | `<pasta de instalação>\Browser\TorBrowser\Data\Browser\profile.default\` *(portátil, geralmente em Desktop)* |
| 3 | Waterfox | `%APPDATA%\Waterfox\Profiles\<perfil>\` |
| 4 | LibreWolf | `%APPDATA%\LibreWolf\Profiles\<perfil>\` |
| 5 | Pale Moon | `%APPDATA%\Moonchild Productions\Pale Moon\Profiles\<perfil>\` |
| 6 | SeaMonkey | `%APPDATA%\Mozilla\SeaMonkey\Profiles\<perfil>\` |
| 7 | Thunderbird (e-mail) | `%APPDATA%\Thunderbird\Profiles\<perfil>\` — **senhas de e-mail** no mesmo formato |

### Legado

| Navegador | Local | Proteção |
|---|---|---|
| Internet Explorer / Edge Legacy | Windows Vault (`%APPDATA%\Microsoft\Vault\`) | DPAPI + Vault schema (ver doc 01) |

## Git e ferramentas de dev

| Item | Caminho |
|---|---|
| Git Credential Manager | Credential Manager do Windows (DPAPI) |
| GitHub CLI (`gh`) | Credential Manager + `%APPDATA%\GitHub CLI\hosts.yml` |
| Credenciais em texto claro | `%USERPROFILE%\.git-credentials` (quando `credential.helper store`) |
| Config global | `%USERPROFILE%\.gitconfig` |
| `.netrc` | `%USERPROFILE%\_netrc` |
| NuGet API keys | `%APPDATA%\NuGet\NuGet.Config` |
| npm tokens | `%USERPROFILE%\.npmrc` |
| pip | `%APPDATA%\pip\pip.ini` |
| AWS CLI | `%USERPROFILE%\.aws\credentials` |
| Azure CLI | `%USERPROFILE%\.azure\` (tokens DPAPI + MSAL cache) |
| Docker | `%USERPROFILE%\.docker\config.json` (auth em Base64 se sem cred helper) |
| kubeconfig | `%USERPROFILE%\.kube\config` (certificados/tokens de cluster) |
| Terraform | `%USERPROFILE%\.terraform.d\credentials.tfrc.json` |
| Helm | `%APPDATA%\helm\registry\config.json` |

## Gerenciadores de senha

| App | Local típico do vault |
|---|---|
| KeePass | arquivo `.kdbx` escolhido pelo usuário — comuns: `Documents`, OneDrive/Dropbox sync |
| Bitwarden (desktop) | `%APPDATA%\Bitwarden\data.json` (criptografado, chave derivada do login) |
| 1Password | `%LOCALAPPDATA%\1Password\` |

## Implicações de segurança

- `sitemanager.xml`, `_netrc`, `.git-credentials`, `.pgpass`, `.env` e `unattend.xml` são os achados mais fáceis: **Base64 não é criptografia** e texto claro não perdoa.
- O `ConsoleHost_history.txt` do PowerShell é uma mina de senhas digitadas — auditar em IR e conscientizar devs a nunca passar senha como argumento.
- IDEs concentram tokens de GitHub, Azure e bancos de produção — uma estação de dev comprometida vale por dez.
- OST/PST contêm a caixa de e-mail inteira — trocar a senha não invalida o que já foi exfiltrado.
- Infostealers (RedLine, Vidar, Lumma, Raccoon) têm listas hardcoded cobrindo praticamente **todos** os caminhos deste documento — inclusive forks obscuros de navegador.
- Reconhecimento de atacantes (T1552/T1555/T1539/T1528/T1114) enumera exatamente essas pastas — ótimo material para **canary files** e regras de detecção.
- Política: proibir `credential.helper store`, exigir senha mestre nos Gecko-based, remover `unattend.xml` pós-deploy, desabilitar basic auth IMAP/POP, preferir gerenciadores de senha corporativos a cofres de navegador.
