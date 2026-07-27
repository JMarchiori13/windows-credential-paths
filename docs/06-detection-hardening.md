# 06 — Detecção e Hardening

Resumo defensivo: como proteger e monitorar cada local documentado neste repositório.

## Hardening por camada

### Sistema operacional

1. **Credential Guard** — isola segredos do LSASS em virtualização (VSM/LSAiso). Requer UEFI + Secure Boot.
2. **LSA Protection (RunAsPPL)** — `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\RunAsPPL = 1`: impede leitura de memória do lsass por processos não protegidos.
3. **WDigest off** — garantir `UseLogonCredential = 0` (padrão desde Win 8.1/2012R2).
4. **CachedLogonsCount** — reduzir para 1–2 em máquinas de administradores.
5. **gMSA / contas de serviço gerenciadas** — elimina senhas de serviço em LSA Secrets.
6. **LAPS / Windows LAPS** — senhas locais de admin rotacionadas e fora do SAM estático.

### Chaves SSH e arquivos de usuário

- Exigir passphrase em todas as chaves; centralizar em agente ou hardware (TPM/FIDO2).
- ACLs restritivas em `.ssh` (ver doc 03).
- Proibir `git config credential.helper store`; usar Git Credential Manager (DPAPI).
- Proibir senhas em `sitemanager.xml`/`.netrc`; usar FileZilla com senha mestre ou SFTP com chave.

### Auditoria e detecção

| Evento | Fonte | O que observar |
|---|---|---|
| Leitura de `SAM`/`SECURITY`/`SYSTEM` | Sysmon EID 11 / SACL nos hives | Qualquer processo fora de backup/EDR legítimo |
| Dump do lsass | Sysmon EID 10 (GrantedAccess 0x1010/0x1FFFFF em lsass.exe) | procdump, rundll32 comsvcs, mimikatz |
| Acesso a masterkeys DPAPI | SACL em `%APPDATA%\Microsoft\Protect\` | Processos não-sistema |
| Leitura de `.ssh`, `.aws`, `.kube` | Sysmon EID 11 | Processos inesperados (scripts, LOLBins) |
| `netsh wlan show profile ... key=clear` | Sysmon EID 1 / linha de comando | Enumeracao de senhas Wi-Fi |
| VSS criado em `config\` ou `NTDS` | EID 1 (vssadmin, wmic shadowcopy) | Extração offline de credenciais |
| Acesso ao registro `HKLM\SECURITY` | EID 13 / ETW | reg save, reg export de hives sensíveis |

Regra Sigma de referência: buscar por `proc_creation_win_reg_save_security_hive`, `proc_creation_win_susp_ntds`, `win_access_susp_sensitive_registry_hive` no catálogo SigmaHQ.

### Canary files (armadilhas)

- Criar `.ssh\id_rsa` falso, `.aws\credentials` falso e `sitemanager.xml` falso com conteúdo-chamariz + SACL de auditoria de leitura. Qualquer leitura = alerta de alta fidelidade.

## Checklist rápido de auditoria

- [ ] Credential Guard e RunAsPPL habilitados?
- [ ] WDigest desabilitado?
- [ ] Alguma chave SSH sem passphrase nos perfis?
- [ ] `.git-credentials`, `_netrc`, `docker config.json` com auth Base64?
- [ ] Credenciais RDP salvas (`vaultcmd /listcreds "Windows Credentials"`)?
- [ ] gMSA em uso para serviços?
- [ ] Sysmon + SACLs cobrindo os caminhos deste repo?

## Referências

- Microsoft Docs: DPAPI, Credential Manager, Credential Guard, Windows LAPS
- MITRE ATT&CK: T1003 (OS Credential Dumping), T1552 (Unsecured Credentials), T1555 (Credentials from Password Stores), T1552.004 (Private Keys)
- SigmaHQ — regras de credential access para Windows
