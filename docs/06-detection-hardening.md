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

### Virtualização e ferramentas de reparo

1. Proibir "remember me" nas conexões vSphere/ESXi da VMware Workstation — a cadeia de decriptação é inteiramente reversível no contexto do usuário (doc 04). Preferir vCenter com SSO e sessão curta.
2. Não usar "remember password" em VMs criptografadas da Workstation: a senha fica no Credential Manager legível via `CredReadW` sem elevação.
3. Limpar credenciais salvas do vmconnect (Hyper-V Settings → User Credentials → Delete Saved Credentials) em estações de administração.
4. Senhas de disco VirtualBox fortes e únicas: o KeyStore é inteiramente atacável offline (PBKDF2), então a senha é a única barreira.
5. Inventariar ferramentas de schematics/reparo (Borneo, ZXW, Refox): tokens de sessão em cache local são conta ativa pronta. Fazer logout ao fim do uso e tratar esses diretórios como sensíveis.

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
| Leitura de `preferences-private.ini` e `ace.dat` | Sysmon EID 11, SACL em `%APPDATA%\VMware\` | Qualquer processo que não seja `vmware.exe`/`vmware-vmx.exe` |
| `CredReadW` em alvos `encryptedVM.guid` / `LegacyGeneric:*` | ETW/telemetria de API, SACL no Vault | Enumeração de credenciais fora de processos legítimos |
| Leitura de KeyStore em `*.vbox` | Sysmon EID 11 | Processos fora do VirtualBox lendo configuração de VMs alheias |
| Acesso a caches de apps de reparo | Sysmon EID 11, SACL em `%LOCALAPPDATA%\Borneo-App-Cache` e `%APPDATA%\<app>` | Processos estranhos lendo `Cookies`, `leveldb`, `Local Storage` |

No catálogo do SigmaHQ, as regras de referência incluem `proc_creation_win_reg_save_security_hive`, `proc_creation_win_susp_ntds` e `win_access_susp_sensitive_registry_hive`.

## Canary files

Crie um `.ssh\id_rsa` falso, um `.aws\credentials` falso e um `sitemanager.xml` falso, todos com conteúdo-chamariz e SACL de auditoria de leitura. Qualquer leitura vira alerta de altíssima fidelidade, porque nenhum processo legítimo tem motivo para tocar neles. A mesma lógica vale para um `ace.dat` falso em `%APPDATA%\VMware\` de uma estação sem Workstation instalada e para um `Borneo-App-Cache` plantado em máquina sem a ferramenta: leitura nesses caminhos é certeza de coleta de credenciais em andamento.

## Checklist de auditoria

- [ ] Credential Guard e RunAsPPL ligados?
- [ ] WDigest desabilitado?
- [ ] Alguma chave SSH sem passphrase nos perfis?
- [ ] `.git-credentials`, `_netrc` ou `config.json` do Docker com auth em Base64?
- [ ] Credenciais RDP salvas (`vaultcmd /listcreds "Windows Credentials"`)?
- [ ] gMSA em uso nos serviços?
- [ ] Sysmon e SACLs cobrindo os caminhos deste repositório?
- [ ] "Remember me" desabilitado nas conexões vSphere/ESXi das Workstations?
- [ ] Alvos `encryptedVM.guid` e `LegacyGeneric:*` presentes no Credential Manager de estações de admin?
- [ ] Logout ativo nos caches de ferramentas de reparo (Borneo, ZXW, Refox)?

## Referências

- Microsoft Docs: DPAPI, Credential Manager, Credential Guard, Windows LAPS
- MITRE ATT&CK: T1003, T1552, T1555, T1552.004
- SigmaHQ: regras de credential access para Windows
