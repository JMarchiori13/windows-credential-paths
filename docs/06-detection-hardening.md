# 06 — Detecção e Hardening

O outro lado do repositório: como proteger e monitorar cada local documentado.

## Hardening por camada

### Sistema operacional

1. Credential Guard, que isola os segredos do LSASS em virtualização. Exige UEFI e Secure Boot.
2. LSA Protection (`RunAsPPL = 1` em `HKLM\SYSTEM\CurrentControlSet\Control\Lsa`), impedindo leitura de memória do lsass por processo não protegido.
3. WDigest desligado (`UseLogonCredential = 0`, padrão desde o Windows 8.1).
4. `CachedLogonsCount` em 1 ou 2 nas máquinas de administradores.
5. gMSA para serviços, eliminando senha de serviço dos LSA Secrets.
6. Windows LAPS, para tirar a senha de admin local do SAM estático e rotacioná-la.

### Chaves SSH e arquivos de usuário

- Passphrase em toda chave, com agente ou hardware (TPM, FIDO2) centralizando.
- ACL restritiva no `.ssh` (doc 03).
- Proibir `credential.helper store`; usar o Git Credential Manager.
- Proibir senha em `sitemanager.xml` e `.netrc`; trocar por SFTP com chave ou senha mestre.

## Auditoria e detecção

| Evento | Fonte | O que observar |
|---|---|---|
| Leitura de `SAM`, `SECURITY`, `SYSTEM` | Sysmon EID 11, SACL nos hives | Qualquer processo fora de backup e EDR legítimos |
| Dump do lsass | Sysmon EID 10 | GrantedAccess 0x1010 ou 0x1FFFFF no `lsass.exe` |
| Acesso a masterkeys DPAPI | SACL em `%APPDATA%\Microsoft\Protect\` | Processos fora do sistema |
| Leitura de `.ssh`, `.aws`, `.kube` | Sysmon EID 11 | Processos inesperados |
| `netsh wlan show profile ... key=clear` | Sysmon EID 1 | Enumeração de senha Wi-Fi |
| Shadow copy em `config\` ou NTDS | Sysmon EID 1 | vssadmin, wmic shadowcopy |
| `reg save` em hive sensível | Sysmon EID 13, ETW | Exportação de SAM, SECURITY, SYSTEM |

No catálogo do SigmaHQ, as regras de referência incluem `proc_creation_win_reg_save_security_hive`, `proc_creation_win_susp_ntds` e `win_access_susp_sensitive_registry_hive`.

## Canary files

Crie um `.ssh\id_rsa` falso, um `.aws\credentials` falso e um `sitemanager.xml` falso, todos com conteúdo-chamariz e SACL de auditoria de leitura. Qualquer leitura vira alerta de altíssima fidelidade, porque nenhum processo legítimo tem motivo para tocar neles.

## Checklist de auditoria

- [ ] Credential Guard e RunAsPPL ligados?
- [ ] WDigest desabilitado?
- [ ] Alguma chave SSH sem passphrase nos perfis?
- [ ] `.git-credentials`, `_netrc` ou `config.json` do Docker com auth em Base64?
- [ ] Credenciais RDP salvas (`vaultcmd /listcreds "Windows Credentials"`)?
- [ ] gMSA em uso nos serviços?
- [ ] Sysmon e SACLs cobrindo os caminhos deste repositório?

## Referências

- Microsoft Docs: DPAPI, Credential Manager, Credential Guard, Windows LAPS
- MITRE ATT&CK: T1003, T1552, T1555, T1552.004
- SigmaHQ: regras de credential access para Windows
