# 04 — Credenciais em Aplicações de Terceiros

## Clientes SSH/FTP

| Aplicação | Local das credenciais | Proteção |
|---|---|---|
| **WinSCP** | `HKCU\Software\Martin Prikryl\WinSCP 2\Sessions` | Ofuscação reversível (sem chave secreta real) |
| **FileZilla** | `%APPDATA%\FileZilla\sitemanager.xml`<br>`%APPDATA%\FileZilla\recentservers.xml` | **Texto claro ou Base64** |
| **mRemoteNG** | `%APPDATA%\mRemoteNG\confCons.xml` | AES com senha mestre (padrão fraco/conhecido em versões antigas) |
| **Xshell/Xftp** | `%USERPROFILE%\Documents\NetSarang Computer\` | Ofuscação própria |
| **MobaXterm** | Registro + `MobaXterm.ini` | AES com chave derivada de hostname+usuário |

## Navegadores (Chromium e Firefox)

| Navegador | Caminho dos logins | Proteção |
|---|---|---|
| Chrome/Edge | `%LOCALAPPDATA%\Google\Chrome\User Data\Default\Login Data`<br>`%LOCALAPPDATA%\Microsoft\Edge\User Data\Default\Login Data` | DPAPI (chave em `Local State`, AES-GCM); Chrome 127+ usa App-Bound Encryption |
| Firefox | `%APPDATA%\Mozilla\Firefox\Profiles\<perfil>\logins.json` + `key4.db` | NSS; sem senha mestre = decriptável offline |
| Cookies (session tokens) | `Network\Cookies` (SQLite) nos mesmos perfis | DPAPI/App-Bound |

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
- Reconhecimento de atacantes (T1552/T1555) enumera exatamente essas pastas — ótimo material para **canary files** e regras de detecção.
- Política: proibir `credential.helper store`, exigir senha mestre no Firefox, preferir gerenciadores de senha corporativos.
