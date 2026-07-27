# Windows Credential & SSH Key Paths

Documentação técnica dos **caminhos do sistema Windows onde senhas, credenciais e chaves SSH são armazenadas**, com detalhes sobre formato, mecanismo de proteção (DPAPI, LSA Secrets, SAM) e implicações de segurança ofensiva e defensiva.

> **Propósito**: referência para equipes de **Red Team (pentest autorizado), Blue Team, DFIR e hardening**. Conhecer onde credenciais residem em disco é essencial tanto para **simular o adversário** (credential access em engajamentos autorizados) quanto para **auditar exposição, detectar credential dumping e proteger endpoints**. Uso fora de ambientes autorizados é ilegal.

## Perspectiva dupla

| 🔴 Red Team | 🔵 Blue Team |
|---|---|
| Onde coletar credenciais em cada fase do engajamento (acesso inicial → pós-exploração → movimento lateral) | Onde colocar SACLs, canary files e regras de detecção |
| Priorização tática dos caminhos — ver **[doc 07](docs/07-priorizacao-ofensiva.md)** | Prioridade de hardening baseada na ordem de coleta do atacante |
| Técnicas MITRE ATT&CK T1003, T1552, T1555, T1528 | Regras Sigma/Sysmon, Credential Guard, LAPS, gMSA |

## Cobertura

| Escopo | Quantidade |
|---|---|
| Navegadores (Chromium, Gecko e legado) | **27** — Chrome, Edge, Brave, Opera, Vivaldi, Arc, Yandex, Firefox, Tor, LibreWolf e mais |
| Plataformas de token de sessão | **9** — Discord, Slack, Teams, Telegram, WhatsApp, Steam, Epic, Spotify, Riot |
| Clientes de e-mail | **5** — Outlook, New Outlook, Thunderbird, eM Client, Mailbird |
| IDEs e editores | **3** — VS Code, JetBrains, Visual Studio |
| Clientes de banco de dados | **6** — DBeaver, SSMS, HeidiSQL, pgAdmin, MySQL Workbench, Azure Data Studio |
| Clientes SSH/FTP | 5 — PuTTY, WinSCP, FileZilla, mRemoteNG, MobaXterm |
| Ferramentas de dev (Git, AWS, Azure, Docker, kubeconfig, Terraform...) | 14 |
| Locais nativos do Windows (DPAPI, SAM, LSA, Vault, Wi-Fi, RDP, VPN, Hello) | 25+ |

---

## Índice rápido (TL;DR)

| Categoria | Caminho / Local | Proteção |
|---|---|---|
| Chaves SSH do usuário | `%USERPROFILE%\.ssh\` | Nenhuma por padrão (passphrase opcional) |
| OpenSSH Agent / Service | `HKLM\SYSTEM\CurrentControlSet\Services\ssh-agent` | DPAPI + ACL do serviço |
| Credential Manager (arquivos) | `%APPDATA%\Microsoft\Credentials\`<br>`%LOCALAPPDATA%\Microsoft\Credentials\` | DPAPI |
| Vault do Windows | `%LOCALAPPDATA%\Microsoft\Vault\`<br>`%APPDATA%\Microsoft\Vault\` | DPAPI + Vault schema |
| Masterkeys DPAPI (usuário) | `%APPDATA%\Microsoft\Protect\{SID}\` | Derivada da senha do usuário |
| Masterkeys DPAPI (máquina) | `C:\Windows\System32\Microsoft\Protect\` | Derivada da senha da máquina (LSA) |
| SAM (senhas locais, hash) | `C:\Windows\System32\config\SAM` | SYSKEY + ACL (SYSTEM only) |
| LSA Secrets | `HKLM\SECURITY\Policy\Secrets` | SYSKEY/LSA encryption |
| Credenciais em cache de domínio | `HKLM\SECURITY\Cache` | MSCash v2 |
| Windows Hello (chaves NGC) | `C:\Windows\ServiceProfiles\LocalService\AppData\Local\Microsoft\Ngc\` | TPM 2.0 (não exportável) / DPAPI |
| Credential Guard / LSASS (memória) | `lsass.exe` (processo) | NTLM/Kerberos em memória |
| Perfis de rede Wi-Fi | `C:\ProgramData\Microsoft\Wlansvc\Profiles\Interfaces\{GUID}\*.xml` | DPAPI |
| Credenciais RDP salvas | Credential Manager (`TERMSRV/host`) | DPAPI |
| Token do Discord | `%APPDATA%\discord\Local Storage\leveldb\` | Electron safeStorage (DPAPI) |
| Histórico do PowerShell | `%APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt` | **Texto claro** |
| Logins de navegadores (Chromium) | `%LOCALAPPDATA%\<Vendor>\<Browser>\User Data\Default\Login Data` | DPAPI + AES-GCM / App-Bound |
| Logins de navegadores (Gecko) | `%APPDATA%\<Vendor>\<Browser>\Profiles\<perfil>\logins.json` + `key4.db` | NSS (senha mestre opcional) |

---

## Documentos

1. **[DPAPI e Credential Manager](docs/01-dpapi-credential-manager.md)** — Masterkeys, blobs de credencial, Vault, escopo usuário vs. máquina.
2. **[SAM, LSA Secrets e cache de domínio](docs/02-sam-lsa-secrets.md)** — hives de registro, SYSKEY, MSCash, Windows Hello/PIN/biometria, segredos de serviço.
3. **[Chaves SSH no Windows](docs/03-ssh-keys.md)** — OpenSSH client/server, agente, PuTTY/Pageant, permissões NTFS.
4. **[Aplicações de terceiros](docs/04-app-specific-paths.md)** — **27 navegadores**, tokens de sessão, e-mail, IDEs, bancos de dados, SSH/FTP, dev tools, shell history, configs esquecidos.
5. **[Credenciais de rede](docs/05-network-credentials.md)** — Wi-Fi, RDP, mapeamentos de drive, VPN.
6. **[Detecção e hardening](docs/06-detection-hardening.md)** 🔵 — como defender cada local, eventos de auditoria e regras de detecção.
7. **[Priorização ofensiva](docs/07-priorizacao-ofensiva.md)** 🔴 — ordem tática de coleta: fase, privilégio necessário, técnica ATT&CK e o que muda na defesa.

---

## Convenções

- `%USERPROFILE%` → `C:\Users\<usuario>`
- `%APPDATA%` → `C:\Users\<usuario>\AppData\Roaming`
- `%LOCALAPPDATA%` → `C:\Users\<usuario>\AppData\Local`
- `{SID}` → Security Identifier do usuário (ex.: `S-1-5-21-...`)
- `{GUID}` → identificador único de interface/objeto

## Aviso legal

Todo o conteúdo descreve **localizações públicas e documentadas** do sistema operacional, com finalidade educacional, de segurança ofensiva **autorizada** e defensiva. Acesso a credenciais de terceiros sem autorização viola legislações como a Lei Carolina Dieckmann (BR), a CFAA (US) e o GDPR.
