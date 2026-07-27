# 04 — Credenciais em Aplicações de Terceiros

## Clientes SSH/FTP

| Aplicação | Local das credenciais | Proteção |
|---|---|---|
| **WinSCP** | `HKCU\Software\Martin Prikryl\WinSCP 2\Sessions` | Ofuscação reversível (sem chave secreta real) |
| **FileZilla** | `%APPDATA%\FileZilla\sitemanager.xml`<br>`%APPDATA%\FileZilla\recentservers.xml` | **Texto claro ou Base64** |
| **mRemoteNG** | `%APPDATA%\mRemoteNG\confCons.xml` | AES com senha mestre (padrão fraco/conhecido em versões antigas) |
| **Xshell/Xftp** | `%USERPROFILE%\Documents\NetSarang Computer\` | Ofuscação própria |
| **MobaXterm** | Registro + `MobaXterm.ini` | AES com chave derivada de hostname+usuário |

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

## Gerenciadores de senha

| App | Local típico do vault |
|---|---|
| KeePass | arquivo `.kdbx` escolhido pelo usuário — comuns: `Documents`, OneDrive/Dropbox sync |
| Bitwarden (desktop) | `%APPDATA%\Bitwarden\data.json` (criptografado, chave derivada do login) |
| 1Password | `%LOCALAPPDATA%\1Password\` |

## Implicações de segurança

- `sitemanager.xml`, `_netrc`, `.git-credentials` e `config.json` do Docker são os achados mais fáceis: **Base64 não é criptografia**.
- Infostealers (RedLine, Vidar, Lumma, Raccoon) têm listas hardcoded cobrindo praticamente **todos** os caminhos de navegadores da tabela acima — inclusive forks obscuros, que usuários instalam achando ser "mais seguros".
- Reconhecimento de atacantes (T1552/T1555/T1539) enumera exatamente essas pastas — ótimo material para **canary files** e regras de detecção.
- Política: proibir `credential.helper store`, exigir senha mestre nos Gecko-based, preferir gerenciadores de senha corporativos a cofres de navegador.
