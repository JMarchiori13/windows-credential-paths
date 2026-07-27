# 07 — Priorização Ofensiva: Ordem de Coleta de Credenciais

> **Perspectiva 🔴 Red Team.** Este documento inverte o ângulo dos demais: em vez de "onde cada credencial está", responde **"o que um operador coleta primeiro, com qual privilégio, e por quê"**. Para a defesa, essa ordem define a prioridade de SACLs, canary files e regras de detecção (ver doc 06).

## Fase 1 — Primeira coleta (usuário comum, sem elevação)

Tudo que pode ser lido **pelo próprio usuário, sem admin**. É o alvo típico de infostealers e do primeiro estágio de um engajamento.

| Caminho | Técnica ATT&CK | Por que é priorizado |
|---|---|---|
| `%USERPROFILE%\.ssh\` | T1552.004 | Chaves privadas em texto claro = acesso direto a servidores Linux, pivoting imediato |
| `%USERPROFILE%\.aws\credentials`<br>`%USERPROFILE%\.azure\`<br>`%USERPROFILE%\.kube\config` | T1552.001 | Acesso a cloud inteira; frequentemente com privilégios amplos |
| `%USERPROFILE%\.git-credentials`<br>`%USERPROFILE%\_netrc` | T1552.001 | Texto claro; abre repositórios e código-fonte |
| `%USERPROFILE%\.docker\config.json` | T1552.001 | Auth Base64 de registries de container |
| `Login Data` + `Local State` dos 20 navegadores Chromium (doc 04) | T1555.003 | Senhas + cookies de tudo que o usuário acessa; DPAPI do próprio usuário não exige admin |
| `logins.json` + `key4.db` (Firefox/forks) | T1555.003 | Sem senha mestre = decriptável offline |
| `%APPDATA%\discord\Local Storage\leveldb\` | T1528 | Token de sessão: conta completa sem senha/2FA; revogação depende do usuário |
| `%APPDATA%\FileZilla\sitemanager.xml`<br>WinSCP/mRemoteNG/MobaXterm | T1552 | Credenciais de servidores — mapa pronto para movimento lateral |
| `*.rdp` em Desktop/Documents | T1552 | Podem conter blob DPAPI de senha RDP |
| `%APPDATA%\Microsoft\Credentials\` + Vault | T1555.004 | Credenciais RDP/web salvas (`TERMSRV/*`) — decriptáveis com a masterkey do próprio usuário |

**Custo para o atacante**: baixo. **Custo para a defesa não notar**: alto — leituras de arquivos pequenos raramente disparam alerta sem SACL/Sysmon dedicado.

## Fase 2 — Pós-elevação (admin local / SYSTEM)

Após escalar privilégio, o alvo passa a ser **outros usuários e a própria máquina como trampolim de domínio**.

| Caminho | Técnica ATT&CK | Por que é priorizado |
|---|---|---|
| Memória do `lsass.exe` | T1003.001 | Hashes NTLM, tickets Kerberos e (com WDigest) senhas em claro de **todas as sessões ativas** — incluindo admins de domínio que fizeram logon na máquina |
| `C:\Windows\System32\config\SAM` + `SYSTEM` (via VSS) | T1003.002 | Hashes de contas locais — reutilização de senha de admin local entre máquinas é comum |
| `HKLM\SECURITY\Policy\Secrets` (hive SECURITY) | T1003.004 | `DPAPI_SYSTEM` (masterkeys de máquina), senhas de serviços, autologon |
| `%APPDATA%\Microsoft\Protect\{SID}\` de **outros usuários** | T1555.004 | Com a senha/hash de outro usuário + masterkey, todos os blobs DPAPI dele abrem offline |
| `HKLM\SECURITY\Cache` (`NL$1`–`NL$10`) | T1003.005 | Hashes MSCash v2 dos últimos logons de domínio — quebra offline |
| `C:\ProgramData\Microsoft\Crypto\RSA\MachineKeys\` | T1552.004 | Chaves privadas de certificados da máquina (auth de cliente, EFS) |
| `C:\ProgramData\Microsoft\Wlansvc\Profiles\` | T1552 | Senhas Wi-Fi (DPAPI de máquina) — acesso físico/persistência em redes corporativas |
| Em DCs: `C:\Windows\NTDS\NTDS.dit` + SYSTEM | T1003.003 | **Game over de domínio** — todos os hashes do AD |

## Fase 3 — Movimento lateral

Credenciais coletadas viram acesso a novos hosts; o foco muda para **alvos e alcance**.

| Fonte | Técnica ATT&CK | Uso |
|---|---|---|
| Credenciais RDP salvas (`TERMSRV/*`) | T1021.001 | Logon interativo em servidores sem malware |
| Sessões WinSCP/mRemoteNG/MobaXterm | T1021.004 (SSH) | Inventário de hosts + credenciais no mesmo arquivo |
| Tokens AWS/Azure/kubectl | T1528 | Movimento lateral **para a cloud** — fora do alcance de EDRs de endpoint |
| Tickets Kerberos (do lsass) | T1550.003 | Pass-the-Ticket para serviços de domínio |
| Hashes NTLM (SAM/lsass/NTDS) | T1550.002 | Pass-the-Hash em SMB/WMI/WinRM |
| Cache de domínio (`NL$*`) de máquinas de admins | T1003.005 | Crack offline → credencial de domínio válida |

## O que isso muda na defesa

1. **Ordene SACLs e canary files pela Fase 1** — é onde o custo/benefício do atacante é melhor e onde a detecção precoce corta a cadeia.
2. **Canary files de alta fidelidade**: `.ssh\id_rsa`, `.aws\credentials` e `sitemanager.xml` falsos com auditoria de leitura (ver doc 06).
3. **Quebre a Fase 2**: Credential Guard + RunAsPPL + LAPS eliminam os três primeiros itens da tabela.
4. **Monitore a Fase 3**: logons RDP/WinRM com contas locais, uso de tickets fora do horário, autenticações cloud de IPs novos.

## Referências

- MITRE ATT&CK: T1003, T1552, T1555, T1528, T1550, T1021
- Relatórios públicos de infostealers (RedLine, Vidar, Lumma) — listas de alvos hardcoded
- Docs 01–06 deste repositório (caminhos e proteções detalhados)
