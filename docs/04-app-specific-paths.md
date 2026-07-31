# 04 — Credenciais em Aplicações de Terceiros

## Clientes SSH/FTP

| Aplicação | Local das credenciais | Proteção |
|---|---|---|
| **WinSCP** | `HKCU\Software\Martin Prikryl\WinSCP 2\Sessions` | Ofuscação reversível, sem chave secreta real |
| **FileZilla** | `%APPDATA%\FileZilla\sitemanager.xml`<br>`%APPDATA%\FileZilla\recentservers.xml` | Texto claro ou Base64 |
| **mRemoteNG** | `%APPDATA%\mRemoteNG\confCons.xml` | AES com senha mestre; versões antigas usam padrão fraco e conhecido |
| **Xshell/Xftp** | `%USERPROFILE%\Documents\NetSarang Computer\` | Ofuscação própria |
| **MobaXterm** | Registro + `MobaXterm.ini` | AES com chave derivada de hostname e usuário |

## Tokens de sessão: mensageria e colaboração

Token de sessão vale tanto quanto senha, e costuma ter menos proteção e revogação mais chata. Quem tem o token assume a conta sem senha e, em geral, sem 2FA, até que alguém revogue manualmente.

### Discord

| Item | Caminho |
|---|---|
| Local Storage (token) | `%APPDATA%\discord\Local Storage\leveldb\*.ldb` e `*.log` |
| Chave de criptografia | `%APPDATA%\discord\Local State`, campo `os_crypt.encrypted_key` |
| Discord Canary | `%APPDATA%\discordcanary\Local Storage\leveldb\` |
| Discord PTB | `%APPDATA%\discordptb\Local Storage\leveldb\` |
| Discord via navegador | `Local Storage\leveldb\` do perfil do navegador |

O token de usuário segue o padrão `[a-zA-Z0-9_-]{24}\.[a-zA-Z0-9_-]{6}\.[a-zA-Z0-9_-]{27,}` (Base64 de `user_id.timestamp.hmac`). Com MFA, ganha o prefixo `mfa.` e cerca de 84 caracteres. Nos arquivos LevelDB aparece como valor da chave `token`.

Sobre proteção, houve uma virada em 2022. Antes, o token ficava em texto claro no LevelDB, leitura direta. Depois, o Discord passou a usar a safeStorage do Electron, AES-GCM com chave DPAPI no `Local State`, o mesmo modelo dos navegadores Chromium. Como a chave DPAPI é do próprio usuário, qualquer processo rodando como ele decripta. A criptografia só protege contra acesso offline ou de outro usuário.

Relevância: do lado ofensivo, T1528, vetor massivo de phishing e account takeover desde 2021. Do lado defensivo, monitorar leituras do `leveldb\` (Sysmon EID 11), exfiltração de `.ldb` e logons vindos de IPs impossíveis.

### Slack

| Item | Caminho |
|---|---|
| Cookies e Local Storage | `%APPDATA%\Slack\Local Storage\leveldb\`<br>`%APPDATA%\Slack\Network\Cookies` |
| Chave de criptografia | `%APPDATA%\Slack\Local State`, `os_crypt.encrypted_key` (safeStorage/DPAPI) |
| Cache de workspaces | `%APPDATA%\Slack\storage\slack-*` |

O token `xoxc-` somado ao cookie `d` (`xoxd-`) autentica o workspace inteiro. Roubo de sessão de Slack abre canais, DMs e arquivos corporativos (T1528/T1539).

### Microsoft Teams

| Item | Caminho |
|---|---|
| Teams clássico | `%APPDATA%\Microsoft\Teams\` (`Local Storage`, `Cookies`, `desktop-config.json`) |
| New Teams (MSIX) | `%LOCALAPPDATA%\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\` |
| Credenciais de conta | Credential Manager (`MicrosoftAccount:*`, `MSO*`) e cache MSAL |

Os tokens de acesso e refresh ficam no cache MSAL e valem até expirar ou serem revogados. Session hijacking de Teams é vetor ativo de phishing corporativo, porque mensagem interna carrega confiança automática.

### Telegram Desktop

| Item | Caminho |
|---|---|
| Sessão | `%APPDATA%\Telegram Desktop\tdata\` |
| Instalação portátil | `<pasta do Telegram>\tdata\` |

A pasta `tdata` inteira é a sessão. Copiá-la para outra máquina assume a conta, sem 2FA local. É alvo padrão de infostealer (T1528). Na resposta a incidente, revogar sessões em Configurações, Dispositivos.

### WhatsApp Desktop

| Item | Caminho |
|---|---|
| Versão da Store (MSIX) | `%LOCALAPPDATA%\Packages\5319275A.WhatsAppDesktop_*\LocalState\` |
| Instalador direto | `%APPDATA%\WhatsApp\Local Storage\leveldb\` |
| Sessão web | Cookies e Local Storage do navegador |

Sessões vinculadas são revogadas em Aparelhos conectados, no celular. Vale incluir isso no playbook de IR.

## Tokens de sessão: jogos e mídia

| Plataforma | Caminho | Proteção |
|---|---|---|
| **Steam** | `C:\Program Files (x86)\Steam\config\loginusers.vdf`<br>`C:\Program Files (x86)\Steam\ssfn*` | `ssfn` mais token permite login sem código de e-mail |
| **Epic Games** | `%LOCALAPPDATA%\EpicGamesLauncher\Saved\` | Tokens OAuth no webcache |
| **Spotify** | `%APPDATA%\Spotify\` | safeStorage/DPAPI |
| **Riot Games** | `%LOCALAPPDATA%\Riot Games\` | Tokens de sessão no client |

## Clientes de e-mail

### Microsoft Outlook (clássico)

| Item | Caminho | Conteúdo |
|---|---|---|
| Perfis | `HKCU\Software\Microsoft\Office\16.0\Outlook\Profiles\` | Contas e servidores configurados |
| Credenciais | Credential Manager, alvos `MS.Outlook:*` e `MicrosoftOffice16_Data:*` | Senhas IMAP/SMTP/POP e tokens OAuth (DPAPI) |
| Cache da caixa | `%LOCALAPPDATA%\Microsoft\Outlook\*.ost` | Conteúdo integral dos e-mails |
| PST arquivado | Onde o usuário salvou, geralmente `Documents\Outlook Files\` | Mesma coisa; a senha de PST é fraca e reversível |

### New Outlook (Monarch)

| Item | Caminho |
|---|---|
| Dados e cache | `%LOCALAPPDATA%\Microsoft\Olk\` |
| Tokens | Cache MSAL (`%LOCALAPPDATA%\Microsoft\IdentityCache\`) e WebView2 em `%LOCALAPPDATA%\Microsoft\Olk\EBWebView\` |

### Thunderbird

Senhas de e-mail em `%APPDATA%\Thunderbird\Profiles\<perfil>\logins.json` com `key4.db`, formato NSS (ver seção Gecko). Sem senha mestre, decriptável offline. A caixa local fica em `Mail\` e `ImapMail\`, em texto claro.

### eM Client

| Item | Caminho | Proteção |
|---|---|---|
| Banco de dados | `%APPDATA%\eM Client\` (`accounts.dat`, `mail_data.dat`) | Criptografia por máquina |
| Backups | `%USERPROFILE%\Documents\eM Client\` | Arquivos `.zip` com a base completa |

### Mailbird

`%APPDATA%\Mailbird\`, com SQLite e Local Storage guardando tokens OAuth e credenciais IMAP/SMTP.

### Relevância

OST e PST contêm a caixa inteira, o que os torna alvo perfeito de T1114 (Email Collection): dá para vasculhar o histórico procurando "senha" e "credentials" sem tocar no servidor. Credenciais IMAP/SMTP em `MS.Outlook:*` permitem acesso persistente via protocolo legado, sem esbarrar em MFA moderno. E um ponto que muita gente esquece em IR: trocar a senha não invalida OST já exfiltrado. Do lado defensivo, desabilitar basic auth em IMAP/POP e monitorar acesso a `.ost` e `.pst` por processos estranhos.

## IDEs e editores

### VS Code

| Item | Caminho | Conteúdo |
|---|---|---|
| Estado global (SQLite) | `%APPDATA%\Code\User\globalStorage\state.vscdb` | Dados de extensões, alguns tokens em claro |
| Secrets de extensões | Credential Manager, alvos `vscode*` | Tokens OAuth de GitHub, Azure, Remote |
| Chave safeStorage | `%APPDATA%\Code\Local State` | DPAPI, mesmo modelo Chromium |
| Settings sync | `%APPDATA%\Code\User\sync\` | Configurações sincronizadas |
| Variantes | `%APPDATA%\Code - Insiders\`, `%APPDATA%\VSCodium\` | Mesma estrutura |

O Remote-SSH reutiliza o `%USERPROFILE%\.ssh\` do doc 03. Tokens de extensão dão acesso a repositórios e cloud do desenvolvedor (T1552.001); trate o `state.vscdb` como arquivo sensível.

### JetBrains (IntelliJ, PyCharm, WebStorm, DataGrip)

| Item | Caminho | Conteúdo |
|---|---|---|
| Config por IDE | `%APPDATA%\JetBrains\<IDE><versão>\` | `options\`, plugins, datasources |
| Password Safe (padrão) | Credential Manager (native keychain) | Senhas de banco, Git, deploy |
| Password Safe (modo KeePass) | Arquivo `.kdbx` no diretório de config | Senha mestre |
| Configuração do safe | `options\security.xml` | Define o backend |
| DataSources | `dataSources\` | Conexões de banco |

### Visual Studio

| Item | Caminho |
|---|---|
| Credenciais de conta | Credential Manager (`MicrosoftAccount`, `VS:*`) |
| Config | `%LOCALAPPDATA%\Microsoft\VisualStudio\<versão>\` |
| Publish profiles | `Properties\PublishProfiles\*.pubxml` nos projetos, com senha de deploy em texto claro |

## Ferramentas de schematics e suporte técnico

### Borneo Schematic

| Item | Caminho | Conteúdo |
|---|---|---|
| Cache da aplicação | `%LOCALAPPDATA%\Borneo-App-Cache` | Cache local do app (equivale a `%AppData%\Local\Borneo-App-Cache`), podendo conter tokens de sessão, cookies de autenticação e dados de login da conta Borneo |

### ZXW Tool (ZXWTeam)

| Item | Caminho | Conteúdo |
|---|---|---|
| Config e conta | `%APPDATA%\ZXW*` e `HKCU\Software\ZXWTeam` | Dados de licenciamento (dongle/conta), configurações e possível cache de login |
| Cache | `%LOCALAPPDATA%\ZXW*` | Cache de schematics baixados e dados de sessão |

### Refox (RE-FOX)

| Item | Caminho | Conteúdo |
|---|---|---|
| Dados da aplicação | `%APPDATA%\refox` (padrão Electron) | `Cookies`, `Local Storage\leveldb` e `Session Storage` com tokens de sessão da conta |
| Cache | `%APPDATA%\refox\Cache` | Schematics e bitmaps em cache |

### Padrão geral (apps de reparo baseadas em Electron)

A maioria das ferramentas modernas de schematics/bitmap são apps Electron. O padrão de armazenamento é previsível: `%APPDATA%\<nome-do-app>\` com `Cookies`, `Local Storage`, `Session Storage` e `IndexedDB` — exatamente os mesmos artefatos de sessão que um navegador Chromium manteria (ver seção Navegadores). Se o app usa login web embarcado, as credenciais de sessão vivem aí.

Ferramentas de schematics para reparo (Borneo, ZXW, Refox e similares) costumam manter sessões longas e armazenar tokens de licença ou credenciais localmente. Em DFIR e red team, o cache dessas apps revela contas ativas e dados de licenciamento. Vale verificar também o `%APPDATA%` (Roaming) da mesma aplicação para arquivos de configuração. Os caminhos exatos variam por versão — confirmar a pasta real instalada antes da coleta.

## Virtualização

### VMware Workstation — credenciais de servidor vSphere/ESXi

Quando o usuário salva a conexão com um servidor ESXi/vSphere ("remember me"), a Workstation grava user:pass localmente em dois arquivos:

| Item | Caminho | Conteúdo |
|---|---|---|
| Config privada | `%APPDATA%\VMware\preferences-private.ini` | `encryption.userKey`, `encryption.keySafe`, `encryption.data` — hostname, usuário e senha (em camadas de criptografia) |
| Dados ACE | `%APPDATA%\VMware\ace.dat` | Segunda camada da cadeia: KDF e chave final da senha |

**Fluxo de decriptação documentado** (pesquisa da XM Cyber, 2022 — [fonte](https://xmcyber.com/blog/decrypting-vmware-workstation-passwords-for-fun/)):

1. `encryption.userKey` → DPAPI do usuário atual → revela o algoritmo (AES-256) e a **KEY_1**
2. `encryption.keySafe` → primeiros 16 bytes são o IV; AES-256 com KEY_1 → **KEY_2**
3. `encryption.data` → IV nos primeiros 16 bytes; AES-256 com KEY_2 → configuração em texto claro (hostname, usuário e a senha ainda cifrada)
4. `ace.dat` (DATA_1) → chave **hardcoded** na `vmwarebase.dll` → revela o KDF (PBKDF2-HMAC-SHA1), o salt e DATA_2
5. **KEY_3** = PBKDF2(segredo hardcoded na DLL, salt) → decripta DATA_2 → **KEY_4**
6. Senha → IV = primeiros 16 bytes do campo cifrado; AES com KEY_4 → **texto claro**

Pontos-chave: a proteção real é só a DPAPI do usuário (passo 1); todo o resto da cadeia usa segredos hardcoded na `vmwarebase.dll`, reversíveis offline por qualquer processo no contexto do usuário. Ferramentas prontas: [XMCyber/VmwarePasswordDecryptor](https://github.com/XMCyber/VmwarePasswordDecryptor) e `diana-workstationdec.py` (tijldeneut/diana).

### VMware Workstation — senha de criptografia de VMs

Para VMs criptografadas com "remember password", a senha vai para o **Windows Credential Manager**, com o nome do alvo igual ao `encryptedVM.guid` do `.vmx` da VM. Recuperação trivial no contexto do usuário via `CredReadW` — sem elevação ([PoC testada no 17.5](https://gist.github.com/andshrew/bf6e5e8fa09b957caffc09c6dee58472)).

| Item | Caminho |
|---|---|
| GUID da credencial | `encryptedVM.guid` dentro do `.vmx` |
| Senha | Credential Manager, alvo = o GUID acima |

### VirtualBox

| Item | Caminho | Conteúdo |
|---|---|---|
| Config global | `%USERPROFILE%\.VirtualBox\VirtualBox.xml` | Configurações globais, lista de VMs, definições de VRDP e NAT |
| KeyStore da VM criptografada | `%USERPROFILE%\VirtualBox VMs\<vm>\<vm>.vbox` (campo `KeyStore`) | Parâmetros completos da criptografia do disco: algoritmo, KDF, salts e hash final |

**Criptografia de disco (documentada):** a criptografia de VMs do VirtualBox usa AES-XTS256-PLAIN64 com derivação PBKDF2-SHA256 em esquema de **duplo salt** (PBKDF2_1 com ~100k iterações, PBKDF2_2 com ~20k, hash final verificável offline — ver [dissecação no fórum do hashcat](https://hashcat.net/forum/thread-6506.html)). O KeyStore no `.vbox` contém tudo que é preciso para cracking offline da senha do disco: ferramentas como vboxdie-cracker demonstram o fluxo. Não é recuperável sem força bruta (diferente do caso VMware acima, que tem chaves hardcoded), mas é inteiramente atacável offline — senhas fracas caem.

### Hyper-V (vmconnect)

Credenciais salvas de conexão com VMs via `vmconnect.exe` vão para o **Credential Manager** como alvos `LegacyGeneric:*` (mesmo esquema do `cmdkey /generic` — [referência](https://superuser.com/questions/1568902/unable-to-pass-stored-credentials-to-vmconnect)):

| Item | Caminho | Conteúdo |
|---|---|---|
| Credenciais salvas | Credential Manager, alvos `LegacyGeneric:target=<usuário/servidor>` | user:pass de conexão às VMs, recuperável via `CredReadW` no contexto do usuário |
| Gestão | Hyper-V Settings → User Credentials → Delete Saved Credentials | Interface oficial para limpar o cache |

O vault em disco fica em `%LOCALAPPDATA%\Microsoft\Vault\` e `%APPDATA%\Microsoft\Credentials\` (arquivos `.vcrd` com a chave em `Policy.vpol` no mesmo diretório — ver T1555.004 e a seção DPAPI deste doc).

## Histórico de shell

| Item | Caminho | Por que importa |
|---|---|---|
| Histórico do PowerShell | `%APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt` | Senhas digitadas em comandos ficam em texto claro. Achado clássico em DFIR e em red team |
| Histórico do cmd | `doskey /history` | Só a sessão atual |
| `cmdkey /list` | Credential Manager | Enumera credenciais salvas sem elevação |
| Transcrições | `Start-Transcript` | Log completo da sessão |

## Clientes de banco de dados

| Aplicação | Caminho | Proteção |
|---|---|---|
| **DBeaver** | `%APPDATA%\DBeaverData\workspace6\General\.dbeaver\credentials-config.json` | AES com chave padrão conhecida quando não há senha mestre |
| **SSMS** | `%APPDATA%\Microsoft\SQL Server Management Studio\<versão>\SqlStudio.bin` | DPAPI do usuário |
| **HeidiSQL** | `HKCU\Software\HeidiSQL\Servers\<nome>` | Ofuscação reversível |
| **pgAdmin** | `%APPDATA%\pgAdmin\pgadmin4.db` + `%USERPROFILE%\.pgpass` | `.pgpass` em texto claro |
| **MySQL Workbench** | `%APPDATA%\MySQL\Workbench\workbench_user_data.dat` | Vault próprio (DPAPI) |
| **Azure Data Studio** | `%APPDATA%\azuredatastudio\` + Credential Manager | safeStorage/DPAPI |

## Arquivos de configuração esquecidos

| Arquivo | Local típico | Conteúdo |
|---|---|---|
| `unattend.xml`, `autounattend.xml` | `C:\Windows\Panther\`, `C:\Windows\System32\sysprep\` | Senha de admin local em texto claro, deixada pela imagem de deploy |
| `web.config` | `C:\inetpub\wwwroot\` | Connection strings com usuário e senha |
| `.env` | Pastas de projeto | Secrets de API, banco e cloud em texto claro |
| `appsettings.json` | Projetos .NET | Connection strings e chaves de API |
| `wp-config.php` | Servidores web | Credenciais de banco |

## Navegadores

### Família Chromium

Todos seguem o mesmo modelo: logins em `<User Data>\Default\Login Data` (SQLite, AES-GCM), cookies em `Network\Cookies`, e a chave em `<User Data>\Local State`, campo `os_crypt.encrypted_key`, protegida por DPAPI. Chrome, Edge e Brave a partir da versão 127 adicionam App-Bound Encryption, que vincula a chave ao binário legítimo.

| # | Navegador | Caminho do perfil |
|---|---|---|
| 1 | Google Chrome | `%LOCALAPPDATA%\Google\Chrome\User Data\` |
| 2 | Microsoft Edge | `%LOCALAPPDATA%\Microsoft\Edge\User Data\` |
| 3 | Brave | `%LOCALAPPDATA%\BraveSoftware\Brave-Browser\User Data\` |
| 4 | Opera | `%APPDATA%\Opera Software\Opera Stable\` (Roaming, sem `User Data`) |
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
| 16 | 360 Secure Browser | `%LOCALAPPDATA%\360Chrome\Chrome\User Data\` |
| 17 | Cent Browser | `%LOCALAPPDATA%\CentBrowser\User Data\` |
| 18 | UC Browser | `%LOCALAPPDATA%\UCBrowser\` |
| 19 | Maxthon (v6+) | `%APPDATA%\Maxthon6\` |
| 20 | Chromium (open-source) | `%LOCALAPPDATA%\Chromium\User Data\` |

Perfis adicionais ficam em `Profile 1`, `Profile 2` e assim por diante; ferramentas de extração iteram todos. Opera e Opera GX são exceção e usam `%APPDATA%` direto.

### Família Gecko (Firefox e forks)

Modelo comum: `logins.json` com as credenciais, `key4.db` com a chave NSS, `cookies.sqlite`. Sem senha mestre, tudo é decriptável offline.

| # | Navegador | Caminho do perfil |
|---|---|---|
| 1 | Mozilla Firefox | `%APPDATA%\Mozilla\Firefox\Profiles\<perfil>\` |
| 2 | Tor Browser | `<pasta de instalação>\Browser\TorBrowser\Data\Browser\profile.default\` (portátil, geralmente no Desktop) |
| 3 | Waterfox | `%APPDATA%\Waterfox\Profiles\<perfil>\` |
| 4 | LibreWolf | `%APPDATA%\LibreWolf\Profiles\<perfil>\` |
| 5 | Pale Moon | `%APPDATA%\Moonchild Productions\Pale Moon\Profiles\<perfil>\` |
| 6 | SeaMonkey | `%APPDATA%\Mozilla\SeaMonkey\Profiles\<perfil>\` |
| 7 | Thunderbird | `%APPDATA%\Thunderbird\Profiles\<perfil>\`, senhas de e-mail no mesmo formato |

### Legado

| Navegador | Local | Proteção |
|---|---|---|
| Internet Explorer / Edge Legacy | Windows Vault (`%APPDATA%\Microsoft\Vault\`) | DPAPI com esquema do Vault (doc 01) |

## Git e ferramentas de dev

| Item | Caminho |
|---|---|
| Git Credential Manager | Credential Manager (DPAPI) |
| GitHub CLI (`gh`) | Credential Manager + `%APPDATA%\GitHub CLI\hosts.yml` |
| Credenciais em claro | `%USERPROFILE%\.git-credentials` (quando `credential.helper store`) |
| Config global | `%USERPROFILE%\.gitconfig` |
| `.netrc` | `%USERPROFILE%\_netrc` |
| NuGet | `%APPDATA%\NuGet\NuGet.Config` |
| npm | `%USERPROFILE%\.npmrc` |
| pip | `%APPDATA%\pip\pip.ini` |
| AWS CLI | `%USERPROFILE%\.aws\credentials` |
| Azure CLI | `%USERPROFILE%\.azure\` |
| Docker | `%USERPROFILE%\.docker\config.json` (auth em Base64 sem cred helper) |
| kubeconfig | `%USERPROFILE%\.kube\config` |
| Terraform | `%USERPROFILE%\.terraform.d\credentials.tfrc.json` |
| Helm | `%APPDATA%\helm\registry\config.json` |

## Gerenciadores de senha

| App | Local típico do vault |
|---|---|
| KeePass | Arquivo `.kdbx` escolhido pelo usuário; Documents e pastas sincronizadas são comuns |
| Bitwarden (desktop) | `%APPDATA%\Bitwarden\data.json` (criptografado, chave derivada do login) |
| 1Password | `%LOCALAPPDATA%\1Password\` |

## Implicações de segurança

`sitemanager.xml`, `_netrc`, `.git-credentials`, `.pgpass`, `.env` e `unattend.xml` são os achados mais fáceis. Base64 não é criptografia, e texto claro não perdoa.

O `ConsoleHost_history.txt` é uma mina de senhas digitadas. Vale auditar em IR e ensinar dev a nunca passar senha como argumento de comando.

IDEs concentram tokens de GitHub, Azure e bancos de produção. Uma estação de dev comprometida vale por dez.

Infostealers como RedLine, Vidar, Lumma e Raccoon carregam listas hardcoded com praticamente todos os caminhos deste documento, inclusive os de forks obscuros que o usuário instalou achando que eram mais seguros. A mesma lógica vale para a defesa: essas pastas são o melhor lugar do mundo para canary files.
